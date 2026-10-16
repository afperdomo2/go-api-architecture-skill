# go-api-architecture-skill

[![skills.sh](https://skills.sh/b/afperdomo2/go-api-architecture-skill)](https://skills.sh/afperdomo2/go-api-architecture-skill/go-api-architecture)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Go](https://img.shields.io/badge/Go-1.22%2B-00ADD8.svg)](https://go.dev)

Skill de [Agent Skills](https://skills.sh) (formato universal `SKILL.md`) que define el estándar
de arquitectura para construir APIs REST profesionales en Go con **Gin + GORM**.

## ✨ Qué incluye

- **Stack y estructura**: Gin + GORM, estructura modular por feature en `internal/`, respuestas
  estandarizadas (RFC 9457) y prevención de consultas N+1.
- **Modelado**: clave `UUID v7`, timestamps, soft delete y bloqueo optimista.
- **Seguridad**: JWT (con validación de `alg`), rate limiting y anti fuerza bruta, CORS, CSRF y
  límite de tamaño de request.
- **Performance y resiliencia**: pool de conexiones, timeouts en 3 capas, idempotencia,
  transacciones y concurrencia, reintentos con backoff, caché, índices y paginación keyset.
- **Observabilidad**: 4 niveles (desde `docker logs` hasta Grafana), métricas, golden signals y CI.
- **Producción**: graceful shutdown, probes de liveness/readiness, Docker/Compose y migraciones
  con `golang-migrate`.
- **Testing**: unitario e integración con `testcontainers-go`.
- **Salida**: genera `AGENTS.md` y `ARCHITECTURE.md` al final del proyecto.

## 🚀 Instalación

Con el CLI de [skills.sh](https://skills.sh):

```bash
npx skills add afperdomo2/go-api-architecture-skill
```

Esto la instala y configura automáticamente para tu agente (Claude Code, Cursor, Copilot,
OpenCode, etc.).

### Instalación manual

Copia la carpeta `skills/go-api-architecture/` a donde tu agente busque skills:

- **Claude Code / claude.ai**: `~/.claude/skills/` o subir como ZIP en Settings → Skills.
- **OpenCode**: `~/.config/opencode/skills/` o `.opencode/skill/` del proyecto.
- **GitHub Copilot (VSCode)**: referenciar como prompt file en `.github/prompts/`.

## 🧭 Cómo funciona

Al activarse, la skill primero hace **preguntas de calibración** (dominio, tamaño, auth, base de
datos, testcontainers, observabilidad…) y luego genera el andamiaje completo de la API de forma
consistente: estructura de carpetas, middlewares, módulo `auth`, primer módulo de negocio,
`Dockerfile`/`compose.yml`, migraciones, observabilidad y la documentación (`AGENTS.md` y
`ARCHITECTURE.md`).

## 📁 Estructura

```
skills/go-api-architecture/
├── SKILL.md                              # flujo de calibración y orden de ejecución
└── references/
    ├── stack-and-structure.md            # stack, RFC 9457, carpetas, modelado, N+1
    ├── security-and-endpoints.md         # seguridad y estandarización de endpoints
    ├── production-and-docs.md            # graceful shutdown, probes, Docker/Compose, migraciones, testing, AGENTS.md
    ├── performance-and-resilience.md     # pool, timeouts, idempotencia, concurrencia, caché, índices
    └── observability-and-ci.md           # métricas/logs/trazas, niveles de observabilidad, CI
```

## ⚙️ Requisitos

- **Go 1.22+**
- **Docker** (para `compose` y los tests de integración con `testcontainers-go`)
- **Git**

## 📄 Licencia

Distribuida bajo licencia [MIT](LICENSE).
