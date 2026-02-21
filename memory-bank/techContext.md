# Tech Context: Kafbat UI Local Development Setup

## WHY: Development Environment Purpose

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LOCAL DEVELOPMENT ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   You need THREE components running to develop locally:                      │
│                                                                              │
│   ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│   │  1. KAFKA STACK  │     │   2. BACKEND     │     │   3. FRONTEND    │   │
│   │     (Docker)     │     │  (Spring Boot)   │     │     (React)      │   │
│   │                  │     │                  │     │                  │   │
│   │  • Kafka Broker  │◀────│  • REST API      │◀────│  • Web UI        │   │
│   │  • Schema Reg    │     │  • WebSocket     │     │  • Hot Reload    │   │
│   │  • Kafka Connect │     │  • Kafka Client  │     │  • Dev Tools     │   │
│   │                  │     │                  │     │                  │   │
│   │  Ports:          │     │  Port: 8080      │     │  Port: 3000      │   │
│   │  9092, 8085,     │     │                  │     │                  │   │
│   │  8083            │     │                  │     │                  │   │
│   └──────────────────┘     └──────────────────┘     └──────────────────┘   │
│           │                        ▲                        │              │
│           │    Kafka Protocol      │        /api proxy      │              │
│           └────────────────────────┴────────────────────────┘              │
│                                                                              │
│   Browser → http://localhost:3000 (Frontend with Hot Reload)                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## PREREQUISITES

### Required Software Versions

| Software | Required Version | Installation Command | Verify Command |
|----------|------------------|----------------------|----------------|
| Java     | **25+** (JDK)    | See SDKMAN section   | `java --version` |
| Node.js  | **22+**          | `nvm install 22`     | `node --version` |
| pnpm     | **10+**          | `npm install -g pnpm@10` | `pnpm --version` |
| Docker   | Latest           | Docker Desktop       | `docker --version` |
| Git      | Any              | Pre-installed        | `git --version` |

### Installing Prerequisites

#### Java via SDKMAN (CONFIGURED)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CURRENT SDKMAN JAVA SETUP                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Installed Versions:                                                        │
│   ┌─────────────┬──────────┬─────────────────┬────────────────────────────┐ │
│   │ Version     │ Vendor   │ Identifier      │ Role                       │ │
│   ├─────────────┼──────────┼─────────────────┼────────────────────────────┤ │
│   │ 11.0.27     │ Temurin  │ 11.0.27-tem     │ Legacy projects            │ │
│   │ 17.0.15     │ Temurin  │ 17.0.15-tem     │ Spring Boot 2.x projects   │ │
│   │ 21.0.10     │ Temurin  │ 21.0.10-tem     │ SYSTEM DEFAULT             │ │
│   │ 25.0.2      │ Zulu     │ 25.0.2-zulu     │ kafka-ui (matches Docker)  │ │
│   └─────────────┴──────────┴─────────────────┴────────────────────────────┘ │
│                                                                              │
│   Auto-switching: ENABLED (sdkman_auto_env=true)                            │
│   kafka-ui: Auto-switches to Java 25 (reads .java-version)                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# SDKMAN Commands Reference
# ─────────────────────────

# List installed versions
sdk list java | grep installed

# Switch temporarily (this terminal only)
sdk use java 25.0.2-zulu

# Set system default (all new terminals)
sdk default java 21.0.10-tem

# Install a new version
sdk install java <version>-<vendor>

# Check current version
sdk current java

# Verify auto-switch works
cd ~/Ruckus/Kafka/kafka-ui && java -version   # → 25.0.2
cd ~ && java -version                          # → 21.0.10
```

```bash
# Installing SDKMAN (if starting fresh)
# ─────────────────────────────────────

# Step 1: Install SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# Step 2: Install Java versions
sdk install java 25.0.2-zulu   # For kafka-ui (matches Dockerfile)
sdk install java 21.0.10-tem   # LTS, good default
sdk install java 17.0.15-tem   # Spring Boot 2.x
sdk install java 11.0.27-tem   # Legacy

# Step 3: Set default
sdk default java 21.0.10-tem

# Step 4: Enable auto-env
sed -i '' 's/sdkman_auto_env=false/sdkman_auto_env=true/' ~/.sdkman/etc/config
```

#### Node.js 22 via NVM

```bash
# WHY: NVM allows switching between Node versions per project
# WHAT: Install Node 22

