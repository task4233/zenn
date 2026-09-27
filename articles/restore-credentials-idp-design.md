---
title: "TBD"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [android, webauthn, passkey, idp, authentication]
published: false
---

<!--
2026-09-27: プロダクト編は restore-credentials-product-guide.md に移した（構成メモ: restore-credentials-product-outline.md）。
このファイルは技術編の素材として残す（特に Q9〜Q30 の IdP 向けの論点）。技術編の構成は別途検討する。

下書き（フローのみ）。記事 ① / 2 本構成の本編（全体像と判断材料）。
- 記事 ② への導線: TL;DR（読み方ガイド）/ API 章末 / 試せるもの章（メインの導線）/ Q11（実測方法）/ Q26〜30 / まとめ
  導線は「② を読むと何が分かるか」を具体的に書き、好奇心の隙間を作る。リンク先の節名は ② の見出しと揃える
- TODO: ② のリンクは公開後の URL に差し替える（Zenn のユーザー名が task4233 であることを要確認）。① より先に ② を公開するか、同時に公開する
- 流れ: なぜ今 → 何が困っていたか → 何でありどう動くか → 既存技術と何が違うか → 試せるか → 自社に入れるとき何を決めるか
  各章が、直前の章を読んだ読者が抱く疑問に答えるように並べる
- 視座: API の解説ではなく「アカウントにログイン経路が 1 本増える」という見方
  （参考: kokukuma 氏の Qiita 記事群 — 評価軸つき比較、Weakest Link、サービス特性ごとの結論）
- 疑問点の章は Q&A 形式。読者層ごとに 3 つに分け、各回答に根拠の種類（実測 / 公式 / 考察 / 未検証）を付ける
- メイン読者: IdP / RP エンジニア、PM。Android 側の実装・検証の詳細は記事 ②（restore-credentials-android-testing）
- 社内情報（アプリ名・社内設計・社内ログ）は書かない
-->

## TL;DR

- TODO: 2027 年 4 月から Google Play で「ゼロタップでのログインの復元」が必須になる
- TODO: Restore Credentials は、端末移行後の初回起動でユーザーの操作なしにサインインさせる仕組み。中身は UV なしの WebAuthn クレデンシャル
- TODO: 「passkey と同じサーバ実装で扱える」が、サーバからは passkey と区別できず、セッションを失効させても止まらない。同じポリシーで扱うと Weakest Link になりうる
- TODO: 導入前に決めること — 保証レベル・種別の管理・失効。

:::message
**この記事の読み方**
- PM の方: 「なぜ今」〜「既存技術との差分」と、Q1〜Q8
- IdP / サーバの方: 全体と、特に Q9〜Q25
- Android の方: 「API とは何か」と Q26〜Q30。実装と検証の詳細は姉妹記事にまとめました
:::

<!-- 導線 1: 読み方ガイド。Android エンジニアに ② の存在を最初に知らせる -->
- TODO: 姉妹記事のリンクカード
https://zenn.dev/task4233/articles/restore-credentials-android-testing

## なぜ今話題になっているのか

- TODO: Google Play の技術的な品質要件「ゼロタップでのログインの復元」— 2027 年 4 月から必須
- TODO: Block Store と統合済みのアプリは 2026/9/30 までに対応していれば準拠扱い（経過措置）
- TODO: 規制対象の業種（金融・医療など）には免除の申請手段がある
- TODO: 対応しないと何が起きるか（Play 上の扱い）— 公式の記述の範囲で書く

## どのような課題を解決するのか

- TODO: 端末を買い替えると、アプリは移行できてもログイン状態は移行できない → 再ログインで離脱する / パスワードを忘れてリカバリに流れる
- TODO: ユーザーにとっての課題（手間・離脱）と、サービスにとっての課題（離脱・サポート問い合わせ・リカバリ経路への負荷）
- TODO: Restore Credentials の解決の仕方を 1 文で — 旧端末でサイレントに鍵を作り、端末移行で運び、新端末の初回起動でサイレントにサインインする
- TODO: 予告 — 「既存の Block Store や Auto Backup ではだめなのか」は、仕組みを説明したあとで比較する

## Restore Credentials API とは何か

<!-- 細かい実装は記事 ② へ。ここは「どう動くか」を掴める粒度にとどめる -->

