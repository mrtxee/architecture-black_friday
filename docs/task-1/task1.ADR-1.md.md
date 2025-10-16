
# Архитектура хранения данных онлайн-магазина «Мобильный мир»

## TO-BE

Проект мульти контейнерного приложения онлайн-магазина «Мобильный мир»
### Основные сервисы

| Сервис                 | Роль                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------- |
| **configSrv**          | Узел mongoDB, Хранит **метаданные о распределении данных**                            |
| **shard1**, **shard2** | Шарды mongoDB, Хранят реальные бизнес данные                                          |
| **mongos_router**      | Роутер mongoDB, принимает запросы → спрашивает `configSrv` → направляет в нужный шард |
| redis                  | служба L2-кэша                                                                        |
| pymango-api            | backend-api                                                                           |

---

### Схема взаимодействия сервисов

```mermaid
graph TD
    %% Сеть Docker
    subgraph "Docker Bridge Network: app-network"
        direction TB

        A[configSrv<br><small>Mongo Config Server</small><br><i>173.17.0.10:27017</i>]:::config

        B[mongos_router<br><small>Mongo Router</small><br><i>173.17.0.7:27020</i>]:::router

        C[shard1<br><small>Shard Node</small><br><i>173.17.0.9:27018</i>]:::shard
        D[shard2<br><small>Shard Node</small><br><i>173.17.0.8:27019</i>]:::shard

        E[pymango-api<br><small>Node.js / Spring Boot</small><br><i>173.17.0.5:3000</i>]:::api
        F[redis<br><small>Кэш данных</small><br><i>173.17.0.6:6379</i>]:::cache

        %% Связи
        E -->|Читает/пишет| B
        E -->|Кэширует| F
        F -->|Хранит| E
        B -->|Запрашивает метаданные| A
        B -->|Маршрутизирует| C
        B -->|Маршрутизирует| D
        C <--->|Обмен данными| D
    end

    %% Стили
    classDef config fill:#f8d7da,stroke:#c66,border:2px solid #c66,color:#721c24;
    classDef router fill:#cce5ff,stroke:#004085,stroke-width:2px,color:#004085;
    classDef shard fill:#d4edda,stroke:#155724,stroke-width:2px,color:#155724;
    classDef api fill:#e2e3e5,stroke:#383d41,stroke-width:2px,color:#383d41;
    classDef cache fill:#ffd54f,stroke:#e6a82e,stroke-width:2px,color:#333;

    style A fill:#f8d7da,stroke:#c66
    style B fill:#cce5ff,stroke:#004085
    style E fill:#e2e3e5,stroke:#383d41
    style F fill:#ffd54f,stroke:#e6a82e
```

#### Схема взаимодействия сервисов draw.io

![task1-TO-BE.drawio.png](task1-TO-BE.drawio.png "task1-TO-BE.drawio.png")

---

### Схема последовательности взаимодействия сервисов

```mermaid
sequenceDiagram
    User ->> pymango-api: GET /user/123
    pymango-api ->> redis: GET user:123
    alt В кэше есть
        redis -->> pymango-api: Данные из Redis
        pymango-api -->> User: Ответ
    else Нет в кэше
        pymango-api ->> mongos_router: Запрос к MongoDB
        mongos_router ->> shard1: Поиск по ID
        shard1 -->> mongos_router: Документ
        mongos_router -->> pymango-api: Данные
        pymango-api ->> redis: SET user:123 {...}
        pymango-api -->> User: Ответ
    end
```

---

### Идеи оптимизации

1. Только один `configSrv` в compose проекте – единая точка отказа, риск. Рекомендуется использовать Replica Set из 3+ `configSrv`
2. Разделить сеть на внутреннюю и внешнею, добавить API Gateaway
