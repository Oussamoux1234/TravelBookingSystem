# Travel Booking System

A scalable microservices-based travel booking platform built with Java and Spring Boot. The system is designed to manage hotel data, flight data, user authentication, and reservations through a central API gateway and service discovery layer.

## Overview

The Travel Booking System is a distributed backend architecture for a travel application. Instead of placing all logic in one monolithic service, the project is divided into independent services, each responsible for a specific domain:

- hotel management
- flight management
- reservation and payment workflows
- authentication and user management
- API routing and service discovery

This modular design makes the platform easier to maintain, scale, and evolve as new features are added.

## Why this project uses microservices

A travel platform naturally has multiple domains that evolve at different speeds:

- hotel availability and rooms change differently from flight schedules;
- booking logic involves validation, user data, and payment flows;
- user auth and security rules are separate from business inventory data;
- the frontend should consume a single stable API instead of calling each internal service directly.

By splitting the backend into microservices, the system gains:

- independent deployment
- isolated fault domains
- focused business logic
- easier horizontal scaling
- cleaner maintenance and future growth

## System architecture

```text
                         +----------------------+
                         | Frontend Application |
                         | React / TypeScript   |
                         +----------+-----------+
                                    |
                                    | HTTP requests
                                    v
                         +----------------------+
                         | API Gateway :8080    |
                         | Spring Cloud Gateway |
                         +----------+-----------+
                                    |
                  +-----------------+------------------+
                  |                 |                  |
                  v                 v                  v
            Hotel Service     Flight Service      Booking Service
            /hotels/**        /vols/**          /reservations/**
                  |                 |                  |
                  +-----------------+------------------+
                                    |
                                    v
                         +---------------------+
                         | Eureka Server :8761 |
                         | Service Registry    |
                         +---------------------+
```

## Microservices

### 1. API Gateway

The API Gateway is the single entry point for client requests.

Responsibilities:

- route requests to the correct microservice
- protect the internal architecture from direct client access
- centralize API access and future cross-cutting logic
- simplify frontend integration with a stable URL

Current routing in this project includes:

| Path | Service |
|---|---|
| `/hotels/**` | `hotel-service` |
| `/vols/**` | `vol-service` |
| `/reservations` | `booking-service` |
| `/reservations/**` | `booking-service` |
| `/auth/**` | `booking-service` |

This allows the frontend to communicate with a single gateway instead of having to know each internal service port or hostname.

### 2. Eureka Server

Eureka provides service discovery. Each microservice registers itself with the Eureka server when it starts up.

This means the gateway can call a service by its logical name, such as `lb://hotel-service`, instead of relying on hard-coded URLs.

Benefits:

- no manual service address configuration for each new instance
- easier scaling through multiple service replicas
- dynamic registration and discovery
- improved resilience and flexibility

### 3. Hotel Service

The hotel service manages hotel-related data and business rules.

Typical responsibilities:

- hotels
- room types
- room availability
- hotel filtering and search logic
- REST endpoints for hotel-related queries

This service is built around Spring Boot, Spring Web, JPA, REST repositories, and discovery support.

### 4. Flight Service (`vol-service`)

The flight service manages flight-related data. It exposes flight operations through the gateway and keeps the flight domain separate from the booking and hotel domains.

This separation makes it easier to evolve the travel catalog independently from the reservation system.

### 5. Booking Service

The booking service is the transaction-oriented layer of the system.

It handles:

- user registration
- authentication and login flows
- JWT token generation and validation
- reservation creation and management
- interaction with the hotel and flight services
- business validation around bookings

This service is important because it connects the domain services into a customer-oriented booking flow.

## How the flow works in practice

### Hotel search flow

```text
Frontend -> API Gateway -> Hotel Service -> Database -> JSON Response -> Frontend
```

The frontend requests hotel data from `/hotels`, the gateway resolves and forwards the request to the hotel service, and the service returns the hotel data as JSON.

### Flight search flow

```text
Frontend -> API Gateway -> Flight Service -> Flight data -> Frontend
```

The flight service exposes a specific flight domain endpoint and keeps that logic isolated from the booking logic.

### Reservation creation flow

```text
Frontend -> API Gateway -> Booking Service -> validation + user auth + booking logic -> persistence -> response
```

The booking service validates the request, authenticates the user if needed, creates the reservation, and returns a confirmation or relevant response.

### Authentication flow