- TODO: 図 — 作成（旧端末）→ バックアップ → 端末移行 → 取得（新端末）→ サーバ検証 のシーケンス図
- TODO: 3 つの操作 — `CreateRestoreCredentialRequest` / `GetRestoreCredentialOption` / `ClearCredentialStateRequest`（数行のコード例）
- TODO: クラウド経路とローカル経路（`isCloudBackupEnabled`）
- TODO: サーバから見ると WebAuthn の登録・認証そのもの。origin は `android:apk-key-hash:` 形式
- TODO: 効く場面と効かない場面（3 状態の表: サインアウトのみ / アンインストール → 再インストール / 端末移行）

<!-- 導線 2: 「API は 3 つだけ、簡単そう」という楽観が生まれた直後に置く -->
- TODO: 一文の導線 — 「API は 3 つだけですが、公式サンプルのとおりに書くと restore によるサインインは完了しません。理由と 1 行の修正は姉妹記事の『`RestoreCredential` は `PublicKeyCredential` のサブクラスではない』で説明しています」

## 既存技術との差分は何なのか

<!-- API を理解した読者の「既存の仕組みではできないの？」に答える章。評価軸つきの比較表を置く -->

- TODO: 比較対象
  - Auto Backup にトークンや Cookie を含める（アンチパターンの例として）
  - Block Store にトークンを保存する（既存の方式、経過措置の対象）
  - passkey の同期（GPM / 1Password など）
  - Restore Credentials
- TODO: 評価軸（案）— ユーザーの操作 / 運ぶもの（トークンか鍵か）/ サーバで失効できるか / サーバが識別できるか / 端末の条件 / Play 要件への適合
- TODO: 表 — 上記の比較
- TODO: 要点 — Restore Credentials は Block Store の上に載ったレイヤーで、「トークンを運ぶ」のではなく「鍵を運び、その場で署名する」。passkey とは違い、ユーザーからは見えず、プロバイダも経由しない

## 実装と試せるものはあるのか

<!-- 要点だけ書き、詳細は記事 ② へ -->

- TODO: 公式サンプル（`android/identity-samples` の Shrine）と参考 RP サーバ（`GoogleChromeLabs/project-sesame`）
- TODO: Android Studio の Backup / Restore App Data で、端末移行を模して試せる
- TODO: 注意 — サンプルはそのままでは restore によるサインインが完了しない / バックアップファイルに秘密鍵が平文で入る / 実ユーザーの移行経路はサイドロードでは試せない
- TODO: 本記事の実測に使った検証環境の要約（詳細は付録）

<!-- 導線 3（メイン）: 意外な事実を 4 つ見せ、② で答えを回収させる。「疑問点の章を読んでから」と添えて、ここで離脱させない -->
:::message
**実装と検証の詳細は姉妹記事にまとめました**
- 公式サンプルのとおりに書いても、restore によるサインインが完了しない理由
- Android Studio の Backup / Restore App Data が裏で何をしているか
- バックアップファイルの中に、秘密鍵が平文で入っていること
- 実機 2 台で端末移行を試して失敗した原因と、実ユーザー経路を検証する方法

導入時に決めることを先に知りたい方は、このまま次の章に進んでください。
:::
https://zenn.dev/task4233/articles/restore-credentials-android-testing

## 実際に導入するときに考えたい疑問点

<!--
Q&A 形式。各回答の先頭に根拠の種類を付ける: 【実測】【公式】【他者】【考察】【未検証】
- 疑問が多いので、テーマごとの節に分け、各節の冒頭で「この節で分かること」を 1 行で示す
- 厚く書くのは Q12・Q14・Q15・Q17・Q22。それ以外は 2〜3 行で答える
- 根拠は ~/work/task4233/restore-credentials/blog-research-notes.md
-->

### 対応の要否と範囲（PM 向け）

#### Q1. 2027 年 4 月の締切に例外はあるのか？

- TODO:【公式】例外は 5 つ — Block Store（2026-09-30 までに本番導入して正常に復元できていること）/ 規制業種（免除される「場合がある」、適用日より前に Play Console から申請）/ 永続的な限定公開アプリ・企業向けデバイス管理アプリ / ゲーム（現時点のみ）/ ログイン機能のないアプリ
- TODO:【公式】「詳細は今後数か月以内に公開」— 例外はまだ確定していない
- TODO:【公式】AEP（Apps Experience Program）の免除 EAB は「高セキュリティ・短命セッション」まで含み、Play 側より広い。両者の関係は不明

