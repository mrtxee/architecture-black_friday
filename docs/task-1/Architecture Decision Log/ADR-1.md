# Контекст

Адаптация системы хранения «Мобильный мир» к динамическим, нерегулярным и предсказуемый скачкам трафика.
# Решение

Декомпозиция базы на шарды. Key-based распределение данных по шардам через MongoDb Config Server

**Основные сервисы**

| Сервис                 | Роль                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------- |
| **configSrv**          | Узел mongoDB, Хранит **метаданные о распределении данных**                            |
| **shard1**, **shard2** | Шарды mongoDB, Хранят реальные бизнес данные                                          |
| **mongos_router**      | Роутер mongoDB, принимает запросы → спрашивает `configSrv` → направляет в нужный шард |
| **pymango-api**        | backend-api                                                                           |

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

        %% Связи
        E -->|Читает/пишет| B
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

    style A fill:#f8d7da,stroke:#c66
    style B fill:#cce5ff,stroke:#004085
    style E fill:#e2e3e5,stroke:#383d41
```

shema
![task1-TO-BE_ADR1.drawio.png](task1-TO-BE_ADR1.drawio.png)
# Последствия

* кратное снижение нагрузки за счет распределения данных по на несколько шардов
* отсутствие консистентности в случае расщепления системы хранения на изолированные сегменты 
* издержки на поддержание мультиконтейнерной системы хранения