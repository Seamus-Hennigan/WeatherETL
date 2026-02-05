# WeatherETL

---
## 📋 Table of Contents

* [Purpose](#purpose)
* [Features](#features)
* [Project Metrics](#project-metrics)
* [Technology Stack & Rationale](#technology-stack)
* [Complete List of Tools & Services](#tools)
* [Technical Diagrams](#diagrams)
   * [Sequence Diagram](#sequence)
   * [AWS Architecture Diagram](#architecture)
   * [Component Diagram](#component)
   * [Exception Flow Diagram](#exception)
   * [ERD Diagram](#erd)
* [Database Schema](#schema)
* [Implementation Challenges & Solutions](#challenges)
* [Cost Analysis](#cost)
* [Setup Guide](#setup)
* [Troubleshooting](#troubleshooting)
* [Contact](#contact)
---
## Purpose
This project demonstrates the design and implementation of a **production-grade, cloud-based ETL (Extract, Transform, Load) pipeline** for automated weather data collection and visualization. Built entirely on AWS infrastructure, it showcases modern data engineering practices including serverless computing, error handling, cost optimization, and interactive data visualization.

---
## Features
- **Automated Data Collection:** Hourly weather data ingestion from OpenWeatherMap API across 5 cities
- **Robust Error Handling:** 3-tier retry logic with exponential backoff and S3 buffering for failed database connections
- **Real-Time Visualization:** Interactive Superset dashboards with city filtering, trend analysis, and cross-chart interactivity
- **Data Quality Monitoring:** Automated logging of API response times, data completeness scores, and validation status
- **Scalable Architecture:** Serverless ETL pipeline supporting multiple cities and extensible to additional data sources
- **Comprehensive Error Tracking:** Separate storage paths for buffered retries and malformed data investigation
- **Multi-Table Schema:** Normalized database design with cities, observations, details, and quality logs
---
## Project Metrics

| Metric | Value |
|--------|-------|
| **Cities Monitored** | 5 (London, Paris, Tokyo, New York, Sydney) |
| **Data Points Collected** | 120+ per day (24 collections × 5 cities) |
| **Monthly Cost** | Less than $10.00 a month |
| **Data Retention** | 30 days (S3 raw), Unlimited (PostgreSQL) |
| **Dashboard Refresh** | Real-time with interactive filtering |
--- 

## Technology Stack & Rationale

| Technology | Purpose | Why This Choice |
|------------|---------|-----------------|
| **AWS Lambda** | Serverless ETL execution | Pay-per-use pricing, automatic scaling, no server management overhead |
| **AWS EventBridge** | Scheduled triggers | Native AWS cron scheduling, reliable event-driven architecture |
| **Amazon S3** | Raw data storage & buffering | Cost-effective object storage, event notifications |
| **EC2 + PostgreSQL** | Database hosting | full control over configuration |
| **Apache Superset** | Data visualization | Open-source BI tool, SQL-native, supports complex dashboards and filtering |
| **Docker** | Container runtime | Consistent development environment, simplified Superset deployment |
| **Python 3.12+** | ETL scripting | native AWS Lambda support, readable syntax |
| **pg8000** | PostgreSQL driver | Pure Python implementation, Lambda-compatible (must be incorperated as a layer), no C dependencies |
| **Mermaid** | Documentation diagrams | Version-controlled diagrams, renders in GitHub, easy to update |

---
## Complete List of Tools & Services

### AWS Services
- **AWS Lambda** - Serverless compute for Extract and Transform/Load functions
- **AWS EventBridge** - Cron-based scheduling for hourly data collection
- **Amazon S3** - Object storage for raw data, buffered retries, and malformed data
- **Amazon EC2** - Virtual server hosting PostgreSQL database (t3.micro instance)
- **Amazon EBS** - Block storage for EC2 instance (20 GB gp3)
- **AWS VPC** - Virtual Private Cloud for network isolation
- **Security Groups** - Firewall rules for EC2 instance (ports 5432, 22)
- **Elastic IP** - Static IP address for EC2 instance
- **CloudWatch Logs** - Centralized logging for Lambda functions
---
## Technical Diagrams

### Sequence Diagram
```mermaid
sequenceDiagram
    participant EB as EventBridge
    participant L1 as Lambda Extract
    participant API as OpenWeather API
    participant S3Raw as S3 raw/
    participant L2 as Lambda Transform
    participant S3Clean as S3 clean/
    participant DB as EC2 PostgreSQL
    participant SU as Superset
    
    EB->>L1: Trigger (hourly)
    L1->>API: GET /weather?q=city
    
    alt API Success
        API-->>L1: JSON response
        L1->>S3Raw: PUT raw data
        S3Raw->>L2: S3 Event trigger
        L2->>L2: Transform data
        
        alt DB Available
            L2->>DB: INSERT into tables
            DB-->>L2: Success
            Note over L2: Data loaded successfully
        else DB Unavailable
            L2-xDB: Connection failed (retry 3x)
            L2->>S3Clean: Buffer to clean/buffered/
            Note over S3Clean: Retry later
        else Malformed Data
            L2->>S3Clean: Save to clean/malformed/
            Note over S3Clean: For investigation
        end
        
    else API Failure
        API--xL1: Timeout/Error
        Note over L1: Log error to CloudWatch
    end
    
    Note over SU,DB: User accesses dashboard
    SU->>DB: SELECT queries
    DB-->>SU: Weather data
```

### AWS Architecture Diagram
```mermaid
---
config:
  layout: dagre
---
flowchart TB
 subgraph External["External Services"]
        API["OpenWeatherMap API"]
  end
 subgraph DB["PostgreSQL Database"]
        T1["cities"]
        T2["weather_observations"]
        T3["weather_details"]
        T4["data_quality_log"]
  end
 subgraph Subnet["Public Subnet: 172.31.0.0/20<br/>AZ: us-east-1a"]
        EC2["EC2 Instance<br>Type: t3.micro<br>OS: Amazon Linux 2023<br>PostgreSQL 17"]
        DB
        SG["Security Group<br>Inbound:<br>• 5432 from 0.0.0.0/0<br>• 22 from 0.0.0.0/0"]
  end
 subgraph VPC["VPC: 172.31.0.0/16"]
    direction TB
        Subnet
  end
 subgraph S3["S3: weather-data-raw-sch"]
        S3Raw["raw/<br/>Raw API Data"]
        S3Buffered["clean/buffered/<br/>DB Unavailable"]
        S3Malformed["clean/malformed/<br/>Invalid Data"]
  end
 subgraph AWS["AWS Cloud Region: us-east-1"]
    direction TB
        EB["EventBridge<br/>cron: 0 13 * * ? *<br>Every day 8am EST"]
        L1["Lambda: Extract<br/>Python 3.14 | 128MB"]
        S3
        L2["Lambda: Transform & Load<br/>Python 3.12 | 256MB<br/>Retry: 3x exponential backoff"]
        VPC
  end
 subgraph Local["Local Environment"]
        Docker["Docker Container<br>Apache Superset"]
  end
    API <-- HTTP GET --> L1
    EB -- Trigger --> L1
    L1 -- PUT --> S3Raw
    S3Raw -. Event .-> L2
    L2 -- INSERT --> EC2
    L2 -.->|On DB Failure| S3Buffered
    L2 -.->|Malformed| S3Malformed
    EC2 --> T1 & T2 & T3 & T4
    EC2 -- Protected by --- SG
    Docker -- Query --> EC2

     API:::external
     T1:::database
     T2:::database
     T3:::database
     T4:::database
     EC2:::compute
     SG:::security
     EB:::orchestration
     L1:::compute
     S3Raw:::storage
     S3Buffered:::buffer
     S3Malformed:::error
     L2:::compute
     Docker:::external
    classDef orchestration fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef compute fill:#ED7100,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef storage fill:#3F8624,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef buffer fill:#FFA500,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef error fill:#FF6B6B,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef database fill:#2E73B8,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef external fill:#0066CC,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef security fill:#DD344C,stroke:#232F3E,stroke-width:2px,color:#fff
```
### Component Diagram
```mermaid
graph TB
    subgraph ExtractLayer["Extract Layer"]
        API[OpenWeather API Client]
        Validate[Response Validator]
        Store[S3 Writer]
    end
    
    subgraph TransformLayer["Transform Layer"]
        Reader[S3 Reader]
        Transform[Data Transformer]
        Loader[DB Loader]
        Buffer[Buffer Manager]
    end
    
    subgraph DataLayer["Data Layer"]
        DB[(PostgreSQL)]
        S3[(S3 Buckets)]
    end
    
    subgraph VisualizationLayer["Visualization"]
        Superset[Apache Superset]
        Dashboards[Dashboards]
    end
    
    API --> Validate --> Store --> S3
    S3 --> Reader --> Transform --> Loader --> DB
    Transform -.->|On Failure| Buffer --> S3
    DB --> Superset --> Dashboards
```

### Exception flow diagram
```mermaid
flowchart TD
    Start[Lambda 2 Triggered<br/>S3 Event] --> ReadS3{Read from S3}
    
    ReadS3 -->|Success| Transform{Transform Data}
    ReadS3 -->|File Not Found| LogS3Error[Log Error]
    
    Transform -->|Success| DBWrite{Write to DB}
    Transform -->|Malformed| BufferMalformed
    Transform -->|DB Error| BufferMalformed
    
    DBWrite -->|Success| QualityLog[Insert to<br/>data_quality_log]
    DBWrite -->|Connection Failed| Retry1{Retry 1/3<br/>Wait 2s}
    DBWrite -->|Data Error| LogDataError[Log to<br/>data_quality_log]
    
    Retry1 -->|Success| QualityLog
    Retry1 -->|Failed| Retry2{Retry 2/3<br/>Wait 4s}
    
    Retry2 -->|Success| QualityLog
    Retry2 -->|Failed| Retry3{Retry 3/3<br/>Wait 8s}
    
    Retry3 -->|Success| QualityLog
    Retry3 -->|Failed| BufferClean[Save to<br/>clean/buffered/]
    
    QualityLog --> Success[Complete<br/>Status: 200]
    BufferClean --> Deferred[Complete<br/>Status: 202<br/>Retry Later]
    BufferMalformed --> Investigate[Complete<br/>Status: 500<br/>Needs Investigation]
    LogS3Error --> Failed[Complete<br/>Status: 404/500]
    LogDataError --> LoggedError[Complete<br/>Status: 500<br/>Logged]
    
    style Transform fill:#FFD700
    style DBWrite fill:#FFD700
    style BufferClean fill:#FFA500
    style BufferMalformed fill:#FF6B6B
    style Success fill:#51CF66
    style Deferred fill:#FFA500
    style Investigate fill:#FF6B6B
    style Retry1 fill:#87CEEB
    style Retry2 fill:#87CEEB
    style Retry3 fill:#87CEEB
```

### ERD Diagram
```mermaid
erDiagram
    CITIES ||--o{ WEATHER_OBSERVATIONS : has
    CITIES ||--o{ DATA_QUALITY_LOG : tracks
    WEATHER_OBSERVATIONS ||--|| WEATHER_DETAILS : contains
    WEATHER_OBSERVATIONS ||--o{ DATA_QUALITY_LOG : logs

    CITIES {
        INT city_id PK
        STRING city_name
        STRING country_code
        FLOAT latitude
        FLOAT longitude
        STRING timezone
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    WEATHER_OBSERVATIONS {
        INT observation_id PK
        INT city_id FK
        TIMESTAMP observation_timestamp
        TIMESTAMP fetch_timestamp
        FLOAT temperature_celsius
        FLOAT temperature_fahrenheit
        FLOAT feels_like_celsius
        FLOAT feels_like_fahrenheit
        INT humidity
        INT pressure
        STRING weather_main
        STRING weather_description
    }

    WEATHER_DETAILS {
        INT detail_id PK
        INT observation_id FK
        FLOAT temp_min_celsius
        FLOAT temp_max_celsius
        INT sea_level_pressure
        INT ground_level_pressure
        FLOAT wind_speed
        INT wind_direction
        FLOAT wind_gust
        INT cloudiness
        INT visibility
        STRING weather_icon
        TIMESTAMP sunrise
        TIMESTAMP sunset
    }

    DATA_QUALITY_LOG {
        INT log_id PK
        INT city_id FK
        INT observation_id FK
        TIMESTAMP fetch_timestamp
        INT api_response_time_ms
        FLOAT data_completeness_score
        BOOLEAN has_nulls
        BOOLEAN validation_passed
        STRING error_message
        TIMESTAMP created_at
    }
```
---
## Database Schema

### Create the Database
```sql
CREATE DATABASE weatherETL;
```
### CITIES
```sql

CREATE TABLE CITIES (
    city_id INT AUTO_INCREMENT PRIMARY KEY,
    city_name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL,
    latitude FLOAT NOT NULL,
    longitude FLOAT NOT NULL,
    timezone VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```
### WEATHER_OBSERVATIONS
```SQL

CREATE TABLE WEATHER_OBSERVATIONS (
    observation_id INT AUTO_INCREMENT PRIMARY KEY,
    city_id INT NOT NULL,
    observation_timestamp TIMESTAMP NOT NULL,
    fetch_timestamp TIMESTAMP NOT NULL,
    temperature_celsius FLOAT,
    temperature_fahrenheit FLOAT,
    feels_like_celsius FLOAT,
    feels_like_fahrenheit FLOAT,
    humidity INT,
    pressure INT,
    weather_main VARCHAR(100),
    weather_description VARCHAR(255),
    
    CONSTRAINT fk_weather_city
        FOREIGN KEY (city_id) REFERENCES CITIES(city_id)
        ON DELETE CASCADE
);
```
### WEATHER_DETAILS
```sql
CREATE TABLE WEATHER_DETAILS (
    detail_id INT AUTO_INCREMENT PRIMARY KEY,
    observation_id INT NOT NULL,
    temp_min_celsius FLOAT,
    temp_max_celsius FLOAT,
    sea_level_pressure INT,
    ground_level_pressure INT,
    wind_speed FLOAT,
    wind_direction INT,
    wind_gust FLOAT,
    cloudiness INT,
    visibility INT,
    weather_icon VARCHAR(10),
    sunrise TIMESTAMP NULL,
    sunset TIMESTAMP NULL,

    CONSTRAINT fk_details_observation
        FOREIGN KEY (observation_id) REFERENCES WEATHER_OBSERVATIONS(observation_id)
        ON DELETE CASCADE
);
```
## DATA_QUALITY_LOG
```sql

CREATE TABLE DATA_QUALITY_LOG (
    log_id INT AUTO_INCREMENT PRIMARY KEY,
    city_id INT NOT NULL,
    observation_id INT NOT NULL,
    fetch_timestamp TIMESTAMP NOT NULL,
    api_response_time_ms INT,
    data_completeness_score FLOAT,
    has_nulls BOOLEAN,
    validation_passed BOOLEAN,
    error_message VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_log_city
        FOREIGN KEY (city_id) REFERENCES CITIES(city_id)
        ON DELETE SET NULL,

    CONSTRAINT fk_log_observation
        FOREIGN KEY (observation_id) REFERENCES WEATHER_OBSERVATIONS(observation_id)
        ON DELETE SET NULL
);
```
---
## Setup Guide
---




