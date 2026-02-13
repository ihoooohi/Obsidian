
## Definition

```mermaid
%%{init: {"flowchart": {"htmlLabels": false, "nodeSpacing": 60, "rankSpacing": 80}}}%%
flowchart TB
  R["Relation: Student"]

  subgraph SCHEMA["Schema (Structure - No Data)"]
    A["Attributes: ID, Name, Age, GPA"]
    D["Domains: ID=int; Name=string; Age=0-150; GPA=0.0-4.0"]
    S["Schema Form: Student(ID:int, Name:string, Age:int, GPA:float)"]
  end

  subgraph STATE["State / Instance (Current Data)"]
    T1["Tuple: (1, Alice, 20, 3.8)"]
    T2["Tuple: (2, Bob, 21, 3.5)"]
  end

  R --> SCHEMA
  R --> STATE
```

```mermaid
%%{init: {"flowchart": {"htmlLabels": false, "nodeSpacing": 60, "rankSpacing": 80}}}%%
flowchart TB

    R["Relation Schema: R(A1, A2, ..., An)"]
    DEG["Degree = n (number of attributes)"]

    DOM["Domains:
    dom(A1), dom(A2), ..., dom(An)"]

    CART["Cartesian Product:
    dom(A1) × dom(A2) × ... × dom(An)"]

    STATE["Relation State:
    r(R) ⊆ dom(A1) × ... × dom(An)"]

    TUP["Tuple:
    t = <v1, v2, ..., vn>"]

    ACCESS["Attribute Access:
    t[Ai] = vi"]

    R --> DEG
    R --> DOM
    DOM --> CART
    CART --> STATE
    STATE --> TUP
    TUP --> ACCESS
```

1. 定义 Schema
2. 每个属性有 Domain
3. Domain 形成 Cartesian Product
4. State 是它的子集
5. State 由 tuples 构成
6. tuple 可以用 t[Ai] 访问

## Relational Constraints

```mermaid
%%{init: {"flowchart": {"htmlLabels": false, "nodeSpacing": 60, "rankSpacing": 80}}}%%
flowchart TB

    RC["Relational Constraints"]

    DC["Domain Constraints
    Value must be in dom(A)
    Atomic values only"]

    KC["Key Constraints
    Tuples must be unique
    Superkey & Key"]

    NC["NULL Constraints
    Some attributes may allow NULL
    Primary key cannot be NULL"]

    EI["Entity Integrity
    Primary key is NOT NULL"]

    RI["Referential Integrity
    Foreign key must reference
    existing primary key"]

    RC --> DC
    RC --> KC
    RC --> NC
    RC --> EI
    RC --> RI
```

## Example

![[Pasted image 20260213104335.png]]

## Relational Algebra Operations

|**Operation**|**Symbol**|**What it does**|**Changes Rows?**|**Changes Columns?**|**SQL Equivalent**|
|---|---|---|---|---|---|
|SELECT|σ|Filters tuples (rows) by condition|✅ Yes (≤ original)|❌ No|WHERE|
|PROJECT|π|Keeps only chosen attributes (columns)|⚠️ Maybe (duplicates removed)|✅ Yes|SELECT col1, col2 + DISTINCT|

```mermaid
%%{init: {"flowchart": {"htmlLabels": false, "nodeSpacing": 60, "rankSpacing": 90}}}%%
flowchart TB

    R["Relation R (Table)"]

    SEL["SELECT (σ)
    Filters rows by condition
    Same columns as R"]

    PROJ["PROJECT (π)
    Keeps chosen columns
    Removes duplicate rows"]

    OUT1["Result: New Relation"]
    OUT2["Result: New Relation"]

    R --> SEL --> OUT1
    R --> PROJ --> OUT2

    SQL1["SQL: WHERE"]
    SQL2["SQL: SELECT ... DISTINCT"]

    SEL --> SQL1
    PROJ --> SQL2
```