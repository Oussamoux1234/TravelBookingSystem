# Travel Booking System

### A modular Spring Cloud platform for hotels, flights, authentication, reservations, and payments

[![Java](https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white)](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-Microservices-6DB33F?logo=spring&logoColor=white)](https://spring.io/projects/spring-cloud)
[![Maven](https://img.shields.io/badge/Maven-Build-C71A36?logo=apachemaven&logoColor=white)](https://maven.apache.org/)
[![JWT](https://img.shields.io/badge/JWT-Security-000000?logo=jsonwebtokens&logoColor=white)](https://jwt.io/)
[![Eureka](https://img.shields.io/badge/Discovery-Eureka-6DB33F?logo=spring&logoColor=white)](https://spring.io/projects/spring-cloud-netflix)

> A domain-driven travel booking backend where independent Spring Boot services collaborate through service discovery and a centralized API gateway.

## What this project is

This repository implements the backend of a travel booking platform. It separates the application into dedicated services for:

- hotel and room inventory;
- flight inventory;
- user registration and authentication;
- reservation creation and ownership;
- payment simulation;
- request routing and service discovery.

The system is designed so that a frontend communicates with one stable gateway URL while internal services remain independently deployable and discoverable.

## Connected frontend

The companion frontend is available here:

**[TravelBookingSystem_FrontEnd](https://github.com/Oussamoux1234/TravelBookingSystem_FrontEnd)**

```text
Frontend
   |
   | HTTP + Authorization: Bearer <JWT>
   v
API Gateway :8080
   |
   +--> Hotel Service :8081
   +--> Flight Service :8082
   `--> Booking Service :8083
            |
            +--> User and reservation database
            +--> Hotel Service through OpenFeign
            `--> Flight Service through WebClient
```

## Repository architecture

```text
TravelBookingSystem/
├── api-gateway/       # Public entry point and route forwarding
├── booking-service/   # Auth, JWT, reservations, payment simulation
├── eureka-server/     # Service registry on port 8761
├── hotel-service/     # Hotels, rooms, and availability
├── vol-service/       # Flights and available seats
├── src/               # Root Spring Boot module scaffold
├── pom.xml            # Maven reactor and dependency management
├── mvnw               # Maven Wrapper for Linux/macOS
└── mvnw.cmd           # Maven Wrapper for Windows
```

The deployable business services are the modules under `api-gateway`, `booking-service`, `eureka-server`, `hotel-service`, and `vol-service`. The root `pom.xml` is the Maven parent and reactor project; it also includes a root application scaffold under `src`.

## Runtime architecture

```text
                                      +------------------+
                                      | Web / Mobile UI  |
                                      +--------+---------+
                                               |
                                               | REST + JWT
                                               v
                                      +------------------+
                                      | API Gateway      |
                                      | localhost:8080   |
                                      +--------+---------+
                                               |
                         +---------------------+----------------------+
                         |                     |                      |
                         v                     v                      v
                 /hotels/**              /vols/**             /reservations/**
                         |                     |                      |
                         v                     v                      v
                 +---------------+     +---------------+     +----------------+
                 | hotel-service  |     | vol-service   |     | booking-service|
                 | :8081          |     | :8082         |     | :8083          |
                 +-------+-------+     +-------+-------+     +--------+-------+
                         |                     |                       |
                         v                     v                       +--> H2 bookingdb
                 +---------------+     +---------------+               |
                 | Hotels / Rooms|     | Flights / Seats|               +--> JWT security
                 +---------------+     +---------------+               +--> payment simulation
                                                                       |
                                                                       +--> OpenFeign -> hotel-service
                                                                       +--> WebClient -> vol-service

                                      +------------------+
                                      | Eureka Server    |
                                      | localhost:8761   |
                                      +------------------+
```

## Service discovery with Eureka

The Eureka server runs as a standalone registry on port `8761` and is configured not to register itself or fetch another registry.

Each client service registers with Eureka:

```text
hotel-service  --registers as--> HOTEL-SERVICE
vol-service    --registers as--> VOL-SERVICE
booking-service--registers as--> BOOKING-SERVICE
api-gateway    --discovers-----> those service names
```

The gateway uses logical load-balanced destinations such as:

```yaml
uri: lb://hotel-service
```

This prevents the frontend from depending on internal hostnames or instance ports. If several instances of a service are registered, the gateway can route traffic across them.

## API Gateway routes

The gateway listens on port `8080` and currently defines these routes:

| Route | Destination | Purpose |
|---|---|---|
| `/hotels/**` | `hotel-service` | Hotel details and availability |
| `/vols/**` | `vol-service` | Flight data |
| `/reservations` | `booking-service` | Create and test reservation access |
| `/reservations/**` | `booking-service` | User reservations and reservation details |
| `/auth/**` | `booking-service` | Registration, login, and current-user access |

The gateway also exposes Actuator endpoints through its management configuration. Review that exposure before production deployment.

## Hotel service

### Domain model

The hotel service uses two JPA entities:

```text
Hotel
  ├── id
  ├── nom
  ├── adresse
  ├── ville
  └── chambres: List<Chambre>

Chambre
  ├── id
  ├── numero
  ├── type: SIMPLE | DOUBLE | SUITE
  ├── prixParNuit
  ├── disponible
  └── hotel
```

`Hotel` owns a lazy, cascading one-to-many relationship with `Chambre`. The room's reverse `hotel` relationship is ignored during JSON serialization to avoid recursive output.

### Endpoints implemented by `HotelController`

| Method | Endpoint | Behavior |
|---|---|---|
| `GET` | `/hotels/{id}` | Returns a hotel or `null` when not found |
| `GET` | `/hotels/{id}/disponibilite/{type}` | Checks whether a room of the requested type is available |

Availability is checked by loading the hotel and searching its rooms for a matching `TypeChambre` and `disponible = true`.

### Development data

`HotelDataInitializer` inserts ten sample hotels and rooms at startup, including examples in Chamonix, New York, Nice, Paris, Marrakech, Kyoto, and Zermatt. The prices and availability are development seed data, not an external inventory integration.

## Flight service

The `vol-service` uses the `Vol` JPA entity:

```text
Vol
  ├── id
  ├── numeroVol
  ├── aeroportDepart
  ├── aeroportArrivee
  ├── heureDepart
  ├── dateHeureDepart
  ├── prix
  └── placesDisponibles
```

`VolDataInitializer` inserts ten sample routes, including Paris–New York, Tokyo–Seoul, London–Singapore, Dubai–Paris, and Miami–Cancun. The data is generated relative to application startup with `LocalDateTime.now()`.

The gateway exposes the flight domain through `/vols/**`.

## Booking service

The booking service owns users and reservations and runs on port `8083` with an in-memory H2 database named `bookingdb`.

### Reservation model

A reservation contains:

- reservation timestamp;
- client email;
- reservation status;
- optional flight ID;
- optional hotel ID;
- total amount;
- simulated payment transaction ID;
- passenger name;
- room type;
- seat number;
- flight class;
- hotel check-in and check-out dates;
- owning authenticated user.

Reservation statuses are represented by the `StatutReservation` enum. The booking workflow starts at `EN_ATTENTE_PAIEMENT`, then becomes `CONFIRMEE` after simulated payment succeeds, or `ANNULEE` when payment fails.

## Authentication and JWT flow

Authentication is implemented inside `booking-service`.

### Registration

```text
POST /auth/register
        |
        v
Validate RegisterRequest
        |
        v
Check UserRepository.existsByEmail()
        |
        v
Hash password with BCrypt
        |
        v
Persist User
        |
        v
Generate JWT
        |
        v
Return AuthResponse
```

### Login

```text
POST /auth/login
        |
        v
AuthenticationManager
        |
        v
DaoAuthenticationProvider
        |
        v
UserDetailsService -> UserRepository.findByEmail()
        |
        v
Validate BCrypt password
        |
        v
JwtService.generateToken()
        |
        v
Return token, user ID, email, and name
```

### Current user

```text
GET /auth/me
Authorization: Bearer <JWT>
        |
        v
JwtAuthenticationFilter
        |
        v
JwtService extracts email and validates expiration
        |
        v
UserRepository loads authenticated User
        |
        v
AuthController returns UserDTO
```

### Security rules

`SecurityConfig` uses stateless sessions and a custom `JwtAuthenticationFilter`.

Public routes:

- `/auth/**`;
- `/h2-console/**`;
- `/actuator/**`;
- `/hotels/**`;
- `/vols/**`;
- preflight `OPTIONS` requests.

Protected routes:

- `/reservations/**`;
- any other route not explicitly permitted.

Passwords are encoded with `BCryptPasswordEncoder`. JWT configuration currently comes from `booking-service/src/main/resources/application.yml`.

> For production, move the JWT secret and database credentials into environment variables or a secrets manager. The repository currently contains a development secret in configuration.

## Reservation workflow in detail

`BookingController` coordinates the reservation process:

```text
POST /reservations
        |
        v
Read authenticated User from SecurityContext
        |
        +--> If volId exists:
        |       WebClient -> http://VOL-SERVICE/vols/{id}
        |       Check placesDisponibles
        |       Add flight price to total
        |
        +--> If hotelId exists:
        |       OpenFeign -> HOTEL-SERVICE /hotels/{id}
        |       OpenFeign -> /hotels/{id}/disponibilite/{type}
        |       Add hotel price to total
        |
        +--> Reject if neither item is available
        |
        +--> Build Reservation entity
        +--> Set ownership to authenticated User
        +--> Save initial reservation
        |
        +--> PaiementService.payer(total)
        |       |
        |       +--> success: transaction ID + CONFIRMEE
        |       `--> failure: ANNULEE + HTTP 500
        |
        `--> Return saved reservation
```

A reservation can contain a flight, a hotel, or both. The total is calculated by adding the selected flight price and hotel nightly price when each item is available.

### Reservation endpoints

| Method | Endpoint | Authentication | Description |
|---|---|---:|---|
| `GET` | `/reservations` | Yes | Returns a service health message |
| `GET` | `/reservations/my` | Yes | Returns reservations owned by the current user |
| `GET` | `/reservations/{id}` | Yes | Returns one reservation after ownership validation |
| `POST` | `/reservations` | Yes | Creates and pays for a reservation |

The ownership check prevents one authenticated user from reading another user's reservation.

## Service-to-service communication

The booking service uses two communication styles:

### OpenFeign to hotel service

`HotelRestClient` is declared with:

```java
@FeignClient(name = "HOTEL-SERVICE")
```

It calls:

```text
GET /hotels/{id}
GET /hotels/{id}/disponibilite/{type}
```

Eureka resolves `HOTEL-SERVICE` to a registered instance.

### WebClient to flight service

The booking controller builds a WebClient with:

```text
http://VOL-SERVICE
```

It calls:

```text
GET /vols/{id}
```

The service name is resolved through the Spring Cloud discovery/load-balancing setup.

## Payment behavior

`PaiementService` currently simulates payment:

```text
amount > 0
    -> return TXN-{timestamp}
    -> reservation becomes CONFIRMEE

amount <= 0 or exception
    -> reservation becomes ANNULEE
```

It is intentionally a placeholder for a future payment provider integration. A production implementation should use an external payment service, idempotency keys, transaction state reconciliation, and secure webhook handling.

## Configuration and ports

| Component | Port | Configuration |
|---|---:|---|
| Eureka Server | `8761` | `eureka-server/src/main/resources/application.yml` |
| API Gateway | `8080` | `api-gateway/src/main/resources/application.yml` |
| Hotel Service | `8081` | module application configuration |
| Flight Service | `8082` | module application configuration |
| Booking Service | `8083` | `booking-service/src/main/resources/application.yml` |

Booking service development configuration includes:

- H2 in-memory datasource `jdbc:h2:mem:bookingdb`;
- Hibernate `create-drop` schema behavior;
- Eureka URL `http://localhost:8761/eureka/`;
- JWT expiration of 24 hours.

## Technology stack

- Java 21
- Spring Boot 3.x
- Spring Cloud Gateway
- Spring Cloud Netflix Eureka
- Spring Data JPA
- Spring Data REST
- Spring Security
- BCrypt
- JSON Web Tokens with JJWT
- OpenFeign
- Spring WebFlux / WebClient
- H2
- Lombok
- Maven
- Spring Boot Actuator

## Build and run

### Prerequisites

- Java 21 or newer;
- Maven 3.9+ or the included Maven Wrapper;
- Git.

### Clone

```bash
git clone https://github.com/Oussamoux1234/TravelBookingSystem.git
cd TravelBookingSystem
```

### Build all modules

```bash
./mvnw clean install
```

Windows:

```powershell
mvnw.cmd clean install
```

### Start the platform

Start services in this order:

```bash
./mvnw -pl eureka-server spring-boot:run
./mvnw -pl hotel-service spring-boot:run
./mvnw -pl vol-service spring-boot:run
./mvnw -pl booking-service spring-boot:run
./mvnw -pl api-gateway spring-boot:run
```

Run each command in a separate terminal. The frontend should use the gateway URL, not the internal service URLs.

## Useful URLs

- Eureka dashboard: `http://localhost:8761`
- API gateway: `http://localhost:8080`
- Gateway health: `http://localhost:8080/actuator/health`
- H2 console, if enabled: the booking-service H2 console endpoint

## Scalability and production roadmap

The current modular architecture is a foundation for horizontal scaling:

```text
                         +--> Hotel instance 1
API Gateway + Eureka ----+--> Hotel instance 2
                         `--> Hotel instance 3
```

Recommended production improvements:

1. Replace H2 with persistent PostgreSQL or MySQL databases.
2. Use a database per service to preserve service ownership.
3. Move secrets and configuration out of Git.
4. Add Flyway or Liquibase migrations.
5. Add Resilience4j timeouts, retries, circuit breakers, and fallbacks.
6. Add Redis caching for hotel and flight search.
7. Replace payment simulation with a real payment service.
8. Add an event broker for reservation-created and payment-completed events.
9. Add Docker Compose for local orchestration.
10. Add Kubernetes manifests or Helm charts for deployment.
11. Add centralized logs, metrics, and distributed tracing.
12. Add contract, integration, and end-to-end tests.
13. Add API rate limiting and stricter gateway security.
14. Add pagination and filtering to catalog endpoints.
15. Make seed data conditional so it does not run in production.

## Current implementation notes

- The application uses development seed data for hotels and flights.
- Hotel availability is currently a simple in-memory/entity lookup over room records.
- Payment is simulated by `PaiementService`.
- Booking service uses H2 with `create-drop`, so data is not persistent between restarts.
- The JWT secret is currently present in development configuration and should be externalized.
- Some service endpoint behavior should be covered with integration tests before production deployment.

## License

No open-source license is currently declared. Add a `LICENSE` file before public distribution or commercial use.

## Related project

[TravelBookingSystem Frontend](https://github.com/Oussamoux1234/TravelBookingSystem_FrontEnd)

## Contributors

- Oussamoux1234
