# Shopify + Nuxt 基礎知識

このドキュメントは、Shopify と Nuxt で headless EC サイトを作るときに、最初に押さえるべき基礎をまとめたものです。

今回の前提は「Nuxt で購入者向けの storefront を作り、Shopify を commerce backend として使う」構成です。ただし、Storefront API だけを学べば完結するわけではありません。商品表示、カート、ログイン、購入、管理系操作、レビュー連携は、それぞれ使う API と責務が違います。

次の Shopify プロジェクトでは、まずこのドキュメントで全体像を確認し、そのあと各公式ドキュメントや実装へ進む流れを想定しています。

## 第0章: Shopify + Nuxt ではどの API を使うのか

Headless Shopify では、Nuxt が画面とアプリケーションロジックを担当し、Shopify が商品、価格、在庫、カート、チェックアウト、注文などの commerce 機能を担当します。

ここで重要なのは、購入者向けサイトだからといって、すべてを Storefront API だけで扱うわけではないことです。Storefront API は storefront 体験の中心ですが、ログインは Customer Account API、管理系操作は Admin API、決済の完了は Shopify Checkout、レビューや独自決済は外部 API が関係します。

### API の使い分け

| 領域 | 主に使うもの | Nuxt での扱い |
| --- | --- | --- |
| 商品、コレクション、ブログ、検索 | Storefront API | 公開可能な取得はブラウザまたは server 側から呼べる |
| カート | Storefront API の Cart オブジェクトと mutation | Cart ID を保持し、Shopify の Cart を正として扱う |
| 購入導線 | Shopify Checkout | Cart の `checkoutUrl` から Shopify Web Checkout へ遷移する |
| 顧客ログイン、注文履歴、住所 | Customer Account API | Nuxt server 側のセッションと httpOnly Cookie で保持する |
| 商品管理、メタフィールド定義、メタオブジェクト定義 | Admin API | Nuxt のブラウザ側から直接呼ばず、server 側で扱う |
| レビュー | Judge.me など | public/private token の違いを分け、秘密情報は server 側に閉じる |
| Stripe 決済 | Stripe API / Stripe Checkout | Shopify の注文・在庫・決済の正本をどうするか別途設計する |

### Nuxt 側の基本方針

Nuxt のブラウザ側から直接扱ってよいのは、公開前提のデータ取得に限ります。Storefront API の public token を使う構成はあり得ますが、private token、Admin API token、Stripe secret key、Judge.me private token のような秘密情報は、必ず Nuxt の server route や backend に閉じます。

検索語、絞り込み条件、ページ番号のように URL で共有したい状態は `route.query` に置きます。Cart ID やログイン状態は、httpOnly Cookie と Nuxt server 側の session で保持します。Nuxt 4 では `nuxt-auth-utils` を使うと、sealed cookie session、`useUserSession()`、`setUserSession()`、`getUserSession()`、`requireUserSession()` を使った構成にできます。商品価格、在庫、カート金額、checkoutUrl、注文は Shopify 側を正本にし、Nuxt 側で再計算しすぎないようにします。

## 第1章: GraphQL を先に理解する

Shopify の Storefront API と Admin API は GraphQL を中心に使います。REST のように「エンドポイントごとに固定レスポンスを受け取る」のではなく、必要なフィールドを query で選んで取得します。

GraphQL では、読み取りは query、作成や更新は mutation です。Shopify の mutation では、成功時のデータだけでなく `userErrors` を確認する実装が重要になります。HTTP として成功していても、入力値や権限の問題で mutation 内の `userErrors` に失敗理由が返ることがあります。

一覧取得では connection を理解する必要があります。Shopify では `products(first: 20)` のように件数を指定し、`pageInfo.hasNextPage` と `pageInfo.endCursor` を見ながら次ページを取得します。大量データを一度に取得する前提ではなく、cursor pagination を前提に設計します。

Admin API では query cost と rate limit も考慮します。必要以上に深いフィールドや大量 connection を取得するとコストが上がります。Storefront API は購入者トラフィック向けに設計されていますが、bot や不正アクセス対策、input array の上限、checkout-level throttle などは意識します。

## 第2章: Shopify の機能とレイヤー

Shopify の標準リソースは、画面上の UI 名称と一対一で考えるより、役割ごとに分けると理解しやすくなります。

