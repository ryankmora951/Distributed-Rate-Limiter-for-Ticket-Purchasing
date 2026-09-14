# Distributed Rate Limiter — Ticket Purchasing System

A distributed, high-concurrency ticket purchasing backend built to explore rate limiting, atomic
inventory management, and virtual queueing under load — the kind of problem real ticket-sale
platforms (Ticketmaster-style "on-sale" drops) face when thousands of users hit "Buy" in the same
second.

> 🚧 **Status:** Early development — currently working through Phase 1 (basic API skeleton).

## Why This Project

When a popular event goes on sale, a system has to:
1. **Rate-limit** requests per user/IP so no one can hammer the API (bots, retries).
2. **Protect ticket inventory** so two people can't buy the last seat at the same time (no overselling).
3. **Queue people fairly** when demand exceeds supply (virtual waiting room), instead of just
   returning errors to everyone.
4. **Scale horizontally** — multiple app instances behind a load balancer, all sharing consistent
   state through Redis.

The rate limiter and inventory counters must stay correct even when multiple copies of the app are
running at once.

## Tech Stack

- **Java 21** (Amazon Corretto)
- **Spring Boot 4.x** — REST API, JPA
- **Redis** — atomic rate limiting counters, inventory locks, waiting-room queue
- **PostgreSQL** — persistent storage for orders, users, events
- **Docker / Docker Compose** — local development environment
- **AWS** (planned) — ECS Fargate, ElastiCache, RDS, ALB

## Architecture

```
                        ┌─────────────────────┐
                        │   Client / Postman   │
                        └──────────┬───────────┘
                                   │ HTTPS
                        ┌──────────▼───────────┐
                        │   Load Balancer        │
                        └──────────┬───────────┘
                 ┌─────────────────┼─────────────────┐
        ┌────────▼───────┐ ┌───────▼────────┐ ┌───────▼────────┐
        │ Spring Boot App │ │ Spring Boot App │ │ Spring Boot App │  (stateless instances)
        └────────┬────────┘ └───────┬────────┘ └───────┬────────┘
                 └──────────────────┼───────────────────┘
                     ┌──────────────▼──────────────┐
                     │           Redis              │  ← rate limits, inventory locks, queue
                     └──────────────┬──────────────┘
                     ┌──────────────▼──────────────┐
                     │        PostgreSQL             │  ← orders, users, events (source of truth)
                     └───────────────────────────────┘
```

## Core Components

| Component | Responsibility | Key Tech |
|---|---|---|
| Rate Limiter Filter | Reject/allow requests per user or IP | Spring Interceptor + Redis (Lua script) |
| Inventory Service | Atomically decrement ticket count, prevent overselling | Redis DECR/Lua |
| Virtual Waiting Room | Issue queue tokens, admit users in order under high demand | Redis Sorted Set |
| Order Service | Persist confirmed purchases | Spring Data JPA + PostgreSQL |
| Event/Ticket API | CRUD for events and ticket types | Spring Boot REST controllers |
| Observability | Metrics and logging under load | Spring Actuator + Micrometer |

## Roadmap

- [ ] **Phase 1** — Basic API skeleton (Event/Ticket entities, CRUD endpoints, Postgres connection)
- [ ] **Phase 2** — Naive rate limiting (Fixed Window Counter via Redis)
- [ ] **Phase 3** — Atomic concurrency-safe purchasing (Redis Lua scripts, no overselling under load)
- [ ] **Phase 4** — Upgraded rate limiting algorithms (Sliding Window Log, Token Bucket)
- [ ] **Phase 5** — Virtual waiting room for high-demand events
- [ ] **Phase 6** — Full Dockerization (app + Redis + Postgres via Docker Compose)
- [ ] **Phase 7** — Observability and load testing (k6/Gatling)
- [ ] **Phase 8** — AWS deployment (ECS Fargate, ElastiCache, RDS, ALB, CI/CD)

## Getting Started

### Prerequisites
- Java 21 (Amazon Corretto recommended)
- Maven (or use the included `./mvnw` wrapper)
- Docker Desktop

### Running Locally
```bash
# Start Redis and Postgres
docker compose up -d

# Run the app
./mvnw spring-boot:run
```

The app will start on `http://localhost:8080`.

## License

MIT License