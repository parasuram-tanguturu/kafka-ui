# Active Context: Kafbat UI

## Current Work Focus

### Development Environment - READY

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CURRENT SETUP STATUS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Status: ✅ DEVELOPMENT ENVIRONMENT READY                                  │
│   Last Updated: 2026-02-20                                                  │
│                                                                              │
│   ┌────────────┬──────────────┬──────────────┬────────────────────────────┐ │
│   │ Component  │ Your Version │ Required     │ Status                     │ │
│   ├────────────┼──────────────┼──────────────┼────────────────────────────┤ │
│   │ Java       │ 25.0.2-zulu  │ 25+          │ ✅ OK (SDKMAN)             │ │
│   │ Node.js    │ 22.22.0      │ 22+          │ ✅ OK (NVM)                │ │
│   │ pnpm       │ 10.x         │ 10+          │ ✅ OK                      │ │
│   │ Docker     │ 29.x         │ Any          │ ✅ OK                      │ │
│   │ Gradle     │ 9.2.0        │ Auto         │ ✅ OK (wrapper)            │ │
│   └────────────┴──────────────┴──────────────┴────────────────────────────┘ │
│                                                                              │
│   ALL PREREQUISITES MET - Ready to start development stack                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Java Environment (SDKMAN)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SDKMAN JAVA CONFIGURATION                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Installed Versions:                                                        │
│   ───────────────────                                                       │
│   ┌─────────────┬──────────┬─────────────────┬────────────────────────────┐ │
│   │ Version     │ Vendor   │ Identifier      │ Status                     │ │
│   ├─────────────┼──────────┼─────────────────┼────────────────────────────┤ │
│   │ 11.0.27     │ Temurin  │ 11.0.27-tem     │ installed                  │ │
│   │ 17.0.15     │ Temurin  │ 17.0.15-tem     │ installed                  │ │
│   │ 21.0.10     │ Temurin  │ 21.0.10-tem     │ installed (DEFAULT)        │ │
│   │ 25.0.2      │ Zulu     │ 25.0.2-zulu     │ installed                  │ │
│   └─────────────┴──────────┴─────────────────┴────────────────────────────┘ │
│                                                                              │
│   Configuration:                                                             │
│   ──────────────                                                            │
│   • Default: Java 21.0.10-tem (for general use)                             │
│   • Auto-env: ENABLED (sdkman_auto_env=true)                                │
│   • kafka-ui project: Auto-switches to Java 25 (reads .java-version)       │
│                                                                              │
│   Homebrew JDKs (kept for tool dependencies):                               │
│   ────────────────────────────────────────────                              │
│   • openjdk       → used by: jmeter, kafka, zookeeper                       │
│   • openjdk@21    → used by: cypher-shell, neo4j                            │
│   • Removed: openjdk@11, openjdk@17 (no dependencies)                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Recent Changes

| Date | Change | Details |
|------|--------|---------|
| 2026-02-21 | YAML-configurable validation timeouts | Made AdminClient timeouts configurable via YAML |
| 2026-02-21 | Created timeout config docs | `CONFIGURABLE_VALIDATION_TIMEOUTS.md` |
| 2026-02-21 | Confluent Cloud fixes | Fixed quorum auth + validation timeout issues |
| 2026-02-21 | Created debug session docs | `CONFLUENT_CLOUD_DEBUG_SESSION.md` |
| 2026-02-21 | Created Quorum explainer | `KAFKA_QUORUM_EXPLAINED.md` |
| 2026-02-20 | Node.js 22 configured | v22.22.0 via NVM, use `nvm use 22` |
| 2026-02-20 | SDKMAN Java setup complete | Installed Java 11, 17, 21, 25 via SDKMAN |
| 2026-02-20 | Set Java 21 as default | System default for new terminals |
| 2026-02-20 | Enabled auto-env | `.java-version` files auto-switch Java |
| 2026-02-20 | Cleaned Homebrew JDKs | Removed unused openjdk@11, openjdk@17 |
| 2026-02-20 | Gradle wrapper ready | Downloaded Gradle 9.2.0 |
| 2026-02-20 | Created Memory Bank | Full project documentation |

---

## Next Steps

### Remaining Setup Tasks

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           NEXT STEPS CHECKLIST                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ✅ 1. Java 25 configured via SDKMAN                                       │
│       └── Auto-switches when entering kafka-ui directory                   │
│                                                                              │
│   ✅ 2. Node.js 22 configured via NVM                                       │
│       └── v22.22.0 installed and active                                    │
│       └── Run: nvm use 22 (in each new terminal)                           │
│                                                                              │
│   □ 3. Start Kafka Stack (Terminal 1)                                       │
│       └── cd documentation/compose                                          │
│       └── docker compose -f kafbat-ui.yaml up kafka0 schemaregistry0 -d    │
│                                                                              │
│   □ 4. Start Backend (Terminal 2)                                           │
│       └── ./gradlew :api:bootRun --args='--spring.profiles.active=local'   │
│                                                                              │
│   □ 5. Start Frontend (Terminal 3)                                          │
│       └── cd frontend && pnpm install                                       │
│       └── VITE_DEV_PROXY=http://localhost:8080 pnpm dev                    │
│                                                                              │
│   □ 6. Verify UI at http://localhost:3000                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Active Decisions and Considerations

### Java Version Management

**Chosen: SDKMAN with auto-env**

```
WHY this option:
────────────────
• Automatic version switching per project (.java-version)
• Multiple Java versions coexist cleanly
• Homebrew tools still work (have their own JDKs)
• No manual JAVA_HOME management needed

Configuration:
──────────────
• Default: Java 21 (most common for other projects)
• kafka-ui: Auto-switches to Java 25
• Other projects: Add .java-version file as needed
```

### Development Mode Choice

**Chosen: Full Development Mode (3 Terminals)**

```
WHY this option:
────────────────
• Hot Module Replacement for frontend changes
• Java debugger can be attached to backend
• Full control over each component
• Best for contributing code changes
```

---

## Quick Commands Reference

```bash
# Check Java version (should show 25 in kafka-ui directory)
cd /Users/parasuram/Ruckus/Kafka/kafka-ui
java -version

# Switch Java manually
sdk use java 21.0.10-tem    # Use Java 21 temporarily
sdk use java 25.0.2-zulu    # Use Java 25 temporarily
sdk default java 21.0.10-tem # Set system default

# List installed Java versions
sdk list java | grep installed

# Test Gradle build
./gradlew --version
./gradlew :api:compileJava

# Start development stack
# T1: docker compose -f documentation/compose/kafbat-ui.yaml up kafka0 schemaregistry0 -d
# T2: ./gradlew :api:bootRun --args='--spring.profiles.active=local'
# T3: cd frontend && VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

---

## Environment Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FINAL ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  YOUR DEVELOPMENT        →  SDKMAN (~/.sdkman/candidates/java/)             │
│  (kafka-ui, etc.)            Java 25, 21, 17, 11 as needed                  │
│                                                                              │
│  HOMEBREW TOOLS          →  Homebrew (/opt/homebrew/opt/)                   │
│  (kafka CLI, zookeeper)      Uses its own bundled Java                      │
│                                                                              │
│  No conflict! They're independent.                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