#### Q2. Restore Credentials ではなく、通常のログインではダメなのか？

- TODO:【公式】準拠の判定は「復元キーを正常に取得できたこと」。Block Store 以外の方式は準拠とみなされない。手動ログインは要件が解決したい課題として書かれている
- TODO:【公式】一方で「ID コンテキストの復元（"Welcome back, Alex" の表示など）で十分」とも書かれている
- TODO:【考察】この一文が設計の自由度になる — 「誰なのかは復元するが、セッションを張る前に認証を求める」という選択肢がありうる（Q16 につなぐ）

#### Q3. 一部のユーザーだけを対象にすればよいのか？ 全ユーザーが対象なのか？

- TODO:【公式】対象はアプリの認証済みの部分。複数アカウントなら、現在（または最後）のアクティブアカウントの鍵を保存する。mobile / tablet が対象
- TODO:【公式に記述なし】アカウント種別や地域で絞り込めるかは書かれていない

#### Q4. すでに Block Store を使っていれば、何もしなくてよいのか？

- TODO:【公式】2026-09-30 までに本番導入し、ログイン状態を正常に復元できている場合に限る。「本番導入」「正常に復元」の判定方法は書かれていない

#### Q5. 恩恵を受けるのはどのユーザーか？ 再インストールしたユーザーは救われるのか？

- TODO:【実測】効くのは新端末セットアップ時だけ。同じ端末での再インストールでは鍵は戻らず、セットアップ後に移行をやり直す手段もない

#### Q6. 対応できる端末の条件は何か？

- TODO:【公式】Android 9 以上。GMS の最小バージョンはドキュメント（24220000）とコード（242200000）で桁が食い違っている
- TODO:【公式】マルチプロファイル端末では、最初にセットアップしたプロファイルだけ

#### Q7. 導入の効果をどう測るのか？

- TODO:【公式】効果を示す公式の数値は無い（あるのは「米国で年 40% が買い替え」だけ）
- TODO:【考察】restore によるサインインの成功率、`NoCredentialException` の割合、移行後の再ログイン率

#### Q8. 検証には何が必要か？ 工数とリードタイムは？

- TODO:【実測】実ユーザーの移行経路はサイドロードでは再現できない → Play の internal testing が必要

### サーバから restore key はどう見えるか（IdP 向け）

#### Q9. 今の passkey 実装のまま受け付けられるのか？

- TODO:【実測】UV=0、AAGUID はオール 0、attestation は none、origin は `apk-key-hash`。UV 必須・AAGUID での制限・web origin のみの許可リストだと弾かれる
- TODO:【公式】Android 公式 skill も "Standard WebAuthn services typically assume user verification is always required" と書いている

#### Q10. サーバは restore key と passkey を区別できるのか？

- TODO:【実測】できない。種別は登録時のクライアントの申告に頼るしかない（参考実装の `?type=rc`）
- TODO:【他者】supabase-flutter の PR でも "Restore keys are indistinguishable from passkeys on the server" と書かれている

#### Q11. 鍵がクラウドに同期されたかどうかを、サーバは知れるのか？

- TODO:【実測】知れない。クラウド鍵でもローカル鍵でも BE=1 / BS=0。比較対象の 1Password の passkey は BS=1
- TODO:【公式】NIST も「public-facing なら BS で受け入れ条件を決めるべきでない」としており、判断材料にしない前提でよい
- TODO: 導線 — フラグの読み出し手順は記事 ②『実測の方法』

#### Q12. 検証をクライアント側に任せているのは意図的なのか？

- TODO:【公式】公式の指示は「DB で区別して保存せよ」「新しい credential type かメタデータで区別せよ」だけ。サーバで暗号的に区別する方法は示されていない
- TODO:【公式に記述なし】UV=0・AAGUID 0・BS=0 の理由を説明した文書は見つからない（Android Docs / Blog / W3C / FIDO）
- TODO:【考察】意図の推測（ユーザーから不可視・プロバイダ非経由の設計と、WebAuthn のフラグの意味の曖昧さ［w3c/webauthn #1933］）と、推測であることの明示
- TODO:【考察】結論 — 意図の有無にかかわらず、RP は「申告は信じない」前提で設計する

