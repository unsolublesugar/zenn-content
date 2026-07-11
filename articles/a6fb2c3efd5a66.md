---
title: "Cloudflare Registrar + R2 で個人開発アプリの公開基盤を構築した記録"
emoji: "🌐"
type: "tech"
topics: [cloudflare, domain, claudecode, dns, 個人開発]
published: true
---

## はじめに

現在、個人開発でWebアプリを作っています。プロダクト自体は未リリースのため具体的なアプリ名は伏せますが、「独自ドメイン取得〜公開基盤の構築」までを数時間で一気に進めたので、その際の手順・設定値・ハマりどころを記事として残しておきます。

:::message
未リリースのプロダクトのため、本記事ではドメインを `myapp.app`、アセット配信用サブドメインを `assets.myapp.app` という仮名で表記します。
:::

作業には Claude Code（Fable 5）を活用しました。Cloudflare を使うのは今回が初めてで、ダッシュボードの画面や設定項目に馴染みがなく迷う場面も多かったのですが、DNS の仕様や GitHub Pages の証明書まわりなど不明瞭な点をその場で検証してもらいながら、帯域設計や配信構成といった設計方針を対話で固めていきました。特に「この構成で本当に大丈夫か？」と不安になる箇所を一つずつ潰していく場面で重宝しました。

### 今回やったこと

1. Cloudflare Registrar での独自ドメイン取得
2. DNS 設定一式（GitHub Pages 検証・メールなりすまし対策含む）
3. GitHub Pages（private リポジトリ + GitHub Actions ソース）での公開基盤構築
4. 正式公開までの coming-soon ゲート設計
5. 大容量アセットの Cloudflare R2 ハイブリッド配信化

## 構成の全体像（完成形）

```text
[ユーザー]
   │ https://myapp.app（apex・.app TLD＝HSTSプリロード済み）
   ▼
[Cloudflare DNS]（Registrar 一体・現状グレー雲＝DNS only）
   │ A レコード×4本（GitHub Pages の IP を直接指す）
   ▼
[GitHub Pages]（private リポジトリ・GitHub Pro・Source: GitHub Actions）
   │ HTML/CSS/JS/LP画像のみ配信（内部ドキュメントは成果物から除外）
   │
   └─ 大容量アセットだけ別オリジン:
      https://assets.myapp.app（Cloudflare R2・egress 無料）
      └─ 数十MB級のバイナリファイル数点（合計約75MB）
```

図中に出てくる用語を先に補足しておきます。

:::details 用語メモ：apex ドメイン
`www` などのサブドメインを付けない、ルート（頂点）のドメインのこと。本記事では `myapp.app` そのものを指します。「ルートドメイン」「ネイキッドドメイン」とも呼ばれます。
:::

:::details 用語メモ：TLD（トップレベルドメイン）
`.app`・`.com`・`.jp` など、ドメイン名の末尾にくる区分のこと。本記事で取得したのは `.app` TLD のドメインです。
:::

:::details 用語メモ：HSTS プリロード
HSTS（HTTP Strict Transport Security）は「このサイトには必ず HTTPS で接続せよ」とブラウザに指示する仕組みです。通常は初回アクセス時のレスポンスヘッダーで有効になりますが、「プリロード」は対象ドメインのリストが **ブラウザ本体にあらかじめ組み込まれている** ため、初回アクセスから HTTP 通信が一切発生しません。`.app` は TLD ごとこのリストに登録されているので、`.app` ドメインのサイトは例外なく HTTPS 必須になります。
:::

ポイントは帯域設計です。GitHub Pages には **100GB/月のソフトリミット** があります（公式ドキュメントの使用制限ページに「GitHub Pages サイトのソフト帯域幅の制限は 1 か月あたり 100 GB」と明記されています）。

https://docs.github.com/ja/pages/getting-started-with-github-pages/github-pages-limits

