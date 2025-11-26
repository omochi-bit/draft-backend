# copilot-instructions.md

## ロール定義

## ツール

## メール送信機能の実装方針
- AWS SESのHTTP APIを利用してメール送信を実装する
- Spring BootのWebClientをRESTクライアントとして使用する
- 認証にはIAMユーザーを使用し、AWS Signature Version 4で署名する
- 認証情報（AWSアクセスキー、シークレットキー）はAWS Secrets Managerで管理し、平文での記載を避ける
- 送信形式はプレーンテキストメールを基本とし、HTMLメールやテンプレートは利用しない
- 非同期送信は@Asyncで実装し、レスポンス性能を向上させる
- メール送信失敗時は1回限りのリトライを実施する
- タイムアウト設定（接続・読み書き）を行う
- 将来的な拡張（HTMLメール、テンプレート対応）は別途検討
