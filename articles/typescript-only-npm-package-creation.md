---
title: "Bun専用(?)TypeScriptそのままnpmパッケージの作成に関する覚え書き"
emoji: "🕊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bun", "typescript", "npm"]
published: false
---

Bunのドキュメントにはこのように[書いてある](https://bun.sh/docs/runtime/modules#importing-packages)。（2024Q2時点）

> **Shipping TypeScript** — Note that Bun supports the special `"bun"` export condition. If your library is written in TypeScript, you can publish your (un-transpiled!) TypeScript files to `npm` directly. If you specify your package's `*.ts` entrypoint in the `"bun"` condition, Bun will directly import and execute your TypeScript source files.

`bun init` で出力される `package.json` の `"module": "index.ts"` も `tsconfig.json` の中身もこれが前提のように見える。

というわけでTypeScriptそのままnpmパッケージを作ってみる。

## 要件

- 恐らくBun専用
    - 他にも対応するパッケージマネージャーが存在するかも
- ESM専用とする
- TypeScriptをトランスパイルせずそのままnpmパッケージに収める
    - エイリアス（ `tsconfig.json` の `paths` ）使用
    - JavaScriptは含まない
- VSCodeのIntelliSenseで型の補完が効くようにする
    - これを満たすために `index.d.ts` の生成が必要になった、詳細は後述
- npmjsレジストリに公開パッケージとしてPublishする

本稿作成時のBunのバージョンは `1.1.3` 。

## package.json

- npmにおける仕様: [package.json (Version 10.x)](https://docs.npmjs.com/cli/v10/configuring-npm/package-json)
- Node.jsにおける仕様: [Modules: Packages (Node.js 20.x)](https://nodejs.org/docs/latest-v20.x/api/packages.html)
- Bunにおける仕様: [Importing packages (Bun 1.x)](https://bun.sh/docs/runtime/modules#importing-packages)
    - import時のエントリーポイントの優先順位は `exports > module > main` となる模様

`bun init` で生成される `package.json` は以下の通り。
これを踏襲してエントリーポイントには `module` を使用することとする。

```json5:package.json
{
  "name": "package-name",
  "module": "index.ts",
  "type": "module",
  "devDependencies": {
    "@types/bun": "latest"
  },
  "peerDependencies": {
    "typescript": "^5.0.0"
  }
}
```

現時点では恐らく無意味だが `engines` も設定しておく。

```json5:package.json
{
  // ...
  "engines": {
    "bun": "^1.0.0"
  },
  // ...
}
```

## ファイル選別

デプロイ時にディレクトリとファイルを一切弄らないため、 `package.json` の `files` キーや `.npmignore` で必要なファイルのみに絞り込む必要がある。

```json5:package.json
{
  // ...
  "files": [
    "**/*.ts",
    "**/*.md",
    "tsconfig.json",
    "!**/*.test.ts",
    "!**/__test__"
  ],
  // ...
}
```

- [`.gitignore` と同じ書式](https://git-scm.com/docs/gitignore)なので `!` や `**` が使える
- テスト用ダミー等の不要なtsファイルが混在するケースでは識別のために二重拡張子にする等の対策が必要になりそう
- 設定ファイルやサンプルなどでtsファイルを追加する際は混入しないよう注意
    - 手間を考えるとsrcディレクトリに入れる構造はそのままでもいいかも知れない
    - もしくはワイルドカードで一括で弾けるような命名規則にするか
        - 例えば `!**/__*__` のような
- トランスパイルしないため `tsconfig.json` はパッケージに含める
    - エイリアスを使っている場合は同梱しないと壊れるはず
- ファイルの確認は `npm pack --dry-run`

## 型定義の追加

ここまでの内容を `npm pack` してローカルで確認したところ、VSCode上でIntelliSenseによる型の補完が正常に機能しなかったため、以下の対応を試した。

- [tsc](https://www.typescriptlang.org/docs/handbook/compiler-options.html)で `emitDeclarationOnly` をtrueにして各tsファイルと同じ場所に対応するd.tsを生成
    - NG
- 前項の構成に `outFile` を追加して単一の型定義ファイルを生成し、 `package.json` の `types` に指定
    - NG
- [tsup](https://tsup.egoist.dev/)で単一の型定義ファイルを生成し、 `package.json` の `types` に指定
    - OK → これを採用する

この対応による編集内容は以下の通り。型定義は `./index.d.ts` に出力される。

```json5:package.json
{
  // ...
  "scripts": {
    "dts": "tsup ./index.ts --dts-only --format esm --out-dir ." // 追加
  },
  "files": [
    "**/*.ts",
    "index.d.ts", // 追加
    "tsconfig.json",
    "!**/*.test.ts",
    "!**/__test__"
  ],
  "type": "module",
  "module": "index.ts",
  "types": "index.d.ts", // 追加
  "devDependencies": {
    "tsup": "^8.0.2", // `bun add -D tsup` で追加
  },
  // ...
}
```

```bash:.gitignore
# 追加
index.d.ts
```

現時点におけるこの方式の問題はIntelliSenseの補完を辿った時に実体ではなく `index.d.ts` に飛んでしまうこと。せっかくソースの現物がそこにあるのに候補にすら出ない。

## Publish

- [Scopes and registries](https://bun.sh/docs/install/registries) によれば標準のレジストリは `registry.npmjs.org`
- [Bunは未対応](https://github.com/oven-sh/bun/issues/1976)のため `npm publish` で実行する
- なお、現時点でGitHub Actionsから `npm publish` したい場合は `setup-bun` とは別に `setup-node` してから実行する必要がある
    - `--dry-run` での確認だけなら `setup-bun` だけでも問題ない（エラーは出るが）

```yaml:.github/workflows/publish.yml
# GitHub Actions設定例

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      # ...
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run check
      - run: bun run test
      - run: bun run build
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: "https://registry.npmjs.org"
      - run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## サンプル

実際にPublishしたTypeScriptオンリーのパッケージがこちら。

https://www.npmjs.com/package/elysia-openid-client

- Codeタブを見ると様子がおかしい
- 実際動く
- importして使用中に不具合を見つけたらその場で修正でき、修正したソースはそのままPRに回せる
    - 軽微なものなら使用しているプロジェクトのエディタから辿るだけでいい
    - ライブラリのルートを開けばテスト関連のファイルやLinter/Formatterの設定がないだけでほぼ元のプロジェクトそのものである

## 【付録】

### 依存関係自動更新の対応状況

- Dependabot
    - [現時点では未対応](https://github.com/dependabot/dependabot-core/issues/6528)
- Renovate
    - [初期対応は完了しており拡張中](https://github.com/renovatebot/renovate/issues/20065)

### Bunによるトランスパイル

`./index.ts` を `./index.js` に出力する場合。

```typescript:build.ts
import fs from "node:fs";
import packageJson from "./package.json";

const { dependencies, peerDependencies } = packageJson;
const external = [
  ...new Set([
    ...Object.keys(dependencies || {}),
    ...Object.keys(peerDependencies || {}),
  ]),
];

Bun.build({
  entrypoints: ["./index.ts"],
  target: "bun",
  format: "esm",
  outdir: ".",
  external,
  minify: true,
});
```

```bash:shell
bun ./build.ts
```

- CLIでも同様の設定で実行可能
    - `--external` を対象の数だけ記述するのが面倒
- ESBuildより若干高速