#### Q13. 1Password で動いてしまうのは問題ではないのか？

- TODO:【実測】restore key は Credential Manager のプロバイダの仕組みを経由せず、GMS の Block Store に直結する。1Password を優先プロバイダにしていても、1Password は一度も呼ばれなかった
- TODO:【実測】1Password に保存されたのは通常の passkey で、restore key とは別のレコード
- TODO:【考察】問題になりうる点 — ユーザーが選んだプロバイダの外に鍵が置かれる / 鍵の保管は Google のバックアップに委ねられる / 企業のポリシーでプロバイダを制限していても効かない

### restore 由来のセッションをどう扱うか（IdP 向け・記事の核）

#### Q14. NIST などの MFA 要件と矛盾しないのか？

- TODO:【公式】NIST SP 800-63B Rev.4 — AAL2 は 2 つの異なる認証要素が必須。UV が無い鍵は単一要素の暗号認証器として扱う（SHALL）。sync fabric は AAL2 相当の MFA で保護（SHALL）
- TODO:【公式】SP 800-53 は管理策の文書で、コンシューマの認証強度は 800-63B で論じる（読者の混同に先回りする）
- TODO:【公式】Google は「D2D 移行は所持の強い証明なので、追加の MFA は求めるべきでない」と書いている
- TODO:【考察】両者の食い違いの整理 — Google は「ロックを解除した旧端末を持っていること」を根拠にしていると読めるが、RP はそれを検証できない。NIST の枠組みでは所持は 1 要素
- TODO:【考察】どちらを取るかはサービスのリスク許容度で決める（Q16）

#### Q15. これは「セッションの復旧」なのか、新しい認証なのか？

- TODO:【実測】Cookie を持ち回すのではなく、その場で新しい assertion を作ってセッションを新規に発行している。旧端末のセッションとは無関係
- TODO:【公式】NIST §5.1 — セッションは認証イベントの AAL を超えてはならない → restore 由来のセッションは AAL1
- TODO:【公式】通知は戻らないので FCM トークンの再登録が必要

#### Q16. restore 由来のセッションは、どこまで信頼してよいのか？

- TODO:【考察】サービスの特性ごとの方針 — 低リスク（そのまま通常のセッション）/ 段階的に要件を上げる（AAL1 扱い、機微な操作の前に step-up）/ 高リスク（ID コンテキストの復元にとどめてセッションは張らない、または免除を申請）
- TODO:【他者】先行実装（kkoiwai）の方針 — `amr` を引き継がない、`acr` をダウングレード、原則 `id_token` を発行しない
- TODO: 図 — 保証レベルと、許可する操作の対応表

#### Q17. セキュリティ上のリスクは無いのか？

- TODO: 表 — リスク × 根拠 × 影響
  - 【実測】セッションの期限と失効をすり抜けられる（サインアウト後も、鍵があれば新しいセッションを作れる）
  - 【他者】盗んだ access token で鍵を差し替えられ、パスワードリセット後も居座られる（kilo #1162）
  - 【実測】種別は申告にすぎず、ソフトウェア生成の鍵を rc と偽って登録できる
  - 【公式・推測】Google アカウントの乗っ取りが波及する（復元時に旧端末の画面ロックを求めるかは未確認）
  - 【実測】signCount が常に 0 でクローンを検知できない
  - 【実測】debug ビルドでは秘密鍵を adb で持ち出せる
  - 【実測・他者】孤立した鍵が溜まる
  - 【実測】UI が無いのに UP=1 — UP を「本人がいた証拠」に使えない（⚠️ 記事にする前に裏付けを取り直す）

#### Q18. 不審なログインや乗っ取りに気づいたとき、どう止めるのか？

- TODO:【実測】セッションを失効させても次の起動で再ログインされる
- TODO:【考察】失効手順に公開鍵の削除を組み込む（全端末サインアウト / パスワード変更・リセット / 乗っ取り対応）。鍵の登録時には直近の認証を求める

#### Q19. 使われなくなった鍵はどうなるのか？

- TODO:【実測】アンインストールで端末側の鍵は消えるが、サーバ側の公開鍵は残る
- TODO:【他者】孤立鍵が登録上限を圧迫する例（supabase-flutter）
- TODO:【考察】TTL と、最終使用日時にもとづく掃除。公式 skill の「1 端末 1 鍵」

