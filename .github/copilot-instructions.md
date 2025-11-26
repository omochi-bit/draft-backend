# copilot-instructions.md

## ロール定義
Spring Boot REST API

## ツール

---

## Lombokの実装方針

### 目的
- 初学者の学習を阻害しない最小限のLombok使用で、冗長性を減らす。
- JPAの制約を尊重し、副作用を避ける。

### 背景と理由
- `@Data`は危険：`equals`/`hashCode`や`toString`がJPAエンティティで予期せぬ副作用を起こすため。
- フィールドインジェクション禁止：テスト容易性・保守性の観点から、コンストラクタインジェクションを採用。
- エンティティの不変性重視：`@Builder`や`@Setter`を付けると、JPAのライフサイクルやドメインモデルの意図が崩れる。

### 全体ルール
- 使用するアノテーションは原則：  
  `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor`, `@Builder`,  
  `@RequiredArgsConstructor`, `@Slf4j`（必要に応じて`@Value`）。
- `@Data`は使用しない。
- フィールドインジェクション禁止。Service/Controllerは`@RequiredArgsConstructor`でコンストラクタインジェクション。

### エンティティ
- `@Getter`＋`@NoArgsConstructor(access = PROTECTED)`を基本とする。
- `@Builder`は付けない理由：  
  - JPAのプロキシ生成と競合する可能性  
  - 不変性が崩れ、意図しない状態変更が発生する恐れ
- 状態変更は明示的メソッドで提供し、公開`@Setter`は付けない。
- 双方向関連がある場合は、`@ToString(exclude = ...)`や`@EqualsAndHashCode(exclude = ...)`で循環参照を避ける。
- `equals`/`hashCode`はIDベースで必要時のみ。推奨：  
  ```java
  @EqualsAndHashCode(onlyExplicitlyIncluded = true)
  public class Todo {
      @EqualsAndHashCode.Include
      private Long id;
  }
  ```

### DTO
- 入力用（Request）：  
  `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`  
  → 柔軟性重視、`@Setter`は入力専用であることを明記。
- 出力用（Response）：  
  - Java Record（推奨、Java 21+）  
  - または`@Value`＋`@Builder`で不変にする。
- DTOに限り`@Builder`を積極的に使用してよい。

### Service / Controller
- `@RequiredArgsConstructor`でDI。
- `@Slf4j`でロギング（Service層＋Controller層）。
- LombokはDIとログ用に限定し、ビジネスロジックの可読性を保つ。

### 拡張（必要になったら採用）
- 継承があるDTOでは`@SuperBuilder`。
- 値の部分差し替えが増えたら`@With`。
- 初期段階では採用しない（シンプルさ優先）。

### IDEと設定
- Lombokプラグインを有効化（VSCode（推奨拡張機能: Lombok Annotations Support, Java Extension Pack））。
- VSCodeでは`settings.json`に以下を追加:
```json
{
  "java.compile.nullAnalysis.mode": "automatic",
  "java.configuration.enableAnnotations": true
}
```
- Java 21以上ならRecordで出力DTOを優先。

### 例外ルール
- テストやFixture生成で一時的に`@Builder`を使う場合はDTOに限定。エンティティには付けない。

#### サンプルコード

##### エンティティ
```java
@Entity
@Table(name = "todos")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Todo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @EqualsAndHashCode.Include
    private Long id;

    private String title;
    private boolean completed;

    public static Todo create(String title) {
        Todo t = new Todo();
        t.title = title;
        t.completed = false;
        return t;
    }

    public void markCompleted() {
        this.completed = true;
    }
}
```

##### DTO（入力用）
```java
@Getter
@Setter // 入力専用
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class TodoRequest {
    private String title;
    private boolean completed;
}
```

##### DTO（出力用）
```java
public record TodoResponse(Long id, String title, boolean completed) {}
```

##### Service
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TodoService {
    private final TodoRepository todoRepository;

    public TodoResponse create(TodoRequest req) {
        log.info("Creating todo: {}", req.getTitle());
        Todo todo = Todo.create(req.getTitle());
        Todo saved = todoRepository.save(todo);
        return new TodoResponse(saved.getId(), saved.getTitle(), saved.isCompleted());
    }
}
```

##### Controller（追加例）
```java
@RestController
@RequestMapping("/todos")
@RequiredArgsConstructor
@Slf4j
public class TodoController {
    private final TodoService todoService;

    @PostMapping
    public TodoResponse create(@RequestBody TodoRequest request) {
        log.info("Received request to create todo");
        return todoService.create(request);
    }
}
```

---

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
