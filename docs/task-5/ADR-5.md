# Контекст

Адаптация системы хранения «Мобильный мир» к динамическим, нерегулярным и предсказуемый скачкам трафика.
# Решение

Для балансировки нагрузки повышения устойчивости системы вводим контейнеры (сервисы)
 - APISIX Gateaway
   - роли
     1. Авторизация запросов
     2. Балансировка нагрузки, маршрутизация
     3. Безопасность (WAF, CORS, Rate limiting)
   - Consul
     - роли
       1. Регистрация, поиск сервисов
       2. Мониторинг здоровья сервисов
       3. Хранилище конфигурации сервисов

### Схема сервисов

---
### Cхема сервисов drawio
схема
![task1-taks5_ADR5.drawio.png](task1-taks5_ADR5.drawio.png)
---

### Схема взаимодействия сервисов
```mermaid
graph LR
%% ========== CLIENT LAYER ==========
    CLIENT[User] -->|HTTP запросы| APISIX[APISIX Gateway]

%% ========== GATEWAY LAYER ==========
    APISIX -->|Service Discovery запрос| CONSUL[Consul Server]

%% ========== SERVICE DISCOVERY LAYER ==========
    CONSUL -->|чтение/запись| CONSUL_KV[Consul KV Store]

%% ========== APPLICATION LAYER ==========
    APISIX -->|балансировка нагрузки| API1[Java Service<br>Pymango-api 1]
    APISIX -->|балансировка нагрузки| API2[Java Service<br>Pymango-api 2]
    APISIX -->|балансировка нагрузки| API3[Java Service<br>Pymango-api 3]

%% Service Registration
    API1 -->|регистрация + healthcheck| CONSUL
    API2 -->|регистрация + healthcheck| CONSUL
    API3 -->|регистрация + healthcheck| CONSUL

%% ========== CACHE LAYER ==========
    API1 -->|кэширование чтений| REDIS[Redis Cache]
    API2 -->|кэширование чтений| REDIS
    API3 -->|кэширование чтений| REDIS

%% ========== DATABASE LAYER ==========
%% MongoDB Router
    API1 -->|запросы данных| MONGOS[MongoS Router]
    API2 -->|запросы данных| MONGOS
    API3 -->|запросы данных| MONGOS

%% Config Server
    MONGOS -->|метаданные кластера| CONFIGSRV[Config Server]

%% Shard 1 - Replica Set
    MONGOS -->|маршрутизация| SHARD1_PRIMARY[Shard1 Primary]
    SHARD1_PRIMARY -->|репликация| SHARD1_SECONDARY1[Shard1 Secondary 1]
    SHARD1_PRIMARY -->|репликация| SHARD1_SECONDARY2[Shard1 Secondary 2]

%% Shard 2 - Replica Set  
    MONGOS -->|маршрутизация| SHARD2_PRIMARY[Shard2 Primary]
    SHARD2_PRIMARY -->|репликация| SHARD2_SECONDARY1[Shard2 Secondary 1]
    SHARD2_PRIMARY -->|репликация| SHARD2_SECONDARY2[Shard2 Secondary 2]

%% ========== STYLING ==========
classDef client fill:#3498db,stroke:#fff,color:#fff
classDef gateway fill:#9b59b6,stroke:#fff,color:#fff
classDef discovery fill:#f39c12,stroke:#fff,color:#fff
classDef application fill:#2ecc71,stroke:#fff,color:#fff
classDef cache fill:#ff6b6b,stroke:#fff,color:#fff
classDef database fill:#45b7d1,stroke:#fff,color:#fff

class CLIENT client
class APISIX gateway
class CONSUL,CONSUL_KV discovery
class API1,API2,API3 application
class REDIS cache
class MONGOS,CONFIGSRV,SHARD1_PRIMARY,SHARD1_SECONDARY1,SHARD1_SECONDARY2,SHARD2_PRIMARY,SHARD2_SECONDARY1,SHARD2_SECONDARY2 database


%% ========== LINK STYLING ==========
%% Client to Gateway
linkStyle 0 stroke:#3498db,stroke-width:2px

%% Service Discovery
linkStyle 1 stroke:#f39c12,stroke-width:2px
linkStyle 2 stroke:#f39c12,stroke-width:2px,stroke-dasharray: 5,5

%% Service Registration
linkStyle 3 stroke:#27ae60,stroke-width:2px,stroke-dasharray: 5,5
linkStyle 4 stroke:#27ae60,stroke-width:2px,stroke-dasharray: 5,5
linkStyle 5 stroke:#27ae60,stroke-width:2px,stroke-dasharray: 5,5

%% Load Balancing
linkStyle 6 stroke:#9b59b6,stroke-width:2px
linkStyle 7 stroke:#9b59b6,stroke-width:2px
linkStyle 8 stroke:#9b59b6,stroke-width:2px

%% Cache Connections
linkStyle 9 stroke:#e74c3c,stroke-width:2px
linkStyle 10 stroke:#e74c3c,stroke-width:2px
linkStyle 11 stroke:#e74c3c,stroke-width:2px

%% Database Connections
linkStyle 12 stroke:#3498db,stroke-width:2px
linkStyle 13 stroke:#3498db,stroke-width:2px
linkStyle 14 stroke:#3498db,stroke-width:2px

%% MongoDB Internal
linkStyle 15 stroke:#2980b9,stroke-width:2px
linkStyle 16 stroke:#2980b9,stroke-width:2px
linkStyle 17 stroke:#2980b9,stroke-width:2px
linkStyle 18 stroke:#2980b9,stroke-width:2px
linkStyle 19 stroke:#2980b9,stroke-width:2px
linkStyle 20 stroke:#2980b9,stroke-width:2px
```

---

### Последовательность взаимодействий сервисов
```mermaid
sequenceDiagram
    participant C as Client
    participant G as APISIX
    participant Consul as Consul
    participant API as Java Service
    participant R as Redis
    participant M as MongoDB
    
    C->>G: GET /api/products/123
    G->>Consul: Get healthy java-service instances
    Consul->>G: [API1:8080, API2:8080, API3:8080]
    G->>API: Route to API1
    API->>R: Check cache for "product:123"
    R->>API: Cache miss
    API->>M: Query MongoDB
    M->>API: Product data
    API->>R: Cache product:123
    API->>G: Response
    G->>C: Product data
```
---

# Последствия

1. рост стоимость потребляемых ресурсов
2. устойчивость в экстремальным нагрузками, DDOS-атакам
3. кратный прирост стабильности сервиса
4. повышенные SLA гарантий по метрикам 
   1. время отклика (ms)
   2. пропускной способности приложения (RPS)
   3. времени доступности сервиса (%)
   4. задержка передачи данных (p <= %)  
