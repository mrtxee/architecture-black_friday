# Контекст

Адаптация системы хранения «Мобильный мир» к динамическим, нерегулярным и предсказуемый скачкам трафика.
# Решение

Внедрение внешнего L2 кэша

**Основные сервисы**

| Сервис              | Роль                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------ |
| **configSrv**       | Узел mongoDB, Хранит метаданные о распределении данных                                                       |
| **mongos_router**   | Роутер mongoDB, принимает запросы → спрашивает `configSrv` → направляет в нужный шард                        |
| **pymango-api**     | backend-api                                                                                                  |
| **Replica Set**     | чтобы данные дублировались внутри региона и система не падала при сбое одного узла.                          |
| **Primary-shard**   | принимает запиcm                                                                                             |
| **Secondary-shard** | получает копию от primary                                                                                    |
| **Arbiter**         | голосует при выборе нового Primary (если старый упал), но не хранит данные. Дает гарантию нечетного кворума. |
| **redis**           | служба L2-кэша                                                                                               |

### Схема сервисов

```mermaid
graph TD
    %% Сеть Docker
    subgraph "Docker Bridge Network: app-network"
        direction TB

        A[configSrv<br><small>Mongo Config Server</small><br><i>173.17.0.10:27017</i>]:::config

        B[mongos_router<br><small>Mongo Router</small><br><i>173.17.0.7:27020</i>]:::router

        %% Shard 1 как Replica Set
        subgraph "Shard 1 Replica Set"
            C[shard1-primary<br><small>Primary</small><br><i>173.17.0.9:27018</i>]:::shard
            D[shard1-secondary<br><small>Secondary</small><br><i>173.17.0.4:27018</i>]:::shard
            E[shard1-arbiter<br><small>Arbiter</small><br><i>173.17.0.5:27018</i>]:::arbiter
            C <---> D
            D <---> E
            E <---> C
        end

        %% Shard 2 как Replica Set
        subgraph "Shard 2 Replica Set"
            F[shard2-primary<br><small>Primary</small><br><i>173.17.0.8:27019</i>]:::shard
            G[shard2-secondary<br><small>Secondary</small><br><i>173.17.0.6:27019</i>]:::shard
            H[shard2-arbiter<br><small>Arbiter</small><br><i>173.17.0.3:27019</i>]:::arbiter
            F <---> G
            G <---> H
            H <---> F
        end

        I[backend-api<br><small>Node.js / Spring Boot</small><br><i>173.17.0.2:3000</i>]:::api
        J[redis<br><small>Кэш данных</small><br><i>173.17.0.1:6379</i>]:::cache

        %% Связи
        I -->|Read/Write| B
        I -->|Cache| J
        J -->|Data store| I
        B -->|Metadate query| A
        B -->|Route to shard1| C
        B -->|Route to shard2| F
    end

    %% Стили
    classDef config fill:#f8d7da,stroke:#c66,border:2px solid #c66,color:#721c24;
    classDef router fill:#cce5ff,stroke:#004085,stroke-width:2px,color:#004085;
    classDef shard fill:#d4edda,stroke:#155724,stroke-width:2px,color:#155724;
    classDef arbiter fill:#e9ecef,stroke:#6c757d,stroke-dasharray:5,5,color:#6c757d;
    classDef api fill:#e2e3e5,stroke:#383d41,stroke-width:2px,color:#383d41;
    classDef cache fill:#ffd54f,stroke:#e6a82e,stroke-width:2px,color:#333;

    style A fill:#f8d7da,stroke:#c66
    style B fill:#cce5ff,stroke:#004085
    style I fill:#e2e3e5,stroke:#383d41
    style J fill:#ffd54f,stroke:#e6a82e
```

---
### схема сервисов drawio
схема
![task1-TO-BE_ADR3.drawio.png](task1-TO-BE_ADR3.drawio.png)

---

### Схема последовательности взаимодействия сервисов

```mermaid
sequenceDiagram
    product ->> pymango-api: GET /product/123
    pymango-api ->> redis: GET product:123
    alt В кэше есть
        redis -->> pymango-api: Данные из Redis
        pymango-api -->> product: Ответ
    else Нет в кэше
        pymango-api ->> mongos_router: Запрос к MongoDB
        mongos_router ->> shard1: Поиск по ID
        shard1 -->> mongos_router: Документ
        mongos_router -->> pymango-api: Данные
        pymango-api ->> redis: SET product:123 {...}
        pymango-api -->> product: Ответ
    end
```

---

### Схема последовательности репликации
```mermaid
sequenceDiagram
    mongos_router ->> shard1-primary: INSERT {product:"PC-T800"}
    shard1-primary -->> shard1-secondary: Реплицирует данные
    shard1-primary -->> shard1-arbiter: Голосует за статус
    shard1-arbiter -->> shard1-primary: Подтверждает кворум
    shard1-primary -->> mongos_router: OK
```

# Последствия

* кратное снижение нагрузки на систему хранения
* риск того, что клиент получит устаревшие данные.