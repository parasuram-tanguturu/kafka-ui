# Local Development Guide

> Complete guide to running Kafbat UI locally for development and contribution.

---

## Architecture Overview

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

## Prerequisites

### Required Versions

| Software | Required Version | Verify Command |
|----------|------------------|----------------|
| Java     | **25+** (JDK)    | `java --version` |
| Node.js  | **22+**          | `node --version` |
| pnpm     | **10+**          | `pnpm --version` |
| Docker   | Latest           | `docker --version` |

### Installing Java 25 (via SDKMAN)

```bash
# Install SDKMAN (if not installed)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# Install Java 25
sdk install java 25-open
sdk default java 25-open

# Verify
java --version
# Expected: openjdk 25.x.x
```

### Installing Node.js 22 (via NVM)

```bash
# Install NVM (if not installed)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Install and use Node 22
nvm install 22
nvm use 22

# Create .nvmrc for auto-switching
echo "22" > .nvmrc

# Verify
node --version
# Expected: v22.x.x
```

### Installing pnpm

```bash
npm install -g pnpm@10

# Verify
pnpm --version
# Expected: 10.x.x
```

---

## ⚠️ Caution: Port Conflicts

### Understanding Docker Port Mapping

Docker exposes container services to your host machine via **port mapping**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DOCKER PORT MAPPING SYNTAX                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ports:                                                            │
│     - "8080:8080"                                                   │
│        ─────┬─────                                                  │
│        │    │                                                       │
│        │    └── CONTAINER PORT (inside Docker)                      │
│        │                                                            │
│        └─────── HOST PORT (your machine, accessible via localhost)  │
│                                                                     │
│   Example: "9092:9092" means:                                       │
│   • Container listens on port 9092 internally                       │
│   • Host can access it via localhost:9092                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Common Port Conflicts

> **Before starting development**, check that these ports are FREE on your machine.

| Port | Used By (This Project) | Common Conflicts |
|------|------------------------|------------------|
| **3000** | Frontend (Vite) | React apps, Node.js servers, Grafana |
| **8080** | Backend (Spring Boot) | Tomcat, Jenkins, other Spring apps, nginx |
| **9092** | Kafka Broker | Homebrew Kafka, other Kafka instances |
| **8085** | Schema Registry | Custom services |
| **8083** | Kafka Connect | Custom services |
| **2181** | (Legacy Zookeeper) | Homebrew Zookeeper |

### Check All Ports at Once

```bash
# Check if any required ports are in use
echo "Checking ports..." && \
for port in 3000 8080 9092 8085 8083 2181; do
  result=$(lsof -i :$port 2>/dev/null | head -2)
  if [ -n "$result" ]; then
    echo "⚠️  Port $port IN USE:"
    echo "$result"
  else
    echo "✅ Port $port is free"
  fi
done
```

### Port 3000 Conflict (Frontend)

Port 3000 is the default for many frontend frameworks. If something else is using it:

```bash
# Find what's using port 3000
lsof -i :3000

# Example output:
# COMMAND   PID  USER   FD   TYPE  DEVICE  NODE NAME
# node     5678  user   22u  IPv4  0x...   TCP *:hbci (LISTEN)

# Option 1: Stop the conflicting service
kill -9 <PID>

# Option 2: Run frontend on a different port
cd frontend
VITE_DEV_PROXY=http://localhost:8080 pnpm dev --port 3001

# Then access UI at http://localhost:3001
```

**Common port 3000 culprits:**

- Create React App (`react-scripts start`)
- Next.js dev server
- Other Vite projects
- Grafana
- Express.js / Node.js servers
- Ruby on Rails (default dev port)

---

### Port 8080 Conflict (Backend)

Port 8080 is heavily used. If something else is using it:

```bash
# Find what's using port 8080
lsof -i :8080

# Example output:
# COMMAND   PID  USER   FD   TYPE  DEVICE  NODE NAME
# java     1234  user   42u  IPv6  0x...   TCP *:http-alt (LISTEN)

# Option 1: Stop the conflicting service
kill -9 <PID>

# Option 2: Run backend on a different port
./gradlew :api:bootRun --args='--spring.profiles.active=local --server.port=8081'

# Then update frontend proxy:
VITE_DEV_PROXY=http://localhost:8081 pnpm dev
```

### Homebrew Kafka/Zookeeper Conflict

> **If you have Kafka or Zookeeper installed via Homebrew**, ensure they are **stopped** before starting the Docker infrastructure.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PORT CONFLICT SCENARIO                          │
├─────────────────────────────────────────────────────────────────────┤
│   HOMEBREW KAFKA         vs.         DOCKER KAFKA                   │
│   ─────────────────                  ─────────────                  │
│   Port 9092 (Broker)                 Port 9092 (kafka0)             │
│   Port 2181 (Zookeeper)              Port 9093 (kafka1)             │
│                                                                     │
│   ⚠️ Two processes CANNOT bind the same port!                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Check and stop Homebrew services:**

