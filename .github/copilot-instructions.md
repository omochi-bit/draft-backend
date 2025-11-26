# copilot-instructions.md

## ロール定義
Spring Boot REST API

## ツール

## メール送信機能の実装方針

1. AWS SESのHTTP API（`SendEmail`エンドポイント）を利用してメール送信を実装する。
   - エンドポイント例: `https://email.<region>.amazonaws.com/v2/email/outbound-emails`
   - HTTPメソッド: POST
   - リクエストBodyはJSON形式で、`Destination`, `Content`, `Subject`, `Body`を含む。

2. Spring BootでRESTクライアントとして`WebClient`を使用する。
   - 接続タイムアウト: 5秒
   - 読み取りタイムアウト: 10秒

3. 認証にはIAMユーザーを使用し、AWS Signature Version 4で署名する。
   - `Authorization`ヘッダーにSigV4署名を付与。
   - `X-Amz-Date`と`Host`ヘッダーを必須設定。

4. 認証情報（AWSアクセスキー、シークレットキー）はAWS Secrets Managerで安全に管理し、平文での直書きを避ける。

5. 送信形式はプレーンテキストメールを基本とする（HTMLメールやテンプレートは利用しない）。

6. 非同期送信は`@Async`で実装し、レスポンス性能を向上させる。

7. 簡易リトライを実装する。
   - 最大3回再試行。
   - 指数バックオフ（例: 1秒 → 2秒 → 4秒）。

8. エラーハンドリング方針。
   - 成功時はINFOログ。
   - 失敗時はWARNログ＋例外スロー。

9. 将来的な拡張（HTMLメール、テンプレート対応）は別途検討。
