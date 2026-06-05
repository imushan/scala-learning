---
name: scala-hex-build
description: Use when user asks to build, compile, test, run, or format a Scala project, or mentions bleep compile/test/run. Do NOT use for Docker/deployment tasks.
---

# Build and run Scala hexagonal project

Build, test, and run a Scala hexagonal architecture project using Bleep.

## Usage
- User says: "build the project" or "compile scala" or "run tests" or "start the app"

## Commands

### Compile
```bash
# Compile all modules
bleep compile

# Compile specific module
bleep compile core
bleep compile logic
bleep compile api
bleep compile infra
```


### Test
```bash
bleep test tests
```

### Run
```bash
bleep run app
```

### Format
```bash
bleep fmt
```

### Clean
```bash
bleep clean
```

### Build Distribution (for Docker)
```bash
bleep my-dist
# or
./bleep my-dist
```

## Module Structure
- `core` - Domain entities, value objects, ports (no external deps)
- `logic` - Business logic, depends on core
- `infra` - Infrastructure, database implementations, depends on core
- `api` - HTTP routes, depends on logic
- `app` - Main entry point, depends on api and infra
- `tests` - Test module

## Dependency Direction
```
App → API → Logic → Core
App → Infra → Core
```

## Important
- Do NOT reinstall bleep or coursier - they are pre-installed
- The `scripts` module in bleep.yaml MUST be preserved for `my-dist` to work
- If `bleep my-dist` fails, check that scripts module exists