商品は Product、Variant、Option、Media を中心に構成されます。Product は商品そのもの、Variant はサイズや色など購入可能な単位、Option は variant を分ける軸、Media は画像や動画などの表示素材です。Nuxt の商品詳細ページでは、Product を表示しつつ、実際にカートへ入れるときは Variant ID を使います。

コレクションは、商品をまとめるための単位です。カテゴリページ、特集ページ、ブランド別一覧、キャンペーン一覧などに使えます。画面上の「カテゴリ」と Shopify の Collection は近い場面もありますが、Shopify taxonomy、product type、tag、collection はそれぞれ役割が違うため、運用前に整理しておく必要があります。

タグは柔軟なラベルです。検索や絞り込みに使えますが、自由度が高いぶん、命名規則や運用ルールがないと崩れやすくなります。タグを主要な分類軸にしすぎるより、Collection、product type、metafield で表現できるものはそちらを優先します。

ブログは Blog と Article で構成されます。EC サイトでは、コラム、ニュース、読み物、商品紹介記事などに使えます。microCMS など外部 CMS を使う選択肢もありますが、Shopify に寄せるなら Blog / Article を Storefront API で読む構成が候補になります。

メタフィールドは、既存リソースへ追加情報を持たせる仕組みです。たとえば商品に「成分」「サイズガイド」「配送目安」「注意事項」を追加できます。Storefront API で読むには、metafield definition の `access.storefront` を公開設定にする必要があります。

メタオブジェクトは、独立した構造化データを作る仕組みです。著者プロフィール、素材マスター、ブランド情報、FAQ、サイズ表など、複数フィールドを持つ再利用可能なデータに向いています。単一リソースに値を足すなら metafield、独立した構造を作って複数箇所から参照するなら metaobject、と考えると判断しやすいです。

## 第3章: ストア種別を選ぶ

Shopify には、学習や実装開始に向いた dev store、クライアントへ引き渡す client transfer store、実運用する通常ストア、既存店舗に参加する collaborator access があります。

学習、検証、アプリや headless 実装の初期開発では dev store を基本にします。dev store は本番データを壊す心配が少なく、API や theme、app の検証に向いています。一方で、実決済や一部機能には制限があります。

クライアントへ新規店舗を納品する前提なら client transfer store を検討します。構築中は開発者側で管理し、完成後にクライアントへ移管する流れです。既に稼働している本番店舗を触る場合は、collaborator access や権限管理を使い、開発用ストアと本番ストアの token や設定を混ぜないようにします。

Nuxt 側では、ストアごとに domain、Storefront API token、Customer Account API 設定、Admin API token、Webhook、外部サービス連携が変わります。環境変数を dev / staging / production で分け、誤って本番データにテスト注文やテストレビューを混ぜない設計にします。

## 第4章: Admin API は管理レイヤー

Admin API は、Shopify 管理画面や業務操作を拡張するための API です。購入者向け画面を作る Storefront API とは責務が違います。

Admin API で扱うものは、商品作成・更新、在庫や注文の管理、metafield definition、metaobject definition、管理系の automation などです。これらは強い権限を持つため、Nuxt のブラウザ側から直接呼んではいけません。

Nuxt で Admin API が必要になる場合は、`server/api` や別 backend を経由します。たとえば、Storefront API で読ませたい metafield を公開するために Admin API で definition を作成・更新する、管理画面から独自データを同期する、といった用途です。

Admin API token は秘密情報です。公開 runtime config、ブラウザ JS、GitHub、ログに出さないようにします。必要な access scope だけを付与し、最小権限で設計します。

## 第5章: カートは Shopify の Cart を正にする

Headless Shopify のカートは、Nuxt の配列だけで完結させるものではありません。Storefront API の Cart を作成し、Cart ID を保持し、Shopify 側の Cart を正として扱います。

基本の流れは、`cartCreate` で Cart を作り、`cartLinesAdd` で variant を追加し、`cartLinesUpdate` で数量を変え、`cartLinesRemove` で削除します。Cart を再取得すると、現在の line item、価格、割引、配送に関係する情報、`checkoutUrl` などが返ります。

