---
title: "Release Drafterが作成予定のtagを事前に自動でpackage.jsonのversionに同期する"
emoji: "🕊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "github", "githubactions"]
published: false
---

## 概要

npmパッケージでは `package.json` にバージョンがハードコーディングされ、レジストリへのPublish時に[その値がバージョン指定として解釈される](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#version)。

他方、[Release Drafter](https://github.com/release-drafter/release-drafter)は汎用的な仕組みのため、npmパッケージ用に設計されている[semantic-release](https://github.com/semantic-release/semantic-release)や[Changesets](https://github.com/changesets/changesets)のような `package.json` の自動更新機能はない。

そのため、npmパッケージのリリースをRelease Drafterで管理する場合には、Release Drafterが発行するtag/releaseの内容と `package.json` の `version` の値をなんらかの方法で同期する必要がある。

本稿ではGitHub Actionsを使用して「Release Drafterの動作時にその時点で発行予定のtagを `package.json` の `version` に同期する」方法を解説する。

- Pros:
    - 「tag発行時やrelease実行時に同期する」方法だと古いバージョンの `package.json` がtagのコミットやreleaseのアーカイブに取り込まれてしまうが、この方法ではそのような問題は起きない
    - トリガーが適切なら原理的に取りこぼしは発生しない
- Cons:
    - リモート側で発生したコミットを取り込むオーバーヘッドが増える
    - 「tag発行時やrelease実行時に同期する」方法よりワークフローの実行頻度は高くなる

## 基本

まず対象ブランチが「保護されていない」場合の手順を解説する。

ワークフローは[Release DrafterのUsage](https://github.com/release-drafter/release-drafter#usage)に載っている `release-drafter.yml` をベースとする。

既定の `release-drafter.yml` はPR操作時専用のワークフローとして `on.push.branches` トリガーを削除した上で残置し、それとは別に同期のためのステップを含んだワークフローを作成する。

- 本例ではリリース対象ブランチは `main` とする
    - 異なる場合は適宜読み替えのこと
- 本例では同期後にJSONをフォーマットする
    - リポジトリのポリシーに合わせる想定
- 本稿のワークフローについてはリポジトリ設定の `Workflow permissions` は編集不要と思われる（ `Read repository contents and packages permissions` のままでよい）

### トリガー

以下の**いずれか**を使用する。（PRへのマージはコミットでもあるため両方使用する意味はない）

#### リリース対象ブランチへのコミット

```yaml:.github/workflows/version-sync-push.yml
name: Version Sync (Push)

on:
  push:
    branches:
      - main
```

- このトリガーでは取りこぼしは発生しないと思われる
    - PRをマージするタイミングでも発火する
- Publish後の最初のコミットで自動的にpatchバージョンが+1される
    - labelでインクリメント対象を指定したPRマージの場合はそちらに従う

#### リリース対象ブランチに対するPull Requestのマージ

```yaml:.github/workflows/version-sync-pr.yml
name: Version Sync (PR closed)

on:
  pull_request:
    types:
      - closed
    branches:
      - main

jobs:
  version_sync:
    # PRがマージされた時だけ実行させる
    if: github.event.pull_request.merged == true
    # ...
```

- こちらを使用する場合はリリース対象ブランチに直接コミットしても同期されない
    - 付録の整合性チェックと併用すると事故を防げる
    - 手動で実行するワークフローを用意するのもアリ

### ジョブ

```yaml:.github/workflows/version-sync-push.yml
name: Version Sync (Push)

on:
  push:
    branches:
      - main

jobs:
  sync_version:
    permissions:
      contents: write
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      # [1] リポジトリのルートにあるファイルをチェックアウト
      # - 通常package.jsonはルートにあるはず
      # - JSONの整形にformatterの設定ファイルが必要な場合もルートにあるはず
      - name: Checkout root files
        uses: actions/checkout@v4
        with:
          sparse-checkout: .

      # [2] jqでpackage.jsonからversionの値を取り出して環境変数に格納
      - name: Get current version from package.json
        run: |
          echo "CURRENT_VERSION=$(jq --raw-output .version package.json)" >> $GITHUB_ENV

      # [3] Release Drafterを実行する
      # - 新しいバージョンは `steps.drafter.outputs.resolved_version` に
      #   SemVer形式で入る
      - name: Get next version from release-drafter
        id: drafter
        uses: release-drafter/release-drafter@v6
        with:
          commitish: main
          # PRは操作しないのでAuto Labelerは不要
          disable-autolabeler: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # [4] 受け取った新しいバージョン文字列が5文字未満なら失敗させる
      # - ついでに `env.NEXT_VERSION` に入れる
      - name: Verify next version
        run: |
          NEXT_VERSION=${{ steps.drafter.outputs.resolved_version }}
          if [ ${#NEXT_VERSION} -lt 5 ]; then
            exit 1
          fi
          echo NEXT_VERSION=$(echo $NEXT_VERSION) >> $GITHUB_ENV

      # [5] Biomeの実行環境としてBunを準備する
      # - フォーマットの有無、使用するフォーマッターによって要調整
      - name: Prepare to use formatter
        # 新旧のバージョンが一致している場合はスキップ（以下同様）
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        uses: oven-sh/setup-bun@v1

      # [6] 同期実行
      - name: Sync version
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        # jqでpackage.jsonを編集→Biomeでフォーマット→Bot名義でCommitしてPush
        run: |
          echo $(jq ".version=\"${{ env.NEXT_VERSION }}\"" package.json) > package.json
          bunx @biomejs/biome format --write package.json
          git config --local user.email "github-actions[bot]@users.noreply.github.com"
          git config --local user.name "github-actions[bot]"
          git add package.json
          git commit -m "chore(npm): bump version to ${{ steps.drafter.outputs.resolved_version }}"
          git push origin HEAD:main
```

- [2]
    - [jq](https://jqlang.github.io/jq/)の出番
- [3]
    - [Release Drafterのオプションとoutputの一覧](https://github.com/release-drafter/release-drafter/blob/master/action.yml)
- [4]
    - [Bun](https://bun.sh/)
        - すごくはやい
- [5]
    - `echo $(jq 処理 foo.json) > foo.json` はjqで同一ファイルを更新する小技
        - `jq 処理 foo.json > foo.json` では空ファイルになってしまう
    - [Formatter | Biome](https://biomejs.dev/ja/formatter/)
        - 設定地獄から解放される
        - 現時点（ `1.7.0` ）では[HTML/CSS/Markdown辺りは非対応](https://biomejs.dev/ja/internals/language-support/)だがピュアJS/TSなら実戦投入可能
    - 本稿ではBOT名義でコミットしている
        - 他の名義で行いたい場合は調整のこと
        - リリース対象ブランチは通常保護されていると思われるため権限設定に留意する
    - コミットメッセージは適宜調整のこと

## 応用: 対象ブランチを保護したい場合

:::message
ある程度知識がある人向け、**非常に面倒臭い**
この件は[GitHub Communityでも長年燻っているテーマ](https://github.com/orgs/community/discussions/25305)のようだ
:::

追加で以下の3点が必要になる。

- 自家製のGitHub Apps
- 自家製のGitHub Appsのためのトークン
- `gh api` を使った処理

個人アカウントのパブリックリポジトリでのみ動作確認しているが、恐らく組織アカウントでも同様の手順で可能と思われる。

### GitHub Appsとトークンの準備

以下の記事の通り。

https://zenn.dev/tmknom/articles/github-apps-token

- 今回の使用方法ではAppsの名前は表に出ないので適当でも構わない
    - PR作成等の名前が出る作業もさせたいなら要考慮
- 権限設定で `Repository permissions -> Contents` を `Access: Read and write` に設定する
- 最終的に以下の状態になっていればよい
    - リポジトリの `Settings -> GitHub Apps` に作成したGitHub Appsが登録されている
    - リポジトリの `Settings -> Secrets and variables -> Actions` に `APP_ID` と `PRIVATE_KEY` が登録されている
    - リポジトリのどこかに上記記事の `script.sh` が配置されている

### Rulesetの作成

- リポジトリの `Settings -> Rules -> Rulesets` からリリース対象ブランチを保護するRulesetを作成する
- 作成したRulesetの `Bypass list` に前の手順で作成・登録したGitHub Appsを追加する
    - ここで名指しでバイパスさせるためにGitHub Appsが必要
- 細かく検証していないが `Settings -> Branches` で設定できるブランチ保護でも問題ない模様
    - 少なくとも `Require a pull request before merging` がONで `Do not allow bypassing the above settings` がOFFの設定では問題なく動き、後者をONにすると失敗する

### ジョブの追加

- 本編の[5]までは同様
- ただし[1]で `sparse-checkout` を使用する場合は `script.sh` が含まれるように調整すること
    - ルートに平置きは微妙なので `.github/utils/token.sh` などに置くことが考えられる
        - この場合だと `sparse-checkout: .github/utils` になる

```yaml:version-sync-app.yml
# 略

jobs:
  push-ruleset:
    runs-on: ubuntu-latest
    steps:
      # 本編の[1]～[5]

      # [6'] package.jsonの編集だけ行う
      - name: Sync version
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        run: |
          echo $(jq ".version=\"${{ env.NEXT_VERSION }}\"" package.json) > package.json
          bunx @biomejs/biome format --write package.json

      # [7] Apps実行用トークン生成
      - name: Generate GitHub Apps token
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        id: generate
        env:
          APP_ID: ${{ secrets.APP_ID }}
          PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}
        # 置き場所に合わせて調整
        run: |
          chmod +x ./script.sh
          ./script.sh

      # [8] 更新前のpackage.jsonのSHAを取得
      # - リリース対象ブランチがデフォルトブランチではない場合これだと取れないかも？
      - name: Get package.json SHA
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        run: |
          RES=$( \
            gh api \
              -H "Accept: application/vnd.github+json" \
              -H "X-GitHub-Api-Version: 2022-11-28" \
              /repos/${GITHUB_REPOSITORY}/contents/package.json \
            )
          echo PACKAGE_SHA=$(echo $RES | jq -r ".sha") >> $GITHUB_ENV
        env:
          GITHUB_TOKEN: ${{ steps.generate.outputs.token }}

      # [9] package.jsonを更新
      - name: Update package.json
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        run: |
          gh api --method PUT \
          -H "Accept: application/vnd.github+json" \
          -H "X-GitHub-Api-Version: 2022-11-28" \
          /repos/${GITHUB_REPOSITORY}/contents/package.json \
          -f "message=chore(npm): bump version to ${{ env.NEXT_VERSION }}" \
          -f "content=$(cat package.json | base64 -w 0)" \
          -f "sha=${{ env.PACKAGE_SHA }}" \
          -f "branch=main" \
          -f "committer[name]=github-actions[bot]" \
          -f "committer[email]=41898282+github-actions[bot]@users.noreply.github.com"
        env:
          GITHUB_TOKEN: ${{ steps.generate.outputs.token }}

      # [10] Apps実行用トークンを失効
      - name: Revoke GitHub Apps token
        if: env.CURRENT_VERSION != env.NEXT_VERSION
        env:
          GITHUB_TOKEN: ${{ steps.generate.outputs.token }}
        run: |
          gh api --method DELETE \
          -H "Accept: application/vnd.github+json" \
          -H "X-GitHub-Api-Version: 2022-11-28" \
          -H "Authorization: Bearer ${GITHUB_TOKEN}" \
          /installation/token
```

- 参考: [gh apiによるコンテンツ操作のドキュメント](https://docs.github.com/en/rest/repos/contents?apiVersion=2022-11-28)

## 【付録】Release Drafterを使用しない整合性チェック

コンテキストからタグ名を取得できるトリガー（少なくともtagsとreleaseは可）であればRelease Drafterを使用せずに整合性をチェックできる。

ただしこのトリガーを使用したチェックでは矛盾の発生を未然に防ぐことはできないと思われる。用途としては `npm publish` の前に念のために確認するくらいだろうか。

```yaml:.github/workflows/release.yml
name: Verify Version

on:
  release:
    types: [released]

# `v0.0.0` 形式のtagの発行をトリガーにする場合
# on:
#   push:
#     tags:
#       - 'v*.*.*'

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: .
      - name: Verify that tag and package.json match
        run: |
          TAGGED_VERSION=$(echo "${{ github.ref_name }}" | grep -oP "[0-9]+\.[0-9]+\.[0-9]$")
          PUBLISH_VERSION=$(jq --raw-output .version package.json)
          if [ "$TAGGED_VERSION" != "$PUBLISH_VERSION" ]; then
            exit 1
          fi
      # 問題なければ `npm release` 等の処理に進む
```

本例の動作は以下の通り。

1. `package.json` を読むのでチェックアウトが必要
    - 後工程でソースを一式使用するなら `sparse-checkout` は外す
1. コンテキストからタグ名を取得する
    - トリガーが `release` なのでReleaseに紐付けられたタグが使用される
    - `v0.0.0` から正規表現で `0.0.0` 部分を取り出す
    - 一致しない場合はgrepが失敗したことになるためステップ自体が失敗した扱いになる
    - `${{ github.ref_name }}` は `$GITHUB_REF_NAME` でもよい
1. jqで `package.json` から `version` の値を取り出す
1. 両者が不一致なら `exit 1` で抜ける
    - 結果を環境変数に出力する等して処理を続けることもできる
    - 前章を流用して「不一致なら同期して続行」も可

Release Drafterが出力するバージョンはSemVerに従うが、タグは自由に命名できる点に留意する。