本アプリは1ファイル20〜30MB程度のバイナリアセットを複数読み込む構成で、1訪問あたり約25MBとすると **月4,000訪問で帯域が枯渇** する計算になります。そこで大容量アセットだけを Cloudflare R2（S3 互換のオブジェクトストレージ。egress＝下り転送が無料なのが特長）に逃がし、Pages 側を数MB/訪問以下に抑える方針にしました。

https://www.cloudflare.com/ja-jp/developer-platform/products/r2/

## 1. ドメイン取得（Cloudflare Registrar）

### なぜ Cloudflare Registrar にしたか

これまでドメインは国内のドメインプロバイダーサービスで取得することが多く、使い慣れていたのですが、今回はすこし事情が違いました。

- 普段使っている国内サービスでは、今回取りたかった `.app` TLD の **取り扱い自体がなかった**
- 取り扱いのある国内サービスでも、Cloudflare と比較して **価格が高い**、更新時の値上がりや **余計なオプション・手数料がかかる** 懸念があった

加えて、かねてから Cloudflare を一度使ってみたいという気持ちもあったので、今回は Cloudflare Registrar を選びました。

https://www.cloudflare.com/ja-jp/products/registrar/

決め手は以下です。

- **レジストリ卸値そのまま・マークアップなし**
  - 取得と更新が同額で「1年目だけ激安」のような撒き餌価格がない
- **価格: `.app` = US$14.20/年**（取得＆更新：2026年7月時点）
  - 参考: `.net` $11.86 / `.dev` $12.20
- **WHOIS 個人情報の秘匿（redaction）が無料・デフォルト有効**
  - 設定作業ゼロ
- `.app` は **TLD ごと HSTS プリロード済み**（ブラウザが最初から HTTPS でしか接続しない）
  - HTTPS（Secure Context）必須のブラウザ API を使うアプリと好相性

:::message
**Cloudflare Registrar の注意点**

- 取得したドメインは **ネームサーバーが Cloudflare 固定**（外部 NS へ変更不可）
- `.moe` など取り扱いのない TLD もある
- 請求は **USD 建て**
:::

### 手順

前述のとおり Cloudflare は今回が初めてなので、アカウント作成からのスタートです。

