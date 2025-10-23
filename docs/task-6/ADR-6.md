# Контекст

Обеспечения точек присутствия (PoP) в основных регионах ведения бизнеса для оптимальной гарантии качества и эффективности сервиса клиентам компании путем организации CDN.

# Решение

При организации Content Delivery Network принято решение внедрить метод разделения данных Anycast. 

На текущем этапе развития компании не достигнута достаточно равномерная дисперсия геокоординат клиентов по сетевым адресам. 

Целесообразно повысить производительность и устойчивость DNS путем наращивания числа точек присутствия с одинаковым адресом, что обеспечит оптимизацию роутинга для клиентов компании в зависимости от особенностей сетевой топологии каждого клиента. 


### Cхема сервисов drawio
![taks6_ADR6.drawio.png](taks6_ADR6.drawio.png)


### Схема сервисов
```mermaid
graph TD
    %% Определения стилей для имитации внешнего вида
    classDef client fill:#E0FFFF, stroke:#99CCFF, color:#000, stroke-width:2px;
    classDef gateway fill:#99CCFF, stroke:#4682B4, color:#000;
    classDef app fill:#99CCFF, stroke:#4682B4, color:#000;
    classDef cache fill:#FFC0CB, stroke:#CD5C5C, color:#000, shape:cylinder;
    classDef db_group fill:#A9A9A9, stroke:#000, stroke-width:2px;
    classDef cdn fill:#FFC0CB, stroke:#CD5C5C, color:#000, shape:cloud;

    %% Уровни (для аннотаций)

    %% Основные компоненты
    subgraph "Внешние пользователи"
        A[User]:::client
    end

    subgraph "On-Permise Мобильный мир"
        B[APISIX Gateway]:::gateway
        C[Consul]:::gateway
        D1[Java<br>pymongo-api]:::app
        D2[Java<br>pymongo-api]:::app
        D3[Java<br>pymongo-api]:::app
    end

    subgraph "Clouds"
        G[CDN]:::cdn
        
    %% ИСПРАВЛЕННЫЙ БЛОК: Удалена строка 'direction TD'
        subgraph "MongoDB ЦОД 1"
            E1[redis cache 1]:::cache
            F1_db[(MongoDB)]
        end
        class F1_db db_group

    %% ИСПРАВЛЕННЫЙ БЛОК: Удалена строка 'direction TD'
        subgraph "MongoDB ЦОД 2"
            E2[redis cache 2]:::cache
            F2_db[(MongoDB)]
        end
        class F2_db db_group

    %% ИСПРАВЛЕННЫЙ БЛОК: Удалена строка 'direction TD'
        subgraph "MongoDB ЦОД 3"
            E3[redis cache 3]:::cache
            F3_db[(MongoDB)]
        end
        class F3_db db_group

        
    end

    %% Потоки данных (Верхний уровень)
    A -->| | B
    A -->| | G

    B --> C
    B --> D1
    B --> D2
    B --> D3

    C --> D1
    C --> D2
    C --> D3

    %% Потоки данных (App -> DB)
    D1 -->| | F1_db
    D2 -->| | F2_db
    D3 -->| | F3_db

    %% Потоки данных (Cache -> App - для чтения, имитируем обратный поток)
    E1 -->| | D1
    E2 -->| | D2
    E3 -->| | D3
    
    %% Взаимодействие между ЦОД (двунаправленное)
    F1_db <--> F2_db
    F2_db <--> F3_db
    

```

---
### Последовательность взаимодействий сервисов
```mermaid
sequenceDiagram
    participant User as Внешний пользователь
    participant APISIX as APISIX Gateway
    participant Consul as Consul (Service Discovery)
    participant API as Java pymongo-api
    participant Cache as Redis Cache
    participant DB as MongoDB ЦОД (DB)

    User->>APISIX: HTTP-запрос (поиск/данные)
    activate APISIX

    APISIX->>Consul: Запрос адреса API-сервиса
    activate Consul
    Consul-->>APISIX: Возврат адреса (D1, D2 или D3)
    deactivate Consul

    APISIX->>API: Запрос данных
    deactivate APISIX
    activate API

    API->>Cache: Чтение данных (Кэш-хит?)
    activate Cache
    alt Кэш-хит
        Cache-->>API: Возврат данных из кэша
    else Кэш-мисс
        Cache-->>API: Нет данных
        
        API->>DB: Запрос данных (select)
        activate DB
        DB-->>API: Возврат данных (результат)
        deactivate DB
        
        API->>Cache: Запись данных (кэширование)
        Cache-->>API: Подтверждение записи
    end
    deactivate Cache
    
    API-->>User: HTTP-ответ (Данные)
    deactivate API

    User->>CDN: Запрос статического контента (параллельно)
    activate CDN
    CDN-->>User: Возврат статических данных
    deactivate CDN
```

# Последствия
1. геошардинг сервиса компании с учетом особенностей топологии роутинга клиента в каждом регионе присутствия
