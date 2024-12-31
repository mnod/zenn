---
title: "AWS IAM Identity Center で SAML IdP を試す"
emoji: "🍉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["apache", "sso", "saml", "aws"]
published: true
---

# 前提

AWS IAM Identity Center を SAML IdP として、mod_auth_mellon を SP として設定してみる。

# Apache mod_auth_mellon で SAML SP の設定

https://zenn.dev/mnod/articles/4faa7d4693765f と同様
IdP_metadata.xml は要書き換え(後述)

# AWS IAM Identity Center の設定

独自アプリのIdPとして利用する場合、IAM Identity Center の前提として Organizations が必要。(Organizations なしでも IAM Identity Center を有効化できるが、独自アプリのIdPとして利用できない)

- 一つのリージョンで有効化したら、他のリージョンでは有効化できないらしい。
- rootユーザではなくて、IAMユーザで有効化可能。

## IAM Identity Center 有効化

1. AWSのマネジメントコンソールにログインして `IAM Identity Center` に移動
1. `Enable` を押下
1. `Enable with AWS Organizations` を選択して `Continue` を押下

## ユーザー作成

1. 左ペインで `Users` へ移動 → 右ペインで `Add user` を押下
1. Username、Email address、First name、Last name を入力。
Password は `Generate a one-time password that you can share with this user` を選択
1. Group は特に指定せずに `Add user` を押下
1. One-time password が表示されるので、パスワードを控えて `Close` を押下

## 確認メール送信

1. 左ペインで `Users` へ移動 → 右ペインで作成したユーザーを選択
1. 画面上部の `Send email verification link` を押下
1. 確認ダイアログが表示されるので、`Send` を押下

メールが送信されるので、`Verify your email address` のリンクをクリックする

## IdP 設定

1. 左ペインで `Application assignments` > `Applications` へ移動 → 右ペインで `Add application` を押下
1. Setup preference で `I have an application I want to set up` を選択 → Application type で `SAML 2.0` を選択して `Next` を押下
1. Configure application で `Display name`、`Description` を適宜入力
1. IAM Identity Center metadata で `IAM Identity Center SAML metadata file` をダウンロード
SP側に /etc/apache2/mellon/IdP_metadata.xml として保存して `apachectl graceful` を実行
1. Application metadata で `Upload application SAML metadata file` を選択 → `Choose file` より SP メタデータ(mellon_metadata.xml)をアップロード
1. 画面最下部の `Submit` を押下

## ユーザ割り当て

1. 左ペインで `Application assignments` > `Applications` へ移動 → 右ペインで `Customer managed` タブへ移動 → 作成したアプリケーションを選択
1. `Assigned users and groups` の箇所で `Assign users and groups` を押下
1. 作成したユーザを検索して選択 → `Assign` を押下

## 属性マッピングの追加

1. 左ペインで `Application assignments` > `Applications` へ移動 → 右ペインで `Customer managed` タブへ移動 → 作成したアプリケーションを選択
1. `Actions` のプルダウンメニューで `Edit attribute mappings` を選択
1. 以下の情報を追加して `Save changes` を押下

| 設定名 | 設定値 |
| --- | --- |
| User attribute in the application | Subject |
| Maps to this string value or user attribute in IAM Identity Center | ${user:subject} |
| Format | transient |

## MFA 無効化

1. 動作確認のため一時的にMfAを無効化します。
1. 左ペインで `Settings` へ移動 → 右ペインで `Authentication` タブへ移動
1. `Multi-factor authentication` の箇所で、`Configure` を押下
1. Prompt users for MFA で `Never (disabled)` を選択
1. 画面最下部の `Save Changes` を押下

動作確認が終わったら `Only when their sign-in context changes (context-aware)` に戻す

# 動作確認

WebサーバのURLにアクセスして、AWS のログイン画面に遷移すること、作成したユーザー名でログインできること、コンテンツを表示できることを確認する。

