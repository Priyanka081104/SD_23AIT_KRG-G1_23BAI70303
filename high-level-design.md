# High-Level Design - Food Delivery System

## System Architecture Diagram

[Customer App] ──┐
[Agent App] ──┼── [Load Balancer] ── [API Gateway]
[Restaurant App]─┘

│
▼
[Order Service]
│
┌────────────┼────────────┐
▼ ▼ ▼
[Agent Service] [Location Service] [Notification Service]
│ │
▼ ▼
[Redis Geo] [Kafka] ──► [ETA Calculator]
│ │
▼ ▼
[PostgreSQL] [TimescaleDB]


## Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| Load Balancer | NGINX | Distribute traffic across servers |
| API Gateway | Kong | Authentication, rate limiting, routing |
| Order Service | Spring Boot / Node.js | Manage order lifecycle |
| Agent Service | Spring Boot / Node.js | Manage delivery agent assignment |
| Location Service | Spring Boot / Node.js | Process real-time location updates |
| Notification Service | Spring Boot / Node.js | Send push notifications |
| Message Queue | Apache Kafka | Handle high-volume location updates |
| Cache | Redis + GeoSet | Store live agent locations |
| Time-series DB | TimescaleDB | Store historical location data |
| Relational DB | PostgreSQL | Store orders, users, restaurants |

## Data Flow

1. **Agent sends location** → API Gateway → Kafka → Location Service → Redis + TimescaleDB
2. **Customer tracks order** → API Gateway → Order Service → Location Service → Redis → Response
3. **Customer places order** → API Gateway → Order Service → Agent Service → Notification Service
