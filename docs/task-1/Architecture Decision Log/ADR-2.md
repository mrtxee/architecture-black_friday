# Контекст

Адаптация системы хранения «Мобильный мир» к динамическим, нерегулярным и предсказуемый скачкам трафика.
# Решение

Реструктурирование шардов в Replica Set-ы с выделением арбитров для гарантии механизма выбора лидера в случае падения primary шарда

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


### Схема взаимодействия сервисов

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

        %% Связи
        I -->|Read/Write| B
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

    style A fill:#f8d7da,stroke:#c66
    style B fill:#cce5ff,stroke:#004085
    style I fill:#e2e3e5,stroke:#383d41
```

shema
![task1-TO-BE_ADR2.drawio.png](task1-TO-BE_ADR2.drawio.png)
# Последствия

* снижения риска получить неконсистентное состояние системы хранения за счет дублирования вероятных точек сбоя