Nuxt 側では、Cart ID の保存場所を決めます。未ログインでも使うなら Cookie や local storage を検討できますが、サーバーサイド処理やログイン連携も考えるなら Cookie + server route の設計が扱いやすくなります。

カート金額は Nuxt 側で独自計算しない方が安全です。価格、割引、税、配送、Markets による通貨や国別価格は Shopify 側のルールが関係します。画面表示では Shopify から返る `cost` や money fields を使います。

ログイン済み顧客とカートを結びつける場合は `buyerIdentity` が関係します。サーバーサイドから Storefront API を呼ぶ場合は、Buyer IP ヘッダーなど公式が求める条件も確認します。

## 第6章: ログインは Customer Account API を第一候補にする

Shopify の顧客ログインは、Storefront API だけで考えない方がよい領域です。Storefront API には legacy customer account 向けの `customerAccessTokenCreate` などがありますが、現在の公式推奨は Customer Account API です。

Customer Account API は、顧客本人にスコープされたデータを安全に扱うための API です。注文履歴、住所、プロフィールなど、ログイン済み顧客の private data を扱うときに関係します。Hosted login や passwordless login の流れも含めて、Shopify 側の customer account の仕組みに沿って設計します。

Nuxt 側では、認証フロー、callback route、session、ログアウト、カートとの連携を整理します。基本構成としては、Supabase などの別 DB を最初から立てなくても、Nuxt server 側の session と httpOnly Cookie で十分現実的です。

Nuxt 4 では、公式レシピとして `nuxt-auth-utils` を使った session 管理があります。`setUserSession(event, data)` でログイン成功後の session を保存し、`useUserSession()` でフロント側からログイン状態を参照します。サーバー API を保護したい場合は、対象の API route で `requireUserSession(event)` を呼び、未ログインなら 401 にします。

Customer Account API と組み合わせる場合、Shopify の認証 callback を Nuxt server route で受け、顧客を識別するために必要な最小限の情報を user session に入れます。アクセストークンなど server 側でだけ使う値は、ブラウザ JS から読める状態にしません。`nuxt-auth-utils` には server route だけで参照する `secure` 領域がありますが、cookie size の上限があるため、保存する値は最小限にします。

この構成なら、まずは「Shopify Customer Account API + Nuxt server session + httpOnly Cookie」で始められます。顧客独自プロフィール、ポイント、会員ランク、長期的な監査ログなど、Shopify 外に永続化したいアプリ固有データが出てきた段階で、Supabase や別 DB を検討します。

「ログイン済みか」だけではなく、「この顧客がこの注文や住所を読んでよいか」まで API 側のスコープで守られる設計にします。

既存実装や要件によって legacy customer account を扱うことはあります。その場合も、新規プロジェクトでは Customer Account API を第一候補にし、legacy の扱いは理由を明記します。

## 第7章: 購入・決済は Shopify Checkout を標準導線にする

Headless Shopify の標準的な購入導線は、Storefront API の Cart から `checkoutUrl` を取得し、Shopify Web Checkout へ遷移する流れです。決済、配送、税、割引、Shopify Functions、Checkout UI extensions など、購入完了に近い複雑な領域は Shopify Checkout に任せます。

Checkout API は非推奨化が進んでいるため、新規実装では Storefront Cart API を前提にします。Nuxt 側で独自のチェックアウト画面を作り込む前に、Shopify Checkout を使えるかをまず検討します。

Stripe を組み合わせる場合は、単に「決済画面を Stripe にする」と考えるだけでは足りません。Stripe Checkout で決済した結果を、Shopify の注文、在庫、顧客、通知とどう整合させるかを設計する必要があります。

商品、価格、在庫、注文の正本を Shopify に置きたいなら、Shopify Checkout を使う方が自然です。Stripe を別で使う場合は、Shopify を商品カタログとして使うのか、注文管理まで Shopify に寄せるのか、外部システムと同期するのかを先に決めます。

## 第8章: 検索は URL クエリを正にする

検索機能では、Nuxt の URL クエリを検索状態の正本にすると扱いやすくなります。検索語、タグ、価格帯、在庫、並び順、ページ番号を URL に表現できれば、リロードしても状態が復元でき、URL 共有もしやすくなります。

