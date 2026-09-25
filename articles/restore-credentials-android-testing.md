---
title: "Android の Restore Credentials を実装・検証するときの落とし穴"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [android, kotlin, credentialmanager, passkey, testing]
published: false
---

<!--
下書き（フローのみ）。記事 ② / 2 本構成の実務編。
- 軸: 公式サンプルどおりに作っても動かず、エミュレータで確認できても実ユーザー経路は保証されない
- 構成: 誤解 5「サンプルどおりに作れば動く」、6「エミュレータで確認できれば十分」を崩す
- メイン読者: Android エンジニア。サブ: PM（検証計画・工数）
- サーバ側の設計判断は記事 ①（restore-credentials-idp-design）に寄せ、ここでは要約とリンクのみ
- 社内情報（アプリ名・社内設計・社内ログ）は書かない
- adb で鍵を抜く手順は全コマンドを載せず、事実と含意にとどめる
- 記事 ① からの着地点になる節: 型の分岐 / E2EE のフォールバック / 鍵を作るタイミング / 鍵を消すタイミング / 事前確認 / 検証の方法 / 検証の限界 / 実測の方法
  ① の導線に書いた節名とここの見出しを一致させる（見出しを変えたら ① も直す）
- 記事 ① への戻り導線: はじめに / 前提の要約 / 実測の方法 / まとめ
-->

## TL;DR

- TODO: 公式サンプル（Shrine）は、そのままでは restore key でのサインインが完了しない
- TODO: 実装で押さえる点 — 起動時の鍵作成 / E2EE のフォールバック / `NoCredentialException` は正常系 / 型の分岐
- TODO: Android Studio の Backup / Restore App Data で確かめられるのは「鍵が戻ればサインインできるか」まで
- TODO: 実ユーザーの端末移行はサイドロードでは再現できない。Play の internal testing を検証計画に含める
- TODO: バックアップファイルには秘密鍵が平文で入る。取り扱いに注意

## はじめに

- TODO: 背景 — 2027 年 4 月の Play 要件（詳細は記事 ①）
- TODO: 本記事の範囲 — クライアント実装と検証方法。サーバ側の設計判断は記事 ① へ
- TODO: 記事 ① のリンクカード
https://zenn.dev/task4233/articles/restore-credentials-idp-design
- TODO: 検証環境の要約（詳細は付録へ）

## 前提の要約

- TODO: 何が起きるか（3 行程度）と、効く場面は新端末セットアップ時だけであること
- TODO: サーバから restore key は passkey と区別できない → 詳しくは記事 ① の Q10
- TODO: 記事 ① を読んでいない読者向けに、3 行の要約（何か / いつ効くか / サーバから何が見えるか）

## 実装の落とし穴

<!-- 誤解 5「サンプルどおりに作れば動く」を崩す -->

### `RestoreCredential` は `PublicKeyCredential` のサブクラスではない

- TODO: クラス階層（javap の結果）
- TODO: サンプルのコード（`is PublicKeyCredential` の内側で型を判定しているため到達不能）と修正例
- TODO: 症状 — 取得には成功するが、無言で `AuthResult.Failure` に落ちる

### `E2eeUnavailableException` のフォールバック

- TODO: ドキュメントの指示（`isCloudBackupEnabled = false` で再試行）と、事前に判定する API が androidx 側に無いこと
- TODO: クラウド鍵とローカル鍵の違い（クラウド復元で使えるかどうか）

### 鍵を作るタイミング

- TODO: サインイン成功時だけでなく、サインイン済みなら起動時にも作る（クラウドへのバックアップのタイミングは制御できない）
- TODO: 認証方式に依存させない（サンプルはパスワード経路でしか作らない）
- TODO: 画面ロックの有無で作成を諦めない（ローカル保存なら不要）

### 取得に失敗したとき

- TODO: `NoCredentialException` は「まだ無い / 移行していない」の正常系。ユーザーに見せない
- TODO: 通常のログイン画面へのフォールバック

### 鍵を消すタイミング

- TODO: サインアウト時、サーバから 401 を受けたとき
- TODO: サンプルでは 401 による自動サインアウトで鍵が消えない

## 検証の方法

### 事前確認：バックアップ対象に認証状態が含まれていないか

- TODO: セッション Cookie がバックアップに含まれると、restore key を使わずにサインインしてしまい効果を測れない
- TODO: 交絡の切り分け方（サーバ側でセッションを無効化してから起動する）

### Android Studio の Backup / Restore App Data

- TODO: 手順の要点
- TODO: 仕組み — 端末からホストに ZIP で吸い出し、復元時に押し戻す（`application.backup` の構成）
- TODO: 実機でも使える（前面のアプリが対象）
- TODO: 詰まった点（adb の常駐接続との競合 など）

### ⚠️ バックアップファイルには秘密鍵が平文で入る

- TODO: `auth_backup` の中身と、秘密鍵であることの確認方法（値は伏せる）
- TODO: `.gitignore` に `*.backup` を入れる。共有・コミットしない
- TODO: debuggable ビルドなら adb だけで鍵を持ち出せる → 検証用ビルドの配布時の取り扱い

## 実測の方法

<!-- 記事 ① の Q10・Q11 からの着地点。IdP エンジニアが自分の環境で再現できるようにする -->

- TODO: 登録レスポンス（attestationObject）から authenticatorData を取り出し、AAGUID と UP / UV / BE / BS、signCount を読む手順
- TODO: clientDataJSON の origin（`android:apk-key-hash:`）を署名証明書の指紋と照合する手順
- TODO: 参考 RP サーバ側の記録（`credentialBackedUp` など）との対応
- TODO: → この結果が設計にどう効くかは記事 ① の Q9〜Q11

## 検証の限界

<!-- 誤解 6「エミュレータで確認できれば十分」を崩す -->

- TODO: Studio で確かめられること / 確かめられないことの表
- TODO: 実機 2 台での検証結果 — クラウド経路（transport が拒否）、ケーブル移行（セットアップ中にアプリが入らず鍵が届かない）
- TODO: 原因 — 端末移行は Play から再インストールする仕組みで、鍵はそのタイミングでしか配送されない
- TODO: セットアップ後に移行をやり直す手段は無い

## 検証計画への落とし込み

<!-- PM 向け -->

- TODO: Play の internal testing での配信と、そのリードタイム
- TODO: 確認項目のチェックリスト

## まとめ

- TODO: 実装の要点と検証の限界の再掲
- TODO: サーバ側で決めること（保証レベル・種別の管理・失効）は記事 ① へ＋リンクカード
https://zenn.dev/task4233/articles/restore-credentials-idp-design

## 付録：検証環境

- TODO: 端末・OS・GMS・ライブラリのバージョン、サンプルのコミット、RP サーバ

## 参考

- TODO: 公式ドキュメント / サンプル / 二次情報
