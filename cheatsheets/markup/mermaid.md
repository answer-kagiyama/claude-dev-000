# Mermaid チートシート

## 基本

```markdown
```mermaid
graph TD
    A[Start] --> B{Decision}
    B -->|Yes| C[OK]
    B -->|No| D[Cancel]
\```
```

## フローチャート

### 基本構文

````markdown
```mermaid
flowchart TD
    A[Square] --> B(Round)
    B --> C{Diamond}
    C -->|One| D[Result 1]
    C -->|Two| E[Result 2]
```
````

### ノードの形状

```mermaid
flowchart LR
    A[Rectangle]
    B(Round edges)
    C([Stadium])
    D[[Subroutine]]
    E[(Database)]
    F((Circle))
    G>Asymmetric]
    H{Diamond}
    I{{Hexagon}}
    J[/Parallelogram/]
    K[\Parallelogram\]
    L[/Trapezoid\]
    M[\Trapezoid/]
```

### 方向

```mermaid
flowchart TB  %% Top to Bottom
flowchart TD  %% Top Down（TBと同じ）
flowchart BT  %% Bottom to Top
flowchart RL  %% Right to Left
flowchart LR  %% Left to Right
```

### 矢印の種類

```mermaid
flowchart LR
    A --> B    %%  実線矢印
    C -.-> D   %%  点線矢印
    E ==> F    %%  太線矢印
    G -- text --> H    %%  テキスト付き
    I -.text.-> J      %%  点線でテキスト付き
    K ==text==> L      %%  太線でテキスト付き
    M --> N & O        %%  複数の接続先
    P & Q --> R        %%  複数の接続元
```

### サブグラフ

```mermaid
flowchart TB
    subgraph one[Subnet 1]
        A1 --> A2
    end
    subgraph two[Subnet 2]
        B1 --> B2
    end
    one --> two
```

## シーケンス図

### 基本構文

````markdown
```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Alice->>Bob: Hello Bob, how are you?
    Bob-->>Alice: Great!
    Alice-)Bob: See you later!
```
````

### メッセージタイプ

```mermaid
sequenceDiagram
    A->>B: 実線矢印
    A-->>B: 点線矢印
    A-)B: 実線矢印（矢尻なし）
    A--)B: 点線矢印（矢尻なし）
    A-xB: 末尾にX
    A--xB: 点線で末尾にX
```

### アクティベーション

```mermaid
sequenceDiagram
    Alice->>+John: Hello John, how are you?
    John-->>-Alice: Great!

    Alice->>+John: Another message
    activate John
    John-->>Alice: Busy
    deactivate John
    John-->>-Alice: Done
```

### ノート

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Note left of Alice: Alice thinks
    Note right of Bob: Bob thinks
    Note over Alice,Bob: Both think
```

### ループ・条件分岐

```mermaid
sequenceDiagram
    Alice->>Bob: Hello

    alt is sick
        Bob->>Alice: Not so good
    else is well
        Bob->>Alice: Feeling fresh
    end

    opt Extra response
        Bob->>Alice: Thanks for asking
    end

    loop Every minute
        Bob->>Alice: Heartbeat
    end

    par Parallel 1
        Alice->>Bob: Message 1
    and Parallel 2
        Alice->>Charlie: Message 2
    end
```

## クラス図

### 基本構文

````markdown
```mermaid
classDiagram
    class Animal{
        +String name
        +int age
        +makeSound()
    }
    class Dog{
        +String breed
        +bark()
    }
    Animal <|-- Dog
```
````

### 関係性

```mermaid
classDiagram
    classA <|-- classB : Inheritance
    classC *-- classD : Composition
    classE o-- classF : Aggregation
    classG <-- classH : Association
    classI -- classJ : Link
    classK <.. classL : Dependency
    classM <|.. classN : Realization
    classO .. classP : Link (Dashed)
```

### 可視性

```mermaid
classDiagram
    class MyClass{
        +public
        -private
        #protected
        ~package
        +method()
    }