# Step 1: Install NVM (if not installed)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Step 2: Install Node 22
nvm install 22

# Step 3: Use Node 22 in this project
nvm use 22

# Step 4: Create .nvmrc for automatic switching
echo "22" > .nvmrc

# HOW TO VERIFY:
node --version
# Expected: v22.x.x
```

#### pnpm 10

```bash
# WHY: pnpm is faster than npm and required by this project
# WHAT: Install pnpm globally

npm install -g pnpm@10

# HOW TO VERIFY:
pnpm --version
# Expected: 10.x.x
```

---

## STEP-BY-STEP: Running Locally

### Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DEVELOPMENT WORKFLOW (3 Terminals)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   TERMINAL 1                TERMINAL 2                TERMINAL 3            │
│   ──────────               ──────────                ──────────             │
│                                                                              │
│   ┌─────────────┐          ┌─────────────┐          ┌─────────────┐        │
│   │  Docker     │          │  Gradle     │          │  pnpm       │        │
│   │  Compose    │          │  bootRun    │          │  dev        │        │
│   │             │          │             │          │             │        │
│   │  Starts:    │          │  Starts:    │          │  Starts:    │        │
│   │  • Kafka    │   ───▶   │  • Spring   │   ◀───   │  • Vite     │        │
│   │  • Schema   │          │    Boot     │          │  • React    │        │
│   │    Registry │          │    API      │          │    HMR      │        │
│   └─────────────┘          └─────────────┘          └─────────────┘        │
│        │                         │                        │                 │
│        ▼                         ▼                        ▼                 │
│   localhost:9092           localhost:8080           localhost:3000         │
│   localhost:8085                                                            │
│                                                                              │
│   START ORDER: Terminal 1 → Terminal 2 → Terminal 3                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### STEP 1: Start Kafka Infrastructure (Terminal 1)

```bash
# ┌─────────────────────────────────────────────────────────────────────────┐
# │ WHY: The backend needs Kafka to connect to. Without Kafka, the         │
# │      backend will fail to start or show connection errors.             │
# └─────────────────────────────────────────────────────────────────────────┘

# Navigate to compose directory
cd /Users/parasuram/Ruckus/Kafka/kafka-ui/documentation/compose

# Start Kafka + Schema Registry + Connect
docker compose -f kafbat-ui.yaml up kafka0 schemaregistry0 kafka-connect0 -d

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ WHAT THIS DOES:                                                         │
# │                                                                          │
# │  • kafka0 (Port 9092)                                                   │
# │    - Single Kafka broker in KRaft mode (no Zookeeper)                   │
# │    - JMX metrics on port 9997                                           │
# │                                                                          │
# │  • schemaregistry0 (Port 8085)                                          │
# │    - Confluent Schema Registry                                          │
# │    - Stores Avro/Protobuf/JSON schemas                                  │
# │                                                                          │
# │  • kafka-connect0 (Port 8083)                                           │
# │    - Kafka Connect worker                                               │
# │    - For connector management                                           │
# └─────────────────────────────────────────────────────────────────────────┘

# HOW TO VERIFY:
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Expected output:
# NAMES              STATUS          PORTS
# kafka0             Up X seconds    0.0.0.0:9092->9092/tcp, ...
# schemaregistry0    Up X seconds    0.0.0.0:8085->8085/tcp
# kafka-connect0     Up X seconds    0.0.0.0:8083->8083/tcp

# Test Kafka is responding:
docker exec kafka0 kafka-topics --bootstrap-server localhost:29092 --list
```

#### Troubleshooting Step 1

| Issue | Cause | Solution |
|-------|-------|----------|
| Port 9092 in use | Another Kafka instance | `docker ps -a` and stop conflicting container |
| Container exits immediately | Missing dependencies | Check logs: `docker logs kafka0` |
| Schema Registry won't start | Kafka not ready | Wait 30s, Schema Registry retries automatically |

---

### STEP 2: Start Backend (Terminal 2)

```bash
# ┌─────────────────────────────────────────────────────────────────────────┐
# │ WHY: The backend serves the REST API that the frontend consumes.        │
# │      It connects to Kafka and exposes cluster management endpoints.     │
# └─────────────────────────────────────────────────────────────────────────┘

# Navigate to project root
cd /Users/parasuram/Ruckus/Kafka/kafka-ui

