| No.  | タイプ | 物理名         | 論理名                   |
| :--- | :----- | :------------- | :----------------------- |
| 1    | table  | drinks         | 飲み物                   |
| 2    | table  | drink_counters | 飲酒杯数カウントテーブル |


```mermaid
erDiagram
  drinks ||--o{ drink_counters : ""
  drinks {
    int8 id PK "identity"
    varchar name
    timestamp created_at "DEFAULT now()"
  }
  drink_counters {
    int8 id PK "identity"
    date date
    int8 drink_id
    int4 count
    timestamp created_at "DEFAULT now()"
    uuid user_id "DEFAULT auth.uid()"
  }
```