#### Q20. ユーザーの passkey 管理画面に、restore key をどう見せるのか？

- TODO:【公式】管理画面からは隠す（システムが管理する鍵であるため）
- TODO:【考察】隠すことと「申告を信じるしかない」ことの組み合わせで、偽って登録された鍵にユーザーが気づけなくなる

#### Q21. ログには何を残すべきか？

- TODO:【考察】ログイン記録に restore 由来フラグを持たせる。後から遡って識別できないので、実装より先に決める

### 既存の認証基盤への組み込み（IdP 向け）

#### Q22. Chrome Custom Tabs を呼び出さずにセッションを復旧できるのか？

- TODO:【公式】restore の作成・取得はネイティブ限定。Custom Tabs と既定ブラウザの Cookie にアプリから書き込む手段は無い
- TODO:【公式】WebView には `CookieManager` で Cookie を入れられる
- TODO:【公式】Custom Tabs は `Browser.EXTRA_HEADERS` でヘッダを付けられる（任意のヘッダは DAL の `use_as_origin` 検証が必要）
- TODO:【考察】現実的な方法 — ネイティブでトークンを得て、ワンタイムのログイン URL を Custom Tabs で開き、サーバに `Set-Cookie` を返させる。セッション固定攻撃への配慮（短命・単回・端末バインド）

#### Q23. WebView / Custom Tabs / 通常ブラウザ / 複数プロファイルで振る舞いはどう違うのか？

- TODO: 表 — Cookie jar / WebAuthn の可否 / RP から見える origin / DAL の要否
  - 【公式】WebView はブラウザと状態を共有しない。WebAuthn は `setWebAuthenticationSupport` で有効化、`mediation:"conditional"` 非対応
  - 【公式】Custom Tabs は Chrome と Cookie を共有。Ephemeral Custom Tab は隔離。Auth Tab は callback で結果を受け、https リダイレクトに DAL が必要
  - 【公式】Android の Chrome はプロファイル 1 つのみ。restore key は最初にセットアップしたプロファイルだけ
  - 【未検証】WebView（FOR_APP）で RP に見える origin

#### Q24. ブラウザでセッションを管理する流れとは、どう両立させるのか？

- TODO:【公式】ブラウザに寄せる流れの根拠 — RFC 8252（埋め込み UA ではアプリが Cookie や資格情報を盗める、SSO が効かない）、Google の WebView OAuth 禁止、FedCM（サードパーティ Cookie への依存を減らす、ブラウザが仲介する UI）、Digital Credentials（custom scheme はブラウザから見えず濫用を抑えられない）、DBSC（セッション Cookie の窃取対策）
- TODO:【公式】逆の流れ — OAuth First-Party Apps（first-party 限定でブラウザを使わない）、Restore Credentials（ネイティブ完結）
- TODO:【考察】二層構造として読める（第三者アプリはブラウザ、first-party はネイティブも可）。ただし明言した一次情報は無い
- TODO:【未検証】Custom Tabs 上で Digital Credentials API が動くか（Chrome 141 で既定有効だが、Custom Tabs / WebView への言及なし）

#### Q25. FIDO アサーションで /authorize 相当のクレデンシャルをブートストラップするのに、FiPA は使えるのか？

- TODO:【公式】draft-ietf-oauth-first-party-apps-04 — Authorization Challenge Endpoint に "signed passkey challenge" を送って認可コードを得る流れを想定している。`auth_session` はデバイスに束縛すべき、`insufficient_authorization` で step-up、`redirect_to_web` でブラウザに切り替え、first-party であることの検証が必須
- TODO:【公式】ただし付録の passkey の例は生体認証か PIN（UV）が前提。UV=0 の assertion の扱いは AS のポリシーで決める
- TODO:【考察】restore の assertion → 認可コード（低い acr）→ 必要なら `insufficient_authorization` で step-up、という組み込み方
- TODO:【他者】独自エンドポイントでトークンを直接発行する先行実装（kkoiwai）は、この形に近い
- TODO:【公式】OpenID Native SSO は同じ端末内のアプリ間共有が目的で、端末移行とは別

### 実装・運用（Android 向け、要点のみ）