```

### メソッド・プロパティ

```mermaid
classDiagram
    class BankAccount{
        +String owner
        +BigDecimal balance
        +deposit(amount)
        +withdraw(amount)
    }
    class SavingsAccount{
        +BigDecimal interestRate
        +addInterest()
    }
    BankAccount <|-- SavingsAccount
```

## 状態図

### 基本構文

````markdown
```mermaid
stateDiagram-v2
    [*] --> Still
    Still --> [*]
    Still --> Moving
    Moving --> Still
    Moving --> Crash
    Crash --> [*]
```
````

### 複合状態

```mermaid
stateDiagram-v2
    [*] --> First
    First --> Second
    First --> Third

    state First {
        [*] --> fir
        fir --> [*]
    }
```

### 選択

```mermaid
stateDiagram-v2
    state if_state <<choice>>
    [*] --> IsPositive
    IsPositive --> if_state
    if_state --> False: if n < 0
    if_state --> True : if n >= 0
```

## ER図（Entity Relationship）

````markdown
```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--|{ LINE-ITEM : contains
    CUSTOMER }|..|{ DELIVERY-ADDRESS : uses

    CUSTOMER {
        string name
        string email
        string phone
    }
    ORDER {
        int orderNumber
        date orderDate
        string status
    }
```
````

### 関係性

```
||--||  : One to one
||--o{  : One to many
}o--o{  : Many to many
||--|{  : One to one or many
}|..|{  : Many to many (dotted)
```

## ガントチャート

````markdown
```mermaid
gantt
    title Project Schedule
    dateFormat YYYY-MM-DD
    section Design
    Requirements    :a1, 2024-01-01, 30d
    UI Design       :a2, after a1, 20d
    section Development
    Backend         :b1, 2024-02-01, 45d
    Frontend        :b2, after a2, 40d
    section Testing
    QA Testing      :c1, after b1, 15d
    UAT             :c2, after c1, 10d
```
````

### タスクのステータス

```mermaid
gantt
    title Task Status
    dateFormat YYYY-MM-DD
    section Tasks
    Completed task      :done, task1, 2024-01-01, 2024-01-15
    Active task         :active, task2, 2024-01-10, 30d
    Future task         :task3, 2024-02-01, 20d
    Critical task       :crit, task4, 2024-01-20, 25d
```

## パイチャート

````markdown
```mermaid
pie title Pets
    "Dogs" : 45
    "Cats" : 30
    "Birds" : 15
    "Fish" : 10
```
````

## ユーザージャーニー

````markdown
```mermaid
journey
    title My working day
    section Go to work
      Make tea: 5: Me
      Go upstairs: 3: Me
      Do work: 1: Me, Cat
    section Go home
      Go downstairs: 5: Me
      Sit down: 5: Me
```
````

## Gitグラフ

````markdown
```mermaid
gitGraph
    commit
    commit
    branch develop
    checkout develop
    commit
    commit
    checkout main
    merge develop
    commit
    commit
```
````

### カスタマイズ

```mermaid
gitGraph
    commit id: "Initial"
    commit id: "Add feature" tag: "v1.0"
    branch develop
    checkout develop
    commit id: "Dev work"
    checkout main
    merge develop
    commit id: "Hotfix" type: REVERSE
```

## マインドマップ

````markdown
```mermaid
mindmap
  root((Project))
    Planning
      Requirements
      Design
    Development
      Backend
      Frontend
      Database
    Testing
      Unit Tests
      Integration Tests
      E2E Tests
    Deployment
      CI/CD
      Monitoring
```
````

## タイムライン

````markdown
```mermaid
timeline
    title History of Social Media
    2002 : LinkedIn
    2004 : Facebook
         : Google
    2005 : Youtube
    2006 : Twitter
    2010 : Instagram
```
````

## スタイリング

### クラススタイル

```mermaid
flowchart LR
    A:::someclass --> B
    B --> C:::someclass
    classDef someclass fill:#f96,stroke:#333,stroke-width:4px
```

### ノードスタイル

```mermaid
flowchart LR
    id1(Start)-->id2(Stop)
    style id1 fill:#f9f,stroke:#333,stroke-width:4px
    style id2 fill:#bbf,stroke:#f66,stroke-width:2px,stroke-dasharray: 5 5
```

### テーマ

````markdown
```mermaid
%%{init: {'theme':'dark'}}%%
graph TD
    A-->B
```

```mermaid
%%{init: {'theme':'forest'}}%%
graph TD
    A-->B
```

```mermaid
%%{init: {'theme':'neutral'}}%%
graph TD
    A-->B
```
````

## 実践例

### システムアーキテクチャ

````markdown
```mermaid
flowchart TB
    subgraph Client
        Web[Web Browser]
        Mobile[Mobile App]
    end

    subgraph AWS
        subgraph VPC
            ALB[Application Load Balancer]
            subgraph Private
                ECS[ECS Fargate]
                RDS[(RDS PostgreSQL)]
            end
        end
        S3[(S3 Bucket)]
        CloudFront[CloudFront CDN]
    end

    Web --> CloudFront
    Mobile --> CloudFront
    CloudFront --> S3
    CloudFront --> ALB
    ALB --> ECS
    ECS --> RDS
    ECS --> S3
```
````

### APIシーケンス

````markdown
```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Auth
    participant DB

    Client->>+API: POST /login
    API->>+Auth: Validate credentials
    Auth->>+DB: Query user
    DB-->>-Auth: User data
    Auth-->>-API: JWT token
    API-->>-Client: 200 OK + token

    Client->>+API: GET /users (with token)
    API->>+Auth: Verify token
    Auth-->>-API: Valid
    API->>+DB: Fetch users
    DB-->>-API: User list
    API-->>-Client: 200 OK + data
```
````

### データベース設計

````markdown
```mermaid
erDiagram
    USER ||--o{ ORDER : places
    USER {
        int id PK
        string email UK
        string name
        datetime created_at
    }
    ORDER ||--|{ ORDER_ITEM : contains
    ORDER {
        int id PK
        int user_id FK
        decimal total
        string status
        datetime created_at
    }
    PRODUCT ||--o{ ORDER_ITEM : "ordered in"
    ORDER_ITEM {
        int id PK
        int order_id FK
        int product_id FK
        int quantity
        decimal price
    }
    PRODUCT {
        int id PK
        string name
        string description
        decimal price
        int stock
    }
```
````

### CI/CDパイプライン

````markdown
```mermaid
flowchart LR
    A[Git Push] --> B{Run Tests}
    B -->|Pass| C[Build Docker Image]
    B -->|Fail| Z[Notify Failure]
    C --> D[Push to Registry]
    D --> E{Environment}
    E -->|Dev| F[Deploy to Dev]
    E -->|Staging| G[Deploy to Staging]
    E -->|Prod| H[Deploy to Production]
    F --> I[Run E2E Tests]
    G --> I
    H --> I
    I -->|Success| J[Complete]
    I -->|Failure| K[Rollback]
```
````

## Tips

```markdown
# コメント
%% これはコメントです

# 改行
<br>を使用

# リンク
click nodeId "https://example.com" "Tooltip"

# アイコン（Font Awesome）
flowchart TD
    A[fa:fa-user User]

# 設定
%%{init: {
  'theme': 'dark',
  'themeVariables': {
    'primaryColor': '#BB2528',
    'primaryTextColor': '#fff'
  }
}}%%
```

## プラットフォーム対応

- GitHub: ネイティブサポート
- GitLab: ネイティブサポート
- Notion: サポート
- VSCode: プラグイン必要
- Obsidian: プラグイン必要

## オンラインエディタ

- [Mermaid Live Editor](https://mermaid.live/)
- [Mermaid Chart](https://www.mermaidchart.com/)
