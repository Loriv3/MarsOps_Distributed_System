# MarsOps: Distributed Automation Platform

**Sapienza University of Rome** — *MSc in Engineering in Computer Science & AI (LoAP)*

**MarsOps** is a distributed automation platform designed to guarantee the survival of occupants in a Martian habitat. By integrating heterogeneous IoT devices into a unified, real-time monitoring and automation system, the platform acts as a critical bridge between a fragile habitat's infrastructure and human operators.

It addresses high-latency challenges in Martian environments by evaluating telemetry against operator-defined criteria to trigger autonomous responses—such as activating cooling systems or emergency ventilation—without requiring manual intervention.

---

## 🚀 Core Functionalities

*   **Real-Time Telemetry Dashboard**: Monitor critical environmental parameters (e.g., oxygen, temperature, radiation, air quality) with automatic WebSocket updates, line-chart trends for the current session, and out-of-range visual highlights (red/orange).
*   **Actuator Control**: View real-time ON/OFF states of all actuators and manually toggle them to intervene directly in emergency situations.
*   **Automation Rules Engine**: Define custom "if-then" rules (e.g., *IF greenhouse temperature > 28°C THEN set cooling fan to ON*) via the dashboard. Rules are evaluated in real-time on every incoming broker event.
*   **Smart Anti-Spam Control Loop**: The automation engine utilizes an in-memory `ConcurrentHashMap` to maintain actuator states. If a command would set an actuator to a state it is already in, the command is suppressed to prevent broker flooding.
*   **Persistent Configuration & Audit**: Automation rules and rule-firing history are safely persisted in SQLite. System state survives restarts, and configuration changes generate `[AUTOMATION-AUDIT]` log traces.
*   **Schema-Driven Data Normalization**: The system abstracts away source-specific complexity by converting different ingestion mechanisms (REST polling vs. SSE streams) into a single `UnifiedEvent` schema.

---

## 🏗️ Architecture & Microservices

The platform follows a decoupled, stateless (where possible) microservices architecture, communicating asynchronously via a central message broker.

### 1. Data Ingestion & Normalization
*   **Simulator (`simulator`)**: A black-box component mimicking the Martian habitat. It provides 8 REST sensor endpoints and 4 telemetry SSE streams (power, environment, thermal, airlock).
*   **Converter Service (`converter`)**: A Python background daemon. It polls the REST sensors every 5 seconds, maintains persistent SSE connections with auto-reconnect, and normalizes raw payloads into standard `UnifiedEvent`s published to the message broker.

### 2. Messaging & State Management
*   **ActiveMQ Artemis (`activemq`)**: The messaging backbone. It handles multicast topics for telemetry (`sensor.events`) and actuator states (`actuator.states`), and an anycast queue for actuator commands (`actuator.commands`).
*   **Redis Cache (`redis`)**: An ephemeral, sub-millisecond in-memory store maintaining the latest known state of all sensors and actuators.
*   **Cache Service (`cache_service`)**: A Python FastAPI service that consumes normalized events to update Redis. It decouples the primary database load and powers the frontend's initial state load.

### 3. Business Logic & Automation
*   **Automation Rules Engine (`automation-rules`)**: A Java Spring Boot (Java 21) backend. It consumes telemetry from `sensor.events`, parses `MetricGroupStateEvent`s, matches them against persisted rules, and emits `ActuatorCommand`s. It utilizes SQLite (via JDBC) to store the `rules` and `rule_firings` tables.
*   **Actuator Service (`actuator_service`)**: The API gateway for actuators. It listens to `actuator.commands` from the automation engine or UI, executes REST calls to the simulator, and broadcasts live state changes to the frontend via WebSockets.
*   **Metrics Service (`metrics_service`)**: A FastAPI middleware acting as a WebSocket hub. It broadcasts general and targeted telemetry streams to the dashboard clients.

### 4. Operator Interface
*   **Frontend Dashboard (`frontend`)**: A React SPA served by Nginx. It features distinct views for Groups Overview, Preferred Dashboard Values (with a live system connection indicator), and Actuator Control / Rule Management.

---

## 🛠️ Technology Stack

*   **Backend**: 
    *   Python 3.12 (FastAPI, Uvicorn, `httpx`, `python-qpid-proton`)
    *   Java 21 (Spring Boot 3.3.2, Lombok, Jackson, Spring JMS/JDBC)
*   **Frontend**: React, Nginx
*   **Message Broker**: ActiveMQ Artemis (AMQP 1.0, STOMP)
*   **Data Storage**: Redis 7 (Ephemeral Cache), SQLite (Rule Persistence)
*   **Containerization**: Docker, Docker Compose

---

## ⚙️ Setup and Installation

### Prerequisites
*   [Docker](https://docs.docker.com/get-docker/)
*   [Docker Compose](https://docs.docker.com/compose/install/)

### Running the Platform
The entire platform is containerized. A `docker-compose.yml` is provided in the `source/` directory to orchestrate all microservices.

1.  Clone the repository and navigate to the `source` directory:
    ```bash
    cd source
    ```
2.  Build and start the containers in detached mode:
    ```bash
    docker compose up --build -d
    ```
3.  To shut down the system and remove the containers:
    ```bash
    docker compose down
    ```
    *Note: Automation rules are saved in a persistent Docker volume (`automation_rules_data`) and will survive restarts.*

---

## 🌐 Accessing the Services

Once the containers are running, you can interact with the system via the following mapped ports on your host machine:

| Service | Port | Description |
| :--- | :--- | :--- |
| **Frontend Dashboard** | `http://localhost:8085` | The main React operator UI. |
| **ActiveMQ Console** | `http://localhost:8161` | Message broker admin interface (Credentials: `admin`/`admin`). |
| **Simulator API** | `http://localhost:8080` | Direct access to the simulator REST endpoints. |
| **Cache Service API** | `http://localhost:8081` | REST endpoints for retrieving sensor/actuator snapshots. |
| **Metrics Service WS** | `http://localhost:8082` | Real-time sensor metrics WebSocket stream hub. |
| **Actuator Service API** | `http://localhost:8083` | Actuator toggle control and state WebSocket stream. |
| **Automation Rules API**| `http://localhost:8084` | REST endpoints for CRUD operations on automation rules. |

---

*MarsOps - Ensuring safety and automation in extreme environments. (Mission: MARSOPS_2036)*