# Start Spring Boot with local profile
./gradlew :api:bootRun --args='--spring.profiles.active=local'

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ WHAT THIS DOES:                                                         │
# │                                                                          │
# │  • ./gradlew                                                            │
# │    - Gradle wrapper (no need to install Gradle globally)                │
# │                                                                          │
# │  • :api:bootRun                                                         │
# │    - Runs the 'api' subproject (the Spring Boot app)                    │
# │    - Uses Spring Boot's bootRun task                                    │
# │                                                                          │
# │  • --args='--spring.profiles.active=local'                              │
# │    - Activates application-local.yml configuration                      │
# │    - Connects to localhost:9092 (Kafka)                                 │
# │    - Connects to localhost:8085 (Schema Registry)                       │
# │    - Disables authentication (for dev convenience)                      │
# │                                                                          │
# │  The server starts on: http://localhost:8080                            │
# └─────────────────────────────────────────────────────────────────────────┘

# HOW TO VERIFY:
# Wait for: "Started KafkaUiApplication in X seconds"
# Then test the API:
curl http://localhost:8080/api/clusters

# Expected: JSON array with cluster info
# [{"name":"local","bootstrapServers":"localhost:9092",...}]
```

#### Alternative: Run from IDE (IntelliJ IDEA)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INTELLIJ IDEA CONFIGURATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Open project: File → Open → select kafka-ui folder                      │
│                                                                              │
│  2. Wait for Gradle import to complete                                       │
│                                                                              │
│  3. Create Run Configuration:                                                │
│     • Run → Edit Configurations → + → Spring Boot                           │
│     • Main class: io.kafbat.ui.KafkaUiApplication                           │
│     • Active profiles: local                                                 │
│     • Module: api.main                                                       │
│                                                                              │
│  4. Click Run (green arrow)                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Troubleshooting Step 2

| Issue | Cause | Solution |
|-------|-------|----------|
| `Unsupported class file major version 69` | Java version mismatch | Install Java 25: `sdk install java 25-open` |
| `Connection refused: localhost:9092` | Kafka not running | Complete Step 1 first |
| Port 8080 in use | Another app | `lsof -i :8080` and kill process |
| Slow first build | Downloading dependencies | Wait ~5min, subsequent builds are faster |

---

### STEP 3: Start Frontend (Terminal 3)

```bash
# ┌─────────────────────────────────────────────────────────────────────────┐
# │ WHY: The frontend provides the React UI with hot module replacement.    │
# │      Changes to React code instantly appear in the browser.             │
# └─────────────────────────────────────────────────────────────────────────┘

# Navigate to frontend directory
cd /Users/parasuram/Ruckus/Kafka/kafka-ui/frontend

# Install dependencies (first time only, or after package.json changes)
pnpm install

# Start development server with proxy to backend
VITE_DEV_PROXY=http://localhost:8080 pnpm dev

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ WHAT THIS DOES:                                                         │
# │                                                                          │
# │  • pnpm install                                                         │
# │    - Installs all npm dependencies                                      │
# │    - Creates node_modules/                                              │
# │    - Uses pnpm's efficient storage                                      │
# │                                                                          │
# │  • VITE_DEV_PROXY=http://localhost:8080                                 │
# │    - Environment variable read by vite.config.ts                        │
# │    - Tells Vite to proxy /api/* requests to backend                     │
# │                                                                          │
# │  • pnpm dev                                                             │
# │    - Starts Vite development server                                     │
# │    - Enables Hot Module Replacement (HMR)                               │
# │    - TypeScript checking in overlay                                     │
# │    - ESLint checking in overlay                                         │
# │                                                                          │
# │  The UI starts on: http://localhost:3000                                │
# └─────────────────────────────────────────────────────────────────────────┘

# HOW TO VERIFY:
# Vite outputs:
#   VITE v6.4.1  ready in XXX ms
#
#   ➜  Local:   http://localhost:3000/
#   ➜  Network: use --host to expose
#
# Open http://localhost:3000 in browser - you should see the Kafbat UI
```

#### Troubleshooting Step 3

| Issue | Cause | Solution |
|-------|-------|----------|
| `ERR_PNPM_UNSUPPORTED_ENGINE` | Node version too old | `nvm use 22` or install Node 22 |
| API calls fail (CORS) | Missing proxy config | Ensure `VITE_DEV_PROXY` is set |
| `Module not found` | Incomplete install | Delete `node_modules` and `pnpm install` again |
| Port 3000 in use | Another dev server | `lsof -i :3000` and kill process |

---

## QUICK REFERENCE: All Commands

### Start Everything (3 terminals)

```bash
# Terminal 1: Kafka Stack
cd /Users/parasuram/Ruckus/Kafka/kafka-ui/documentation/compose
docker compose -f kafbat-ui.yaml up kafka0 schemaregistry0 kafka-connect0 -d