1. [サインアップ画面](https://dash.cloudflare.com/sign-up)からアカウント作成 → 2FA 設定
   - 後述しますが Google アカウント認証で作成しました
   - メールアドレスで登録する場合は **メール認証** を済ませないと登録に進めない
2. Billing で支払い方法を登録
3. Domain Registration → Register Domains → 検索 → 購入（登録者情報は英語表記推奨）

:::message
取得したドメインは60日間、他のレジストラへ転出（移管）できません。これは Cloudflare に限らない ICANN 共通のルールです。
:::

ドメイン購入の検索画面では、取得価格と更新価格が同額（$14.20）で表示されます。「卸値そのまま・撒き餌なし」が一目で伝わる画面です。

![](https://static.zenn.studio/user-upload/f07fcfe6ecc9-20260711.png)

### ハマりどころ: Google アカウント連携（OAuth）でサインアップすると 2FA が設定できない

Cloudflare のアカウント作成では、手軽さ重視で Google アカウントの OAuth 認証でサインアップしました。ところが、二段階認証（2FA）を設定しようとした画面で **「現在のパスワード」を要求されて詰みました**。

![](https://static.zenn.studio/user-upload/47cf0bbb616b-20260711.png)

OAuth サインアップなので Cloudflare のパスワードはそもそも設定しておらず、入力しようがありません。ドメインという重要資産を預けるのに 2FA が張れない状態です。

調べてみると Reddit に同じ症状の投稿があり、そこで示されていた **「パスワードリセットのフローを使ってパスワードを新規設定する」** という方法で解決できました。

https://www.reddit.com/r/CloudFlare/comments/1hhdl1t/how_can_i_set_up_2fa_on_my_account_when_i_use/

1. 一度ログアウトし、ログイン画面の「パスワードを忘れましたか？」からパスワードリセットを実行
2. Google アカウントに紐づいたメールアドレスにリセットメールが届く → そこから Cloudflare 用のパスワードを設定
3. 以後は「現在のパスワード」を入力できるようになり、2FA（認証アプリ）の設定が通る

![](https://static.zenn.studio/user-upload/22e31076401a-20260711.png)

OAuth ログイン自体はその後も併用可能です。「ソーシャルログインで作ったアカウントはパスワード起点のセキュリティ設定で詰む → リセットフローがパスワードの後付け手段になる」というのは、Cloudflare に限らず使える回避パターンだと思います。

## 2. DNS 設定（Cloudflare ダッシュボード）

ドメインが取得できたので、次は DNS 設定です。Cloudflare Registrar で取得したドメインは最初から Cloudflare DNS で管理されるため、ダッシュボードの DNS レコード画面からそのまま設定していきます。GitHub Pages 向けの検証用・本体用レコードに加えて、メールなりすまし対策の TXT レコードもここでまとめて登録しました。

:::message
レコードはすべて **プロキシオフ（グレー雲・DNS only）** で作成します。GitHub の DNS チェックと Let's Encrypt 証明書発行は、A レコードが GitHub の IP を直接返すことが前提のためです（オレンジ雲だと Cloudflare の IP が返って失敗します）。
:::

| タイプ | 名前 | 値 | 用途 |
|---|---|---|---|
| TXT | `_github-pages-challenge-<ユーザー名>` | （GitHub 発行のコード） | Verified domain（所有権証明） |
| A | `@` | 185.199.108.153 / 109.153 / 110.153 / 111.153 | GitHub Pages 本体 |
| CNAME | `www` | `<ユーザー名>.github.io` | www → apex は GitHub 側が自動リダイレクト |
| TXT | `@` | `v=spf1 -all` | メール送信サーバーなし宣言 |
| TXT | `_dmarc` | `v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s;` | なりすましメールの受信側拒否指示 |
| TXT | `*._domainkey` | `v=DKIM1; p=` | DKIM 空鍵（全署名無効） |

A レコードに設定する4つの IP アドレス（`185.199.108.153` など）は、GitHub 公式ドキュメントに記載されている GitHub Pages 用の値です。

https://docs.github.com/ja/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site

ポイントは3つあります。

- Cloudflare の「名前」欄はゾーン名を自動補完するので `_github-pages-challenge-...` の部分だけ入力する（フル指定すると二重になる）
- Cloudflare は権威 DNS なので反映はほぼ即時（GitHub 画面の「最大24時間」は一般論）
- **メールを使わないドメインこそ SPF/DMARC/DKIM の「使わない宣言」を置く** のがベストプラクティス（`info@` などの詐称メール防止）。将来メールが必要になれば Cloudflare Email Routing（無料転送）導入時に書き換えればOK

「レコードの追加」ダイアログでは、プロキシステータスのトグルが **デフォルトでオン（オレンジ雲）** になっている点が今回の要注意ポイントです。GitHub Pages 向けレコードは毎回オフに切り替えます。

![](https://static.zenn.studio/user-upload/931c2f43aad6-20260711.png)

全レコード登録後の一覧です。グレー雲が並び、R2 カスタムドメイン行のみオレンジ（Cloudflare 管理）になっています。

![](https://static.zenn.studio/user-upload/15f708443cb8-20260711.png)

DNS 画面上部の「推奨事項」パネルでは、未設定の項目があると Cloudflare が自動で指摘してくれます。この推奨事項に従って潰していくだけでも、設定漏れをかなり防げます。

![](https://static.zenn.studio/user-upload/ca1cb0ccb164-20260711.png)

設定後の検証コマンドです（作業環境が Windows だったため PowerShell で実行）。

```powershell
Resolve-DnsName "_github-pages-challenge-<ユーザー名>.myapp.app" -Type TXT
Resolve-DnsName "myapp.app" -Type A   # GitHub の IP が「直接」返ればグレー雲も確認できる
Resolve-DnsName "_dmarc.myapp.app" -Type TXT
Resolve-DnsName "test._domainkey.myapp.app" -Type TXT   # ワイルドカード確認は任意のセレクタで
```

## 3. GitHub Pages（private リポジトリ + GitHub Pro）

ホスティングは GitHub Pages です。GitHub Pro プランなら private リポジトリのまま Pages を公開できます。GitHub Pages × GitHub Actions の静的サイト構成は、以前コミュニティポータルサイトを構築した際にも採用しています。

https://zenn.dev/easy_easy/articles/89318e4d0acb4f

### Verified domain（乗っ取り防止）を先に設定する

サイト公開前のドメインには、第三者が自分の Pages にそのドメインを設定してしまう乗っ取りのリスクがあります。いわゆる「サブドメインテイクオーバー」と同種のリスクです。

https://jprs.jp/glossary/index.php?ID=0267

**リポジトリ設定ではなくアカウント設定**（github.com/settings/pages）の「Add a domain」で TXT 検証しておくと、自分のアカウント以外はこのドメインを Pages に使えなくなります。

![](https://static.zenn.studio/user-upload/be40f65769d7-20260711.png)

:::message
アカウント設定の Verified domain と、リポジトリ設定の Custom domain（後述）は **別物** です。名前も画面も似ているので混同に注意してください。
:::

検証成功後、アカウント設定側の Verified domains 一覧に緑の Verified バッジが表示されます。

![](https://static.zenn.studio/user-upload/4e733beac1a1-20260711.png)

### Build and deployment の Source は「GitHub Actions」を選択

ビルド工程のない静的サイトですが、あえて branch デプロイにしませんでした。

![](https://static.zenn.studio/user-upload/8680171e0e00-20260711.png)

理由は3つあります。

1. **branch デプロイはリポジトリの全ファイルを配信する**。private リポジトリの内部ドキュメント（CLAUDE.md や README 等）が `https://<ドメイン>/CLAUDE.md` で誰でも読める状態になる。Actions ソースなら配信物を明示的に組み立てられる
2. Jekyll ビルドを完全回避できる（branch デプロイはデフォルトで Jekyll が走る）
3. 後述の coming-soon ゲートのような「デプロイ時の分岐」を仕込める

:::details 用語メモ：Jekyll
GitHub Pages に組み込まれている静的サイトジェネレーター。branch デプロイでは、リポジトリの内容がデフォルトで Jekyll によってビルドされてから配信されます（`_` 始まりのファイルが除外される等の暗黙の挙動もある）。今回のように「置いたファイルをそのまま配信したい」場合には不要な工程です。

https://docs.github.com/ja/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll
:::

ワークフローの骨子はこうです。**載せるものだけを列挙する「明示コピー方式」** で成果物を組み立てます。

```yaml
permissions: { contents: read, pages: write, id-token: write }
steps:
  - uses: actions/checkout@v4
  - name: Assemble site
    run: |
      mkdir -p _site
      cp index.html styles.css main.js _site/
      cp -r app js images _site/   # 明示コピー方式＝載せるものだけ列挙
  - uses: actions/upload-pages-artifact@v3
    with: { path: _site }
  - uses: actions/deploy-pages@v4
```

### カスタムドメインと証明書

1. リポジトリの Settings → Pages → Custom domain に `myapp.app` を入力 → Save（**先に A レコード必須**。Save した瞬間に DNS チェックが走る）
2. Let's Encrypt 証明書が自動発行される。`.app` は HSTS プリロード済みのため、証明書が出るまで **一切表示できない**（HTTP フォールバック不可）
3. 発行後に Enforce HTTPS を有効化（API でも可: `gh api -X PUT repos/<owner>/<repo>/pages -F https_enforced=true`）

状態確認は API が便利です。

```bash
gh api repos/<owner>/<repo>/pages \
  --jq '{cname, https_enforced, certificate_state: .https_certificate.state, domains: .https_certificate.domains}'
```

:::message
**ハマりどころ**: `www` の CNAME を **後から** 追加したため、証明書の www カバレッジの再発行が長引きました（`www.` アクセスは `*.github.io` の証明書が返って TLS エラーになる）。apex と www の DNS レコードは **最初から両方** 用意してから Custom domain を保存するのがきれいです。長引く場合は Custom domain の削除→再入力で再発行をトリガーできます。
:::

この待ち時間中の Custom domain 設定画面では、apex は「DNS valid for primary」なのに www は `InvalidDNSError` の警告、Enforce HTTPS は「certificate has not yet been issued」でグレーアウト、という3つの状態が同時に見られました。

![](https://static.zenn.studio/user-upload/94b34839adc8-20260711.png)

なお、この www の `InvalidDNSError` は DNS 側の実態（CNAME は正しく引ける状態）と食い違っており、**GitHub 側のチェック結果が古いだけ** でした。警告横の「Check again」を押すと再チェックが走り、`DNS check successful` に変わります。

![](https://static.zenn.studio/user-upload/dca19121cd21-20260711.png)

エラー表示を見ても慌てず、まず実際の DNS を自分で引いて（`Resolve-DnsName` / `dig`）実態と突き合わせ、食い違っていたら「Check again」を押す、という切り分けが有効でした。

## 4. coming-soon ゲート（正式公開までダミーページ）

プロダクトは未リリースなので、正式公開まではダミーページだけを配信します。ここでは「Cloudflare でアクセスを塞ぐ」のではなく「**本体をそもそもデプロイしない**」方式にしました。配信されていないものは、どんな抜け道からも見えません。

- ワークフローに `env: PUBLISH_MODE: comingsoon` を追加。comingsoon の間は自己完結のダミー HTML をすべての入口パス（`index.html`・`app/index.html` など）にコピーして配信する（エッジのリライト設定が何であってもダミーが出る）
- ダミーページは `noindex` 付き・プロダクト名も伏せて絵文字1つ＋「Coming Soon」のみ（タイトル・favicon 含め名前が漏れない）
- 公開時は `PUBLISH_MODE: full` に1行変えてマージするだけ
- 「オーナーの明示指示があるまで comingsoon を維持」を CLAUDE.md に明記して、AI エージェント経由の事故切り替えも防止

実際のワークフローは、前述の「Assemble site」ステップを `PUBLISH_MODE` で分岐させる形です。

```yaml
env:
  PUBLISH_MODE: comingsoon   # 公開時に full へ変更

steps:
  - uses: actions/checkout@v4
  - name: Assemble site
    run: |
      mkdir -p _site/app
      if [ "$PUBLISH_MODE" = "comingsoon" ]; then
        # 全入口パスをダミーページで上書き
        cp comingsoon.html _site/index.html
        cp comingsoon.html _site/app/index.html
      else
        # 本体を明示コピー（大容量アセットは R2 配信のため含めない）
        cp index.html styles.css main.js _site/
        cp -r app js images _site/
      fi
```

検証は「ダミーが出ること」だけでなく「本体が本当に消えていること」まで確認します。

```bash
curl -s https://myapp.app/ | grep "Coming Soon"       # ダミーが出る
curl -s -o /dev/null -w "%{http_code}" https://myapp.app/styles.css       # → 404
curl -s -o /dev/null -w "%{http_code}" https://myapp.app/assets/model-a.bin  # → 404
```

## 5. R2 ハイブリッド化（大容量アセットの帯域対策）

冒頭の帯域試算のとおり、数十MB級のバイナリアセットを GitHub Pages から配信すると月4,000訪問で帯域が尽きます。そこでアセットだけを Cloudflare R2 の別オリジンから配信する構成にしました。R2 の詳細は公式ドキュメントを参照してください。

https://developers.cloudflare.com/r2/

### R2 側（Cloudflare ダッシュボード）

R2 の画面へは、ダッシュボード左メニューの「ビルド」配下にある「ストレージとデータベース」→「R2 オブジェクト ストレージ」からアクセスします。

![](https://static.zenn.studio/user-upload/7919e9f568bc-20260711.png)

やることは3つです。

1. バケット作成（ロケーション自動）。初回はカード登録が必要だが、無料枠はストレージ 10GB・読み取り1,000万回/月・**下り転送（egress）は完全無料**
2. アセットをアップロード。**オブジェクトキーをリポジトリ内のパスと同一に** しておくのがポイント（コード側が基底 URL の切り替えだけで済む）。ライセンス表記が必要なアセットは表記ファイルも同梱する
3. カスタムドメイン接続: バケット設定 → `assets.myapp.app`（同一アカウントのゾーンなので DNS レコードは自動追加される）。`r2.dev` の公開 URL は使わない（レート制限付きの開発用）

アップロードは、バケットの画面にファイル/フォルダをドラッグ＆ドロップするだけです。

![](https://static.zenn.studio/user-upload/581bf39f8fbb-20260711.png)

:::message alert
ライセンスで再配布が禁止されているアセットは **公開ストレージに置かない** よう注意してください。公開バケットに置いたファイルは URL を知っていれば誰でも取得できてしまいます。
:::

続いて **CORS ポリシー** を設定します（バケット設定）。編集画面を開くとデフォルトで以下の雛形が入っていますが、これは Cloudflare のサンプル値なので上書きして問題ありません。

```json
[
  {
    "AllowedOrigins": [
      "http://localhost:3000"
    ],
    "AllowedMethods": [
      "GET"
    ]
  }
]
```

本プロジェクトでは localhost から R2 を読まない（ローカル開発は同梱ファイルへの相対パスで完結する）ため、許可オリジンは本番のみに置き換えました。

```json
[
  {
    "AllowedOrigins": ["https://myapp.app"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["Content-Length", "ETag"],
    "MaxAgeSeconds": 86400
  }
]
```

最後に **キャッシュルール** を設定します。ここが迷いポイントで、設定場所は **R2 の画面ではなくゾーン側**（対象ドメイン → Caching → Cache Rules）にあります。

- 式: `(http.host eq "assets.myapp.app")`
- キャッシュの適格性: 対象 ／ エッジ TTL: キャッシュ制御ヘッダーを無視して1ヶ月

:::message
式の **ホスト名限定を忘れると Pages 側の HTML まで長期キャッシュされてしまいます**。必ず `assets.` サブドメインに限定してください。
:::

![](https://static.zenn.studio/user-upload/e35aecbec8b1-20260711.png)

![](https://static.zenn.studio/user-upload/9a4e9388184d-20260711.png)

### コード側（ローカル開発フォールバックの設計）

JS の設定ファイル冒頭で、基底 URL をホスト名判定で切り替えます。**本番ドメインのときだけ R2、それ以外（localhost・LAN プレビュー）は git 同梱ファイルへの相対パス**とすることで、ローカル開発は R2 なしで完結します。

```js
const ASSETS_BASE =
  location.hostname === 'myapp.app' || location.hostname === 'www.myapp.app'
    ? 'https://assets.myapp.app/'
    : '';
// 使用側: url: ASSETS_BASE + 'assets/model-a.bin'
```

あわせてデプロイワークフローの full モードから大容量アセットのディレクトリを除外します（git には残す＝ローカル開発の実体）。

### 検証コマンド

CORS は「許可オリジンに返ること」だけでなく「**許可外オリジンに返らないこと**」まで確認します。

```bash
# CORS: 許可オリジンには ACAO が返る（Range 付き GET で 206 も確認）
curl -s -H "Origin: https://myapp.app" -D - -o /dev/null -r 0-0 \
  https://assets.myapp.app/assets/model-a.bin | grep -iE "^HTTP|access-control"

# 許可外オリジンには ACAO が「返らない」ことも必ず確認
curl -s -H "Origin: https://evil.example.com" -D - -o /dev/null -r 0-0 \
  https://assets.myapp.app/assets/model-a.bin | grep -ic access-control-allow-origin
```

結果は、全ファイル 200・許可オリジンのみ `access-control-allow-origin` 返却・Range リクエスト 206 対応・`cf-cache-status: MISS` → 2回目以降エッジキャッシュ作動、となり期待どおりでした。

## 6. クリーンURL化（進行中）

`/` や `/app` といったパスを、リダイレクトで実体の HTML ファイル名に変えることなく、そのままの URL で配信する計画も進めています。Cloudflare プロキシ + Transform Rules を使う方式で、こちらは **まだ作業途中** です。

事前準備として、ローカルで URL 書き換え相当のサーバーを立てて実機検証だけ済ませました。Python の `SimpleHTTPRequestHandler.translate_path` をオーバーライドして、`/` や `/app` を同一 URL のまま実体ファイルにマッピングするサーバーです。拡張子なしのパスでもアプリが正しく動作することを確認できました。

本番適用は「www の証明書発行後 → SSL/TLS を Full (Strict) に → apex の A レコードをオレンジ雲化 → Transform Rules 追加」という順序で行う必要があり、順序を守らないと証明書発行の妨害やリダイレクトループが起きます。このあたりは実施後に別途まとめるかもです。

## 学び・ハマりどころまとめ

1. **`.app` TLD は HSTS プリロード済み**。証明書が出るまで1バイトも表示できない。逆に言えば「HTTPS 対応」は TLD 選定の時点で強制完了する
2. **Google OAuth でサインアップしたアカウントは 2FA 設定で「現在のパスワード」を要求されて詰む**。パスワードリセットのフローでパスワードを後付けすれば解決（Cloudflare 以外のサービスでも使える回避パターン）
3. **GitHub の Verified domain（アカウント設定）と Custom domain（リポジトリ設定）は別物**。前者はサイト公開前にやる乗っ取り保険
4. **www の DNS レコードは最初から作っておく**。後出しすると証明書の再発行待ちが発生する
5. **private リポジトリ × branch デプロイは内部ドキュメントが漏れる**。Actions ソース + 明示コピー方式が安全（GitHub Pro なら private のまま Pages 公開可）
6. **メール不使用ドメインの三点セット**（SPF `-all`・DMARC reject・DKIM 空鍵）はドメイン取得日にやるのが一番安い保険
7. **R2 の egress 無料は大容量静的アセットの特効薬**。オブジェクトキーをリポジトリパスと揃えると、コード側は「基底 URL の切り替え」1点で済み、ローカル開発のフォールバックも自然に手に入る
8. **CORS は「許可外に返らないこと」までテストする**・**キャッシュルールはホスト限定を忘れない**
9. **公開前ゲートは「配信しない」が最強**。アクセス制御を足すより、デプロイ成果物に含めない方が確実で、切り替えも1行

## おわりに

ドメイン取得からデプロイ基盤・帯域対策まで、数時間でここまで整えられたのは、Cloudflare（Registrar・DNS・R2）と GitHub Pages の組み合わせがよくできていることに加え、Claude Code（Fable 5）と対話しながら不明点の検証と設計判断を並行して進められたことが大きかったと感じます。

特に「Registrar と DNS とオブジェクトストレージが同一アカウントに揃っている」ことによるシームレスさは、国内プロバイダーで取得してから外部サービスに繋ぎ込む従来の手順と比べて体験が良かったです。

プロダクト本体は現在鋭意開発中です。リリースの際にはまた記事にしようと思います。
