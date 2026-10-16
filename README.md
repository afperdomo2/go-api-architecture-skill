# go-api-architecture-skill

Skill de Agent Skills (formato universal `SKILL.md`) que define el estándar de arquitectura
para construir APIs REST profesionales en Go: stack (Gin + GORM), estructura de
carpetas modular, respuestas estandarizadas (RFC 9457), prevención de N+1, performance y
resiliencia (pool de conexiones, timeouts, idempotencia, concurrencia), observabilidad (niveles
desde `docker logs` hasta Grafana), seguridad, documentación Swagger, testing (unitario y con
`testcontainers-go`), graceful shutdown, probes de liveness/readiness, buenas prácticas de
Docker/Compose en producción, y generación de `AGENTS.md`/`ARCHITECTURE.md`.

## Instalación

Con el CLI de [skills.sh](https://skills.sh):

```bash
npx skills add afperdomo2/go-api-architecture-skill
```

Esto la instala y configura automáticamente para tu agente (Claude Code, Cursor, Copilot,
OpenCode, etc.).

## Instalación manual

Copia la carpeta `skills/go-api-architecture/` a donde tu agente busque skills:

- **Claude Code / claude.ai**: `~/.claude/skills/` o subir como ZIP en Settings → Skills.
- **OpenCode**: `~/.config/opencode/skills/` o `.opencode/skill/` del proyecto.
- **GitHub Copilot (VSCode)**: referenciar como prompt file en `.github/prompts/`.

## Estructura

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

[![skills.sh](https://skills.sh/b/afperdomo2/go-api-architecture-skill)](https://skills.sh/afperdomo2/go-api-architecture-skill)