#### Q26. 鍵の登録オプションで不明瞭な点は無いのか？

- TODO:【公式】`requestJson` は「PublicKeyCredentialCreationOptionsJSON」とだけ書かれ、どのフィールドが効くか・無視されるかは記述が無い。明記は取得側の `userVerification` を `discouraged` に上書きすることだけ
- TODO:【公式】androidx が検査するのは `user.id` の有無だけ。仕様違反は `CreateRestoreCredentialDomException(DataError)` 1 種類にまとめられる
- TODO: 表 — 不明瞭な点（UV / residentKey / attachment / alg / attestation / excludeCredentials / 既存の鍵がある状態での再作成 / ローカル鍵が D2D で移るか / 特定の鍵だけ消せない）
- TODO: → 記事 ②『鍵を作るタイミング』

#### Q27. 鍵はいつ作り、いつ消すのか？

- TODO:【公式】サインイン直後と、サインイン済みなら起動時にも作る。サインアウト時と 401 を受けたときに消す
- TODO: → 記事 ②『鍵を作るタイミング』『鍵を消すタイミング』

#### Q28. E2EE が使えない端末ではどうなるのか？

- TODO:【公式・実測】`E2eeUnavailableException` → `isCloudBackupEnabled = false` で再試行。ローカル鍵はクラウド復元では使えない
- TODO: → 記事 ②『`E2eeUnavailableException` のフォールバック』

#### Q29. すでに Auto Backup を使っている場合、どう関係するのか？

- TODO:【実測】バックアップに認証状態が含まれていると、restore key を使わずにサインインしてしまう。効果の測定もできなくなる
- TODO: → 記事 ②『事前確認：バックアップ対象に認証状態が含まれていないか』

#### Q30. テストするときに気をつけることは何か？

- TODO:【実測】バックアップファイルに秘密鍵が平文で入る / debuggable ビルドなら adb だけで鍵を持ち出せる / サンプルの不具合
- TODO: → 記事 ②『検証の方法』『検証の限界』

## 既存の議論と先行事例

<!-- 「他に誰か検討していないの？」に答える。実測の再現性の裏付けにもなる -->

- TODO: 表 — 出典 × 日付 × 論点（区別 / UV / 失効 / 組み込み）
  - kinmemodoki 氏の Zenn scrap（実測値が本記事と一致）
  - kkoiwai/RestoreCredentialsAndroid（`credential_type`、acr のダウングレード、独自エンドポイント。README の「クラウド同期時 BS=true」は本記事の実測と食い違う）
  - supabase-flutter PR #1824、corbado/flutter-passkeys PR #305
  - bpronin90/kilo #1162（鍵の差し替えによる居座り）
  - w3c/webauthn #1933（BE の意味の曖昧さ）
- TODO: 見つからなかったもの — W3C WebAuthn・Stack Overflow・カンファレンス講演での議論は 0 件（2026-09 時点）

## まとめ

- TODO: Restore Credentials はアカウントにログイン経路を 1 本増やすものであり、その強度はプラットフォームに委ねられている
- TODO: 同じコードで扱えるが、同じポリシーで扱ってはいけない
- TODO: 次の行動 — 誰と何を合意するか（PM: 対象範囲と検証計画 / IdP: 保証レベル・種別・失効 / Android: 鍵の作成・削除のタイミング）
<!-- 導線 5: 読み終えた Android エンジニア・検証担当を ② へ送る -->
- TODO: 一文の導線 — 「Android 側で実装と検証に進む方は、姉妹記事へどうぞ」＋リンクカード

## 未検証事項

- TODO: Play 配信アプリでの実ユーザー経路
- TODO: クラウド経路の強度の詳細（E2EE の鍵が何に依存するか）
- TODO: restore credential 単体で assetlinks.json が必須か
- TODO: Play 要件の免除条項と AEP の関係
- TODO: バックアップのタイミングと、クラウド経路の失敗条件

## 付録：検証環境

- TODO: 端末・OS・GMS・ライブラリのバージョン、サンプルのコミット、RP サーバ

## 参考

- TODO: 公式ドキュメント / サンプル / 参考実装 / 二次情報
- TODO: 関連する考え方として kokukuma 氏の記事（Passkey Fallback Strategies、認証と身元確認の強度、RP 側の考慮事項）
