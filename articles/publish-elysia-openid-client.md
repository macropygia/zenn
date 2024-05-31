---
title: "ElysiaJS用OpenID Connectクライアントプラグインを作った"
emoji: "🕊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bun", "elysiajs", "oidc", "openid", "npm"]
published: true
---

https://www.npmjs.com/package/elysia-openid-client

当初の目的は「セルフホストGitLabのアカウントを使用したSSO環境に業務用のWebアプリを組み込むこと」だったが、作っている内に認証部分が汎用OIDCクライアントとして分離してしまった。

## 概要

- [ElysiaJS](https://elysiajs.com/)プラグイン
- （恐らく）[Bun](https://bun.sh/)専用npmパッケージ
    - TypeScriptファイルのみ同梱、JavaScriptにはトランスパイルしない
    - VSCodeで補完が壊れるため仕方なく[tsup](https://tsup.egoist.dev/)で型定義を出力
    - 経緯や詳細は[関連記事](https://zenn.dev/macropygia/articles/typescript-only-npm-package-creation)参照
- ESM専用
- 大筋としては[openid-client](https://www.npmjs.com/package/openid-client)のラッパー
    - ElysiaJSの機能を利用して諸々のエンドポイントとCookieベースのセッション管理機能をパッケージングしたもの
- 複数のOP（OpenID Provider）との同時連携に対応
    - ユーザーがサービスを選んでログインできる仕組みを簡単に作れる
- Cookie上のセッションIDを介してブラウザーとサーバーサイドのデータを紐付ける
    - 全てのトークンがサーバーサイドに隠蔽されるため比較的安全
    - Cookieは既定では `httpOnly` `secure` `sameSite=lax` `path=/` となる
- ElysiaJSの[onBeforeHandleフックとresolveフック](https://elysiajs.com/life-cycle/before-handle.html)を使用してサーバー内部で認証・認可の情報やユーザー情報を受け渡す
    - センシティブ情報を含む場合でも表に出さなくて済む
- サーバーサイドにおけるセッションデータの保持方法をデータアダプターによって変更可能
    - 標準でSQLite/[LokiJS](https://www.npmjs.com/package/lokijs)/[Lowdb](https://github.com/typicode/lowdb)/Redis（[ioredis](https://github.com/redis/ioredis)）に対応
        - 既定ではSQLiteインメモリーモード
        - SQLiteは[Bunのビルトインドライバー](https://bun.sh/docs/api/sqlite)を使用
        - 一部はインメモリー動作かファイルやデータベースで永続化するか選択可能
    - カスタムデータアダプターをフルスクラッチで作ることも可能
- ロガーには[pino](https://getpino.io/)をそのまま投入できる
    - 既定では[Console](https://bun.sh/docs/api/console)を使用した簡易的なロガーを使用する
    - pino互換のメソッドを用意すれば他のロガーも使用可

その他細かいことはドキュメンテーション（日本語あり）を頑張ったのでそちらを参照のこと。

https://macropygia.github.io/elysia-openid-client/ja/

:::details 本稿作成時のランタイム/ライブラリのバージョン

|App/Package|Version|
|---|---|
|Bun|1.1.3|
|elysia|1.0.14|
|openid-client|5.6.5|
|typescript|5.4.5|
|elysia-openid-client|0.1.5|

:::

### OIDC RP（Relying Party）としての仕様・制限

- `Authorization Code Flow` （認証コードフロー）専用
- `Confidential Client` 専用
- Client metadata:
    - `client_secret` 必須
    - `response_types` は `["code"]` に固定される
- Authorization parameters:
    - `response_type` は `code`に固定される
    - `response_mode` は `query` に設定するか、既定値（設定なし）である必要がある
    - `code_challenge` , `state` , `nonce` は自動で生成される
    - `code_challenge_method` は `S256` に固定される
    - `scope` には自動で `openid` が追加される

## 動作機序

単一OPと連携する設定例を元に解説する。

```typescript:single-issuer.ts
import Elysia from "elysia";
import { OidcClient } from "elysia-openid-client";

// 初期化
const rp = await OidcClient.create({
  baseUrl: "https://app.example.com", // RP(Webサイト/Webサービス）のURL
  issuerUrl: "https://issuer.exmaple.com", // OPのURL
  clientMetadata: {
    client_id: "client-id", // OPで発行する
    client_secret: "client-secret", // OPで発行する
  },
});

// OPのメタデータ出力
console.log(rp.issuerMetadata);

// プラグイン取得（↓の二つはそれぞれ個別のプラグイン）
const endpoints = rp.getEndpoints(); // エンドポイント
const hook = rp.getAuthHook(); // フック

const app = new Elysia()
  .use(endpoints) // エンドポイント適用
  .guard((app) => // この中が認証エリア
    app
      .use(hook) // フック適用
      .onBeforeHandle(({ sessionStatus, sessionClaims }) => {
        // ログイン時に得られる情報で認可を行いたい場合は更にフックを噛ませられる
        // 更にUserInfoエンドポイントを叩きに行く処理なども可
      })
      // フックの返す `sessionStatus` がnullでなければ認証済
      .get("/", ({ sessionStatus }) =>
        sessionStatus ? "Logged in" : "Restricted",
      )
      .get("/status", ({ sessionStatus }) => sessionStatus)
      .get("/claims", ({ sessionClaims }) => sessionClaims),
  )
  .get("/free", () => "Not restricted") // ここは認証不要
  .get("/logout", () => "Logout completed")
  .listen(80);
```

- クライアントを初期化し、エンドポイントとフックのプラグインを取り出して適用する
- この設定ではOPに登録するコールバックURLは `https://app.example.com/auth/callback` になる
- 初期化時点でOPにアクセスしてメタデータを取得している
    - OPが対応するエンドポイントや機能、取得できる情報の内訳等が確認できる

### フック

`.use(hook)` で[ElysiaJSのライフサイクル](https://elysiajs.com/life-cycle/overview.html)における `onBeforeHandle` に認証系の処理を挿入している。

- このフックは[`guard`](https://elysiajs.com/essential/scope.html#guard)の内側にのみ適用される、つまりguardの内側が認証エリアになる
- 認証状態をチェックし、ログイン状態と設定によって後続の処理が分岐する
    - 既定では非ログイン状態だとLoginエンドポイントにリダイレクトされ、そこから更にOPのログイン画面にリダイレクトされる
    - ログイン状態では [`resolve`](https://elysiajs.com/life-cycle/before-handle.html#resolve) フックが `sessionStatus` と `sessionClaims` に内容を入れて後続のライフサイクルに進む
    - 非ログイン状態でもリダイレクトさせない設定も可能、その場合は `sessionStatus` と `sessionClaims` は `null` になる
- resolveの説明に書いてある通り `onBeforeHandle` と `resolve` は何度もチェーンできる
    - このプラグインの `resolve` の出力を自前の `onBeforeHandle` で受けてユーザーのグループや権限等をチェックする、といったフローが作れる

### エンドポイント

`.use(endpoints)` で「OPのエンドポイントの使用」と「セッションデータの取得」を行う以下のエンドポイントを登録している。

- エンドポイントは認証エリアの外に設置する必要がある
    - 中に設置するとそもそもログイン導線にアクセスできなくなる
- パスは既定のものであり変更可能
    - `/auth` はインスタンス共通のprefixでその後ろがエンドポイント毎のパス
        - 複数のOPを扱う時はprefixは個別のものになる
- `ALL` 表示のものは全てのメソッドで使用可能
- 機能によってはOPが対応していない場合もある、 `issuerMetadata` で確認のこと

#### Login (GET: `/auth/login` )

- `openid-client` の `client.authorizationUrl` を呼び出す
- OPの認証エンドポイントにリダイレクトする
    - 通常はID/パスワード入力画面に遷移する

#### Callback (GET: `/auth/callback` )

- `openid-client` の `client.callbackParams` と `client.callback` を呼び出す
- OPからリダイレクトされて戻ってきた後、ログイン完了ページ（ `callbackCompletedPath` ）にリダイレクトする
- `baseUrl` と繋げたURLがOPに登録しておく「コールバックURL」になる
- 個別に使用することはないはず

#### Logout (GET: `/auth/logout` )

- `openid-client` の `client.endSessionUrl` を呼び出す
- OPのログアウトエンドポイントにリダイレクトする
- OPでログアウト処理に成功するとログアウト完了ページ（ `logoutCompletedPath` ）にリダイレクトされて戻ってくる

#### UserInfo (ALL: `/auth/userinfo` )

- `openid-client` の `client.userinfo` を呼び出す
- レスポンス（ユーザー情報）をそのまま返す

#### Introspect  (ALL: `/auth/introspect` )

- `openid-client` の `client.introspect` を呼び出す
- レスポンスをそのまま返す

#### Refresh (ALL: `/auth/refresh` )

- `openid-client` の `client.refresh` を呼び出す
- ID Tokenに含まれるクレームを返す
- ID Tokenクレームにはセンシティブ情報が含まれることがあるため仕様検討中
    - claimsエンドポイントで明示的に取得した方がいい気がしている

#### Resource (GET: `/auth/resource?url=<resource-url>`)

- `openid-client` の `client.requestResource` を呼び出す
- リソースプロバイダーからのレスポンスを返す
- （実験中）

#### Revoke (ALL: `/auth/revoke` )

- `openid-client` の `client.revoke` を呼び出す
- 成功時は `204` を返す

#### Status (ALL: `/auth/status` )

- セッションのステータスを取得する
    - フックが返す `sessionStatus` と同じ内容
- OPにはアクセスしない

#### Claims (ALL: `/auth/claims` )

- ID Tokenに含まれるクレームを取得する
    - フックが返す `sessionClaims` と同じ内容
- OPにはアクセスしない

## 複数OPの設定例

```typescript:multiple-issuer.ts
import Elysia from "elysia";
import { OidcClient } from "elysia-openid-client";
import { SQLiteAdapter } from "elysia-openid-client/dataAdapters/SQLiteAdapter";

const baseUrl = "https://app.example.com";
const dataAdapter = new SQLiteAdapter(); // データアダプターは共通

const rp1 = await OidcClient.create({
  baseUrl,
  issuerUrl: "https://issuer.exmaple.com",
  clientMetadata: {
    client_id: "client-id",
    client_secret: "client-secret",
  },
  dataAdapter, // データアダプターを指定
});
const endpoints1 = rp1.getEndpoints();

const rp2 = await OidcClient.create({
  baseUrl,
  issuerUrl: "https://another-issuer.exmaple.com",
  clientMetadata: {
    client_id: "another-client-id",
    client_secret: "another-client-secret",
  },
  dataAdapter, // 1と同じデータアダプターを指定
  settings: {
    pathPrefix: "/another", // エンドポイントが被らないようにprefixを変更
  },
});
const endpoints2 = rp2.getEndpoints();

// フックは任意の1個を使用する（どれを使ってもよい）
const hook = rp1.getAuthHook({
  loginRedirectUrl: "/select", // ユーザーがOPを選べるように選択画面に飛ばす
});

new Elysia()
  .use(endpoints1) // それぞれのエンドポイントを適用
  .use(endpoints2)
  .guard((app) =>
    app
      .use(hook) // フックは1個
      .get("/", ({ sessionStatus }) =>
        sessionStatus ? "Logged in" : "Restricted",
      )
      .get("/status", ({ sessionStatus }) => sessionStatus)
      .get("/claims", ({ sessionClaims }) => sessionClaims),
  )
  .get("/select", ({ set }) => { // OP選択画面
    set.headers["Content-Type"] = "text/html";
    return `
<html>
<body>
<p><a href="/auth/login">Issuer</a></p>
<p><a href="/another/login">Another</a></p>
</body>
</html>
    `;
  })
  .get("/free", () => "Not restricted")
  .get("/logout", () => "Logout completed")
  .listen(80);
```

- 2個以上のOPと連携する場合はその分だけインスタンスが増える
- データアダプターとフックは共通のものを使用する
- 認証後にRPのエンドポイントを叩く場合、resolveフックやStatus/ClaimsエンドポイントからユーザーがどのOPを使用しているかを判別した上でそのOP用のエンドポイントを叩く必要がある
    - OP選択画面で行っている `/auth/login` と `/another/login` の選択をセッション情報を元に自動で行うということ
    - `Record<IssuerUrl, PathPrefix>` を用意しておくのが妥当か（初期化にも使える）

他の設定例は気が向いたら[Examples](https://macropygia.github.io/elysia-openid-client/ja/examples/multiple-issuer/)に追加予定。

## 余談

- テストには[Bunのビルトイン機能](https://bun.sh/docs/cli/test)を使用
    - 概ねJest・Vitestと同様
    - Coverageは取れるが[現時点では出力できない](https://github.com/oven-sh/bun/issues/4015)のでCodeCov等との連携はできない
- Linter/Formatterには[Biome](https://biomejs.dev/)を使用
    - まだ荒削りな部分もあるがJS/TSのみのプロジェクトなら大丈夫そう
        - 短絡評価させたい演算子をまとめてしまったり、import文が複雑だとformatterが整形時に壊したりする
    - HTML/CSS/YAML/Markdown辺りは[まだ未対応](https://biomejs.dev/internals/language-support/)
- 今更ながら[commitlint](https://github.com/conventional-changelog/commitlint)を導入
    - ルールはひとまず `@commitlint/config-conventional`
- ~~リリース管理には[Release Drafter](https://github.com/release-drafter/release-drafter)を使用~~
    - `package.json` の `veresion` とReleaseで発行するtagを同期させるために四苦八苦するなどした → [関連記事](https://zenn.dev/macropygia/articles/sync-release-drafter-tag-to-package-json)
    - 初期開発の段階では細かい修正で一々PRを経由するのが面倒なため[Changesets](https://github.com/changesets/changesets)に移行した
- ~~[TypeDoc](https://typedoc.org/)の出力をGitHub ActionsでGitHub Pagesにデプロイするワークフローを導入~~
    - ~~日本語READMEを追加するためにimportして使おうとしたら[Bunが非対応だった](https://github.com/oven-sh/bun/issues/2445)~~
    - ~~幸いCLIでならビルドできるのでBun.spawnでサブプロセスからビルドして力技で合体~~
    - ~~これがなかったらREADME作りで力尽きていた~~
    - 限界を感じたためドキュメントは[Starlight](https://starlight.astro.build/)に移行した
        - TypeDocはプラグインでStarlight内に組み込める
- GitHubリポジトリのcontribute導線は[Renovate](https://github.com/renovatebot/renovate)などで行われているIssuesを閉じてDiscussionsに誘導する方式にした
    - 果たして使われることはあるのだろうか
- Dependabotは[Bunに非対応](https://github.com/dependabot/dependabot-core/issues/6528)だった
    - 今回はGitHubの機能に全振りする予定なのでRenovateは使わず一旦npm-check-updatesを使ったマニュアル管理とする
- `npm publish` の `--provenance` フラグを導入
    - GitHub Actionsでデプロイしていればフラグと `id-token: write` の追加だけで対応できる
    - `package.json` に `"publishConfig": { "provenance": true }` を追記してもよい