```bash
# Check if Homebrew Kafka/Zookeeper are running
brew services list | grep -E "kafka|zookeeper"

# Stop them if running
brew services stop kafka
brew services stop zookeeper

# Verify ports are free
lsof -i :9092  # Should return nothing
lsof -i :2181  # Should return nothing
```

**Why Docker Kafka is recommended for this project:**

- Uses **KRaft mode** (no Zookeeper dependency)
- Matches production configuration
- Includes pre-configured Schema Registry and Kafka Connect

---

## Quick Start (3 Terminals)

### Terminal 1: Kafka Infrastructure

```bash
cd documentation/compose
# Start the main Kafka infrastructure services (Kafka broker, Schema Registry, Kafka Connect) in detached mode
docker compose -f kafbat-ui.yaml up kafka0 schemaregistry0 kafka-connect0 -d
```

### Terminal 2: Backend

```bash
./gradlew :api:bootRun --args='--spring.profiles.active=local'
```

### Terminal 3: Frontend

```bash
cd frontend
pnpm install
VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

### Open Browser

```
http://localhost:3000
```

---

## Step-by-Step Guide

### Step 1: Start Kafka Infrastructure

```bash
# Navigate to compose directory
cd documentation/compose

# Start Kafka + Schema Registry + Kafka Connect
docker compose -f kafbat-ui.yaml up kafka0 schemaregistry0 kafka-connect0 -d
```

**What this starts:**

| Service | Port | Purpose |
|---------|------|---------|
| kafka0 | 9092 | Kafka broker (KRaft mode) |
| schemaregistry0 | 8085 | Confluent Schema Registry |
| kafka-connect0 | 8083 | Kafka Connect worker |

**Verify:**

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Expected:
# NAMES              STATUS          PORTS
# kafka0             Up X seconds    0.0.0.0:9092->9092/tcp
# schemaregistry0    Up X seconds    0.0.0.0:8085->8085/tcp
# kafka-connect0     Up X seconds    0.0.0.0:8083->8083/tcp
```

---

### Step 2: Start Backend

```bash
# Navigate to project root
cd /path/to/kafka-ui

# Start Spring Boot with local profile
./gradlew :api:bootRun --args='--spring.profiles.active=local'
```

**What this does:**

- Starts Spring Boot application on port 8080
- Uses `application-local.yml` configuration
- Connects to Kafka at `localhost:9092`
- Connects to Schema Registry at `localhost:8085`
- Authentication is **disabled** for development

**Wait for:**

```
Started KafkaUiApplication in X seconds
```

**Verify:**

```bash
curl http://localhost:8080/api/clusters
# Expected: JSON array with cluster info
```

---

### Step 3: Start Frontend

