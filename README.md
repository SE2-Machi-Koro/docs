# Machi Koro Project - Docs

## Documentation
- [Project Overview](docs/project-setup.md)
- [Guidelines](docs/guidelines.md)
- [Business-Logic](docs/business-logic.md)
- [Issue Template](docs/.github)

---

## 🏗 System Architecture & Overview

The Machi Koro project is a digital multiplayer adaptation of the board game, built using a decoupled client-server architecture. Real-time, bidirectional communication is achieved via WebSockets using the STOMP protocol.

```text
┌─────────────────┐    WebSocket (STOMP / SockJS)     ┌────────────────────────┐
│  Android Client │ <───────────────────────────────> │   Spring Boot Server   │
│ (Jetpack Compose)│   Port Configuration / JSON      │ (Kotlin / Exposed DSL) │
└─────────────────┘                                   └───────────┬────────────┘
                                                                  │  Exposed DSL
                                                                  ▼
                                                      ┌────────────────────────┐
                                                      │ PostgreSQL Database    │
                                                      └────────────────────────┘
```

### 🖥 Backend Server

The backend acts as the authoritative game logic engine and manages real-time multiplayer sessions, player communication, state validation, and data persistence.

* **Core Stack**: Kotlin 2.2.21, Spring Boot 4.0.3, and PostgreSQL 18.0.
* **Game Logic Engine**: Tracks strict turn phases (Roll Dice, Resolve Effects, Buy/Build, End Turn) and handles win condition checking (e.g., landmark completion).
* **Data Layer (Exposed DSL)**: Uses JetBrains Exposed 1.0.0 exclusively in DSL (Domain-Specific Language) mode for type-safe, explicit SQL query building. Raw database rows (`ResultRow`) are safely isolated and transformed into clean domain models inside the Data Access Objects (DAOs) before reaching the service layer.
* **Database Migrations**: Handled automatically on startup via Flyway to ensure schema synchronization across environments.
* **API & WebSocket Documentation**:
* Interactive REST endpoints documentation is available via **Swagger UI** at `http://localhost:8080/swagger-ui.html`.
* Auto-generated asynchronous channel documentation and event testing are available via **Springwolf UI (AsyncAPI)** at `http://localhost:8080/springwolf/asyncapi-ui.html`.



### 📱 Android Client

The frontend application serves as the reactive user interface that interacts with the backend components.

* **Core Stack**: Kotlin, Jetpack Compose (Material 3) for a modern declarative UI, and Android ViewModel + StateFlow for predictable state management following MVVM/Clean Architecture patterns.
* **Networking**: Powered by OkHttp combined with a custom STOMP client implementation.
* **Centralization**: All WebSocket/STOMP endpoints and contract constants are strictly encapsulated within `WebSocketContract.kt` to ensure system-wide consistency.
* **In-App Features**: Includes specialized navigation layouts (Home, Lobby, Game screens) and a native `PdfViewerScreen` used for rendering official document resources such as game rules directly inside the app.

---

## ⚙️ Environment Configuration & Deployment

### Environment Variables

The application relies on specific environment fields for networking. For local emulator environments, the client is configured with the following defaults to bridge to the server component:

| Property Name | Default Value (Emulator) | Description |
| --- | --- | --- |
| `backendBaseUrl` | `http://10.0.2.2:8080` | The REST API base URL used by the client |
| `websocketUrl` | `ws://10.0.2.2:8080/ws` | The STOMP WebSocket communication endpoint |

### Local Quickstart

#### Running the Server Locally

```bash
./gradlew build
./gradlew bootRun
```



#### Running the Complete Stack with Docker Compose

To execute an end-to-end environment that mirrors the containerized setup, configure a local `.env` file and execute:

```bash
docker compose -f compose.yaml -f compose.local-test.yaml --env-file .env.test up -d --build
```



#### Running the Client

Open the client directory inside Android Studio (Ladybug 2024.2.1+ recommended) or deploy directly via CLI:

```bash
./gradlew installDebug
```



---

## 🧪 Testing & Code Quality

Quality control is strictly enforced across repositories. Both the Client and Server projects are bound to a strict **≥80% Jacoco line coverage quality gate**.

* **Docs Validation**:
```bash
./gradlew test
```



* **Backend Execution (Unit + Integration with Testcontainers)**:
```bash
./gradlew check
./gradlew jacocoTestReport
```



* **Client Execution (Unit + Android Instrumented UI Tests)**:
```bash
./gradlew testDebugUnitTest
./gradlew connectedDebugAndroidTest
```




---

## 🛠 Development Rules & Workflow

### Important Guidelines

* 🚫 **Never push directly to main** — all updates must go through feature branches.
* 👥 **Code Review** — Create Pull Requests for merging all features.
* 🔄 **Continuous Integration** — All GitHub Actions workflows, tests, and SonarCloud checks must pass successfully before a merge is authorized.
* 🟢 **Stability** — Keep the `main` branch completely stable and runnable at all times.

### Git Workflow

```text
pull ──> branch ──> develop ──> push ──> PR ──> merge ──> pull
```



---

## 👥 Team Members

* Danylo Lavrov
* Fabian Barrasch
* Ivona Goranova
* Lea Kandut
* Lea Kutschera
* Lev Starman
* Moritz Gutschi
* Valentina Schiavon

---

## 🎯 Project Goal

Build a fully functioning, digital multiplayer version of Machi Koro featuring:

* Real-time bidirectional event synchronization (WebSocket STOMP).
* Robust backend business logic rules combined with secure data storage (PostgreSQL).
* Interactive, responsive, and mobile UI (Android Jetpack Compose).
* Clean architecture models built under highly protective test coverage (Jacoco ≥80%).

---

*Last Updated: May 16, 2026*
