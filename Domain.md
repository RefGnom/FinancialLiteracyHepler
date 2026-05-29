erDiagram
    Transaction {
        guid Id PK
        datetime Date
        decimal Amount
        string Description
        string Comment
        guid AccountId
        guid CategoryId FK
        guid ScopeId FK
        bool IsDeleted
        datetime CreatedAt
        datetime UpdatedAt
        bool IsForeignCurrency
        json Metadata
    }
    Position {
        guid Id PK
        guid TransactionId FK
        string Name
        decimal Amount
        guid CategoryId FK
        string Comment
        bool IsDeleted
    }
    Tag {
        guid Id PK
        string Name
    }
    Category {
        guid Id PK
        string Name
        guid ParentId FK
        string Description
    }
    Scope {
        guid Id PK
        string Name
        string Description
    }
    Compensation {
        guid Id PK
        guid FromTransactionId FK
        guid ToTransactionId FK
        decimal Amount
    }
    InternalTransfer {
        guid Id PK
        guid FromTransactionId FK
        guid ToTransactionId FK
    }
    TransactionTag ||--o{ Tag : "m:n"
    PositionTag ||--o{ Tag : "m:n"