```bash
# Navigate to frontend directory
cd frontend

# Install dependencies (first time only)
pnpm install

# Start development server with proxy
VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

**What this does:**

- Starts Vite development server on port 3000
- Enables Hot Module Replacement (HMR)
- Proxies `/api/*` requests to backend at `:8080`
- TypeScript and ESLint checking enabled

**Wait for:**

```
VITE v6.4.1  ready in XXX ms

➜  Local:   http://localhost:3000/
```

---

## Stopping Everything

```bash
# Stop Frontend: Ctrl+C in Terminal 3

# Stop Backend: Ctrl+C in Terminal 2

# Stop Kafka Stack:
cd documentation/compose
docker compose -f kafbat-ui.yaml down

# Remove volumes (clean restart):
docker compose -f kafbat-ui.yaml down -v
```

---

## Port Reference

| Port | Service | Purpose |
|------|---------|---------|
| 3000 | Vite Dev Server | Frontend (development only) |
| 8080 | Spring Boot | Backend REST API |
| 9092 | Kafka Broker | Client connections |
| 9997 | Kafka JMX | Metrics collection |
| 8085 | Schema Registry | Schema management |
| 8083 | Kafka Connect | Connector management |
| 9090 | Prometheus | Metrics storage (optional) |

---

## Configuration Files

| File | Purpose |
|------|---------|
| `api/src/main/resources/application-local.yml` | Local dev config |
| `frontend/vite.config.ts` | Frontend dev server config |
| `documentation/compose/kafbat-ui.yaml` | Docker Compose stack |

---

## Troubleshooting

### Java Version Error

**Error:** `Unsupported class file major version 69`

**Cause:** Wrong Java version installed

**Solution:**
```bash
sdk install java 25-open
sdk default java 25-open
java --version  # Verify: 25.x.x
```

### Node Version Error

**Error:** `ERR_PNPM_UNSUPPORTED_ENGINE`

**Cause:** Wrong Node.js version

**Solution:**
```bash
nvm install 22
nvm use 22
node --version  # Verify: v22.x.x
```

### Kafka Connection Refused

**Error:** `Connection refused: localhost:9092`

**Cause:** Kafka not running

**Solution:**
```bash
cd documentation/compose
docker compose -f kafbat-ui.yaml up kafka0 -d
docker logs -f kafka0  # Wait for "started" message
```

### Port Already in Use

**Error:** `Port 8080/3000 already in use`

**Solution:**
```bash
# Find process using port
lsof -i :8080

# Kill process
kill -9 <PID>
```

**Alternative - Run on Different Port:**
```bash
# Backend on port 8081
./gradlew :api:bootRun --args='--spring.profiles.active=local --server.port=8081'

# Frontend with updated proxy
VITE_DEV_PROXY=http://localhost:8081 pnpm dev
```

**Common port 8080 culprits:**

- Other Spring Boot applications
- Apache Tomcat
- Jenkins
- nginx (sometimes)
- Docker containers from other projects

**Common port 3000 culprits:**

- Create React App / Next.js / Vite projects
- Grafana
- Express.js / Node.js servers
- Ruby on Rails

**Run frontend on different port:**
```bash
cd frontend
VITE_DEV_PROXY=http://localhost:8080 pnpm dev --port 3001
# Access at http://localhost:3001
```

### Homebrew Kafka/Zookeeper Conflict

**Error:** `Bind for 0.0.0.0:9092 failed: port is already allocated`

**Cause:** Homebrew-installed Kafka or Zookeeper is running and using the same ports as Docker containers.

**Solution:**
```bash
# Stop Homebrew services
brew services stop kafka
brew services stop zookeeper

# Verify ports are free
lsof -i :9092  # Should return nothing
lsof -i :2181  # Should return nothing

# Then start Docker infrastructure
cd documentation/compose
docker compose -f kafbat-ui.yaml up kafka0 schemaregistry0 -d
```

### Frontend API Calls Fail

**Error:** CORS errors or 404 on `/api/*`

**Cause:** Missing proxy configuration

**Solution:**
```bash
# Make sure to set the proxy environment variable
VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

---

## IDE Setup (IntelliJ IDEA)

### Backend Run Configuration

1. **Open project:** File → Open → select `kafka-ui` folder
2. **Wait for Gradle import** to complete
3. **Create Run Configuration:**
   - Run → Edit Configurations → + → Spring Boot
   - Main class: `io.kafbat.ui.KafkaUiApplication`
   - Active profiles: `local`
   - Module: `api.main`
4. **Click Run** (green arrow)

### Frontend (VS Code / WebStorm)

1. Open `frontend/` folder
2. Run `pnpm install` in terminal
3. Create `.env.local` with:
   ```
   VITE_DEV_PROXY=http://localhost:8080
   ```
4. Run `pnpm dev`

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REQUEST FLOW (Development)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Browser                                                                     │
│     │                                                                        │
│     │ GET http://localhost:3000/                                            │
│     ▼                                                                        │
│  ┌─────────────────────┐                                                    │
│  │   Vite Dev Server   │  Serves React app + Hot Module Replacement         │
│  │      :3000          │                                                    │
│  └─────────┬───────────┘                                                    │
│            │                                                                 │
│            │ GET /api/clusters  (proxied)                                   │
│            ▼                                                                 │
│  ┌─────────────────────┐                                                    │
│  │   Spring Boot API   │  REST API + WebFlux                                │
│  │      :8080          │                                                    │
│  └─────────┬───────────┘                                                    │
│            │                                                                 │
│            │ Kafka Protocol                                                 │
│            ▼                                                                 │
│  ┌─────────────────────┐                                                    │
│  │   Kafka Broker      │  Topics, Messages, Consumer Groups                 │
│  │      :9092          │                                                    │
│  └─────────────────────┘                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Useful Commands

```bash
# Check all versions
java --version && node --version && pnpm --version && docker --version

# View Kafka logs
docker logs -f kafka0

# View Schema Registry logs
docker logs -f schemaregistry0

# Test backend API
curl http://localhost:8080/api/clusters

# List Kafka topics
docker exec kafka0 kafka-topics --bootstrap-server localhost:29092 --list

# Clean Gradle build
./gradlew clean

# Clean frontend
cd frontend && rm -rf node_modules && pnpm install

# Full stack restart
docker compose -f documentation/compose/kafbat-ui.yaml down -v
docker compose -f documentation/compose/kafbat-ui.yaml up kafka0 schemaregistry0 kafka-connect0 -d
```

---

## Alternative: Full Docker Stack (No Code Changes)

If you just want to run Kafbat UI without development capabilities:

```bash
cd documentation/compose
docker compose -f kafbat-ui.yaml up -d
```

Then open http://localhost:8080

---

## Related Documentation

- [Official Docs](https://ui.docs.kafbat.io/)
- [Configuration Guide](https://ui.docs.kafbat.io/configuration/configuration-file)
- [Contributing Guide](https://ui.docs.kafbat.io/development/contributing)
- [Docker Compose Examples](https://ui.docs.kafbat.io/configuration/compose-examples)