# Terminal 2: Backend
cd /Users/parasuram/Ruckus/Kafka/kafka-ui
./gradlew :api:bootRun --args='--spring.profiles.active=local'

# Terminal 3: Frontend
cd /Users/parasuram/Ruckus/Kafka/kafka-ui/frontend
pnpm install && VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

### Stop Everything

```bash
# Stop Frontend: Ctrl+C in Terminal 3

# Stop Backend: Ctrl+C in Terminal 2

# Stop Kafka Stack:
cd /Users/parasuram/Ruckus/Kafka/kafka-ui/documentation/compose
docker compose -f kafbat-ui.yaml down
```

### Clean Restart

```bash
# Nuclear option: remove all containers and volumes
docker compose -f kafbat-ui.yaml down -v

# Clean Gradle build
./gradlew clean

# Clean frontend
cd frontend && rm -rf node_modules && pnpm install
```

---

## PORT REFERENCE

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PORT MAPPING                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Port      Service              Purpose                                     │
│   ────      ───────              ───────                                     │
│   3000      Vite Dev Server      Frontend (development only)                │
│   8080      Spring Boot          Backend API                                 │
│   9092      Kafka Broker         Client connections                          │
│   9997      Kafka JMX            Metrics collection                          │
│   8085      Schema Registry      Schema management                           │
│   8083      Kafka Connect        Connector management                        │
│   9090      Prometheus           Metrics storage (optional)                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## CONFIGURATION FILES

| File | Purpose |
|------|---------|
| `api/src/main/resources/application-local.yml` | Local dev config (Kafka URLs, disabled auth) |
| `frontend/vite.config.ts` | Frontend dev server, proxy config |
| `documentation/compose/kafbat-ui.yaml` | Docker Compose for full stack |
| `build.gradle` | Root Gradle config |
| `api/build.gradle` | Backend dependencies and build |
| `frontend/package.json` | Frontend dependencies and scripts |

---

## DATA FLOW IN DEVELOPMENT

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      REQUEST FLOW (Development Mode)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Browser (http://localhost:3000)                                            │
│     │                                                                        │
│     │ GET /                              (React App)                        │
│     ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │  Vite Dev Server (:3000)                                     │            │
│  │                                                               │            │
│  │  • Serves React app                                          │            │
│  │  • Hot Module Replacement                                    │            │
│  │  • TypeScript compilation                                    │            │
│  │  • Proxies /api/* to :8080                                  │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                             │                                                │
│                             │ GET /api/clusters (proxied)                   │
│                             ▼                                                │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │  Spring Boot API (:8080)                                     │            │
│  │                                                               │            │
│  │  • REST controllers                                          │            │
│  │  • Kafka Admin Client                                        │            │
│  │  • Schema Registry Client                                    │            │
│  │  • WebSocket for live updates                                │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                             │                                                │
│                             │ Kafka Protocol                                │
│                             ▼                                                │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │  Kafka Broker (:9092)                                        │            │
│  │                                                               │            │
│  │  • Topic management                                          │            │
│  │  • Message storage                                           │            │
│  │  • Consumer groups                                           │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## CURRENT ENVIRONMENT STATUS

| Component | Your Version | Required | Action Needed |
|-----------|--------------|----------|---------------|
| Java | 21.0.8 | 25+ | `sdk install java 25-open` |
| Node.js | 20.19.4 | 22+ | `nvm install 22 && nvm use 22` |
| pnpm | 10.4.1 | 10+ | ✅ OK |
| Docker | 29.1.4 | Any | ✅ OK |
| Docker Compose | 5.0.1 | Any | ✅ OK |

---

## NEXT STEPS

1. **Upgrade Java to 25** (required for backend)
2. **Upgrade Node.js to 22** (required for frontend)
3. Run the three-terminal setup
4. Open http://localhost:3000
5. You should see the Kafbat UI connected to your local Kafka cluster