Shopify 側では、Storefront API の `products(query:)`、`search` query、collection filter などを要件に応じて使います。サイト全体検索なのか、特定 collection 内の絞り込みなのかで選ぶ API や query が変わります。

Nuxt 側では、ユーザー入力をそのまま Shopify の検索構文として渡さない方が安全です。画面上の検索条件をアプリ側で検証し、許可した条件だけを Shopify API の query や filter に変換します。

ページネーションは GraphQL の cursor pagination と URL 状態の両方を考えます。単純な `page=2` だけで扱うのか、cursor を URL に持つのか、検索結果の UX に合わせて決めます。

## 第9章: レビューは Shopify 標準外として扱う

Judge.me は Shopify の外部レビューサービスです。Shopify 商品ページにレビューを表示できますが、Shopify 標準の Storefront API だけで完結する機能ではありません。

Judge.me を使う場合は、まず widget を埋め込むのか、Judge.me API で取得して Nuxt 側で表示するのかを決めます。テーマ向け widget は Shopify theme 前提の部分があるため、headless Nuxt でそのまま使えるかは確認が必要です。

API で扱う場合、Public API token と Private API token の違いを分けます。Private token は server 側に閉じ、ブラウザへ出しません。商品 ID、handle、Shopify product ID、Judge.me 側の product ID の紐づけも確認します。

Shopify 側には metafield や metaobject でレビュー的なデータを表現する選択肢もあります。標準 metaobject の `product_review` を使うのか、Judge.me を正本にするのか、SEO や構造化データ、管理画面での運用も含めて決めます。

## 次案件開始時のチェックリスト

新しい Shopify + Nuxt プロジェクトを始めるときは、最初に次を確認します。

- [ ] 開発に使うストア種別は dev store か、client transfer store か、本番既存ストアか
- [ ] Storefront API token、Customer Account API、Admin API token の用途と保管場所を分けたか
- [ ] Nuxt のブラウザ側から秘密情報を使っていないか
- [ ] 商品、価格、在庫、カート、注文の正本を Shopify に置く方針か
- [ ] カートは Storefront Cart API を使い、購入は `checkoutUrl` から Shopify Checkout へ進める方針か
- [ ] ログインは Customer Account API と Nuxt server session + httpOnly Cookie を第一候補として検討したか
- [ ] 保護 API route では `requireUserSession()` で session を確認しているか
- [ ] メタフィールドとメタオブジェクトの使い分けを決めたか
- [ ] 検索条件を URL クエリに表現できるか
- [ ] Judge.me や Stripe など外部サービスの秘密情報を server 側に閉じたか
- [ ] GraphQL query は必要なフィールドだけを取得し、pagination と errors / userErrors を扱っているか

## 参考ドキュメント

- [Custom storefronts](https://shopify.dev/docs/storefronts/headless/getting-started)
- [Building with the Storefront API](https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api)
- [Create and update a cart with the Storefront API](https://shopify.dev/storefronts/headless/building-with-the-storefront-api/cart/manage)
- [Building with the Customer Account API](https://shopify.dev/storefronts/headless/building-with-the-customer-account-api)
- [Sessions and Authentication - Nuxt 4 Recipes](https://dev.nuxt.com/docs/4.x/guide/recipes/sessions-and-authentication)
- [nuxt-auth-utils](https://nuxt.com/modules/auth-utils)
- [GraphQL Admin API reference](https://shopify.dev/docs/api/admin-graphql)
- [Shopify API authentication](https://shopify.dev/docs/api/usage/authentication)
- [Shopify API limits](https://shopify.dev/docs/api/usage/limits)
- [Products and collections](https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/products-collections)
- [Retrieve metafields with the Storefront API](https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/products-collections/metafields)
- [About metafields](https://shopify.dev/docs/apps/build/custom-data)
- [About metaobjects](https://shopify.dev/docs/apps/build/metaobjects)
- [Create dev stores](https://shopify.dev/docs/api/development-stores)
- [Stores - Dev Dashboard](https://shopify.dev/docs/apps/build/dev-dashboard/stores)
- [Judge.me widgets overview](https://judge.me/help/en/articles/8415708-overview-of-the-judge-me-widgets)
- [Using Judge.me API](https://judge.me/help/en/articles/8409180-using-judge-me-api)
- [Stripe Checkout](https://docs.stripe.com/payments/checkout)