```text
Frontend -> /auth/register or /auth/login -> Booking Service -> UserRepository + JWT service -> token response
```

The service creates or validates user credentials and issues a token that can be used for secure requests.

## Technology stack

- Java 21
- Spring Boot 3.x
- Spring Cloud
- Spring Cloud Gateway
- Netflix Eureka
- Spring Data JPA
- Spring Security
- JWT
- OpenFeign
- Spring WebFlux / WebClient
- H2 Database (for development)
- Maven
- Actuator

## Scalability

This project is designed to be scalable in a real distributed environment.

### 1. Independent service scaling

Each service can scale independently based on traffic demand:

- hotel searches may spike during travel periods
- flight availability may receive heavy load at certain times
- booking operations may need extra capacity during promotions or busy seasons

Because the services are independent, you can scale just the one that needs it.

### 2. Service discovery

Eureka allows the gateway to discover service instances dynamically. If more instances of a service are running, the gateway can load-balance traffic without hard-coded IPs or ports.

### 3. Load-balanced routing

The gateway uses names like `lb://hotel-service`, which makes it possible to run multiple instances of the same service behind the gateway.

```text
API Gateway
   |---> hotel-service instance 1
   |---> hotel-service instance 2
   |---> hotel-service instance 3
```

### 4. Independent deployment

A change in the hotel service does not require redeploying the whole system. Each service has its own lifecycle, codebase module, and runtime process.

### 5. Production-ready extension paths

This architecture can be scaled further with:

- Docker and Kubernetes orchestration
- Redis caching for frequent search results
- PostgreSQL or MySQL as production databases
- Kafka or RabbitMQ for async events
- Prometheus and Grafana for metrics
- OpenTelemetry tracing
- Spring Cloud Config for centralized config
- Resilience4j for retries, fallbacks, and circuit breakers
- API rate limiting and gateway security policies

## Project structure

```text
TravelBookingSystem/
├── api-gateway/
│   ├── src/
│   ├── pom.xml
│   └── ...
├── booking-service/
│   ├── src/
│   ├── pom.xml
│   └── ...
├── eureka-server/
│   ├── src/
│   ├── pom.xml
│   └── ...
├── hotel-service/
│   ├── src/
│   ├── pom.xml
│   └── ...
├── vol-service/
│   ├── src/
│   ├── pom.xml
│   └── ...
├── pom.xml
├── mvnw
├── mvnw.cmd
├── .gitignore
└── README.md
```

## Module responsibilities summary

| Module | Responsibility |
|---|---|
| `api-gateway` | routing, entry point, central access |
| `eureka-server` | service discovery |
| `hotel-service` | hotel/catalog management |
| `vol-service` | flight/catalog management |
| `booking-service` | auth, booking, reservation logic |

## Run locally

### Prerequisites

- Java 21+
- Maven 3.9+
- Git

### Clone the repository

```bash
git clone https://github.com/Oussamoux1234/TravelBookingSystem.git
cd TravelBookingSystem
```

### Build the project

```bash
./mvnw clean install
```

### Start services in order

1. Eureka Server
2. Hotel Service
3. Vol Service
4. Booking Service
5. API Gateway

Example:

```bash
./mvnw -pl eureka-server spring-boot:run
./mvnw -pl hotel-service spring-boot:run
./mvnw -pl vol-service spring-boot:run
./mvnw -pl booking-service spring-boot:run
./mvnw -pl api-gateway spring-boot:run
```

## Default endpoints

- API Gateway: `http://localhost:8080`
- Eureka dashboard: `http://localhost:8761`
- Actuator health: `http://localhost:8080/actuator/health`

## Frontend integration

The frontend repository for this project is:

https://github.com/Oussamoux1234/TravelBookingSystem_FrontEnd

The frontend communicates with the backend through the API gateway, rather than directly with individual services. This is one of the main benefits of the microservice setup: the UI remains simple and stable while the backend can evolve behind the scenes.

## Current project notes

This backend is a strong foundation for a production travel platform. It is already structured around real-world microservice principles, and it can be extended with more business features and deployment infrastructure over time.

## License

This project does not currently declare an explicit license. Add a `LICENSE` file before public distribution or commercial use.

## Contributors

- Oussamoux1234

---

This system demonstrates how a travel booking platform can be built using domain-based microservices, service discovery, and a central API gateway to keep the architecture scalable, modular, and maintainable.
