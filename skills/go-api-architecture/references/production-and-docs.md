# Producción, testing y documentación para agentes

## Graceful shutdown (cierre controlado)

Docker envía `SIGTERM` al PID 1 al detener/reiniciar un contenedor. Si el proceso no termina en
el tiempo límite (10s por defecto, configurable con `stop_grace_period`), fuerza `SIGKILL`.

- Capturar `SIGTERM`/`SIGINT` con `signal.NotifyContext`.
- Al recibir la señal, marcar el servicio como "no listo" primero (ver readiness abajo).
- Usar `srv.Shutdown(ctx)` (no `Close()`) con timeout razonable (15–20s).
- Cerrar después el pool de GORM (`sqlDB.Close()`) y otros recursos (Redis, colas).
- El `Dockerfile` debe usar **forma exec** en `CMD`/`ENTRYPOINT` (`CMD ["./api"]`, nunca
  `CMD ./api`) — la forma shell antepone `/bin/sh -c`, que se queda con el `SIGTERM` y nunca lo
  reenvía al binario de Go.

## Liveness vs. readiness

Dos endpoints separados, con semántica distinta:

- **Liveness** (`/health` o `/healthz`): el proceso está vivo. NO depende de la DB — un timeout
  de DB no significa que el proceso esté muerto. Si falla sostenido, el orquestador reinicia.
- **Readiness** (`/ready` o `/readyz`): la instancia está lista para tráfico — valida DB
  (`sqlDB.PingContext`) y dependencias críticas. Si falla, el proxy saca la instancia del
  balanceo, pero NO se reinicia el contenedor.
- Durante el shutdown: `/ready` responde `503` de inmediato al recibir `SIGTERM`, mientras
  `Shutdown(ctx)` drena las peticiones activas. Liveness sigue en `200` hasta que el proceso
  realmente termina.

## Docker y Compose en producción

- `CMD` en forma exec (ver arriba).
- Usuario no-root dentro del contenedor (`USER appuser`).
- `HEALTHCHECK` apuntando a **readiness**, no liveness.
- `stop_grace_period` explícito en `compose.yml`, con margen sobre el timeout de
  `Shutdown(ctx)`.
- Restart policy explícita (`unless-stopped` o `on-failure:5`; evitar `always` sin límite).
- Límites de recursos (`deploy.resources.limits` o `--memory`/`--cpus`).
- Archivo separado para producción: `compose.yml` (base/dev) + `compose.prod.yml` (override),
  `docker compose -f compose.yml -f compose.prod.yml up`.
- Secrets nunca en `compose.yml` versionado ni horneados en la imagen: `env_file` fuera del
  repo, o secret manager del proveedor.
- Multi-stage build para que la imagen final no incluya el toolchain de Go.

## Testing

- Tests unitarios de `service.go` con mocks del repositorio.
- Tests de integración de endpoints críticos con `httptest`.
- Makefile con target `make test`.

### Contenedores efímeros (`testcontainers-go`)

Uso condicionado a la respuesta de la pregunta 6 del SKILL.md. Librería:
`github.com/testcontainers/testcontainers-go` (módulo `testcontainers-go/modules/postgres`).

**A qué código se aplica** (NO a todo el proyecto):
- Solo a procesos críticos que dependen fuertemente de la DB: autenticación, pagos, reglas de
  negocio complejas entre varias tablas, endpoints propensos a N+1.
- CRUDs simples se quedan con tests unitarios + mocks.

**Flujo:** levanta un contenedor real de Postgres → corre migraciones → ejecuta el
repositorio/servicio real contra esa DB → destruye el contenedor al terminar
(`defer container.Terminate(ctx)`). Requiere Docker disponible en la máquina/CI.

## Generación de AGENTS.md y ARCHITECTURE.md

`AGENTS.md` es un estándar abierto (Linux Foundation, soportado por Claude Code, Cursor,
Copilot, Codex, Gemini CLI, OpenCode) que se lee en cada interacción del agente — por eso debe
ser corto y operativo, no un documento de arquitectura completo.

### `AGENTS.md` (raíz) — operativo

- Comandos: instalar, correr, testear, migrar, generar Swagger.
- Convenciones de código (`gofmt`/`golangci-lint`).
- Reglas de dónde va cada cosa (ej. "handler nunca llama a GORM directo").
- Restricciones explícitas (qué no tocar sin confirmar).
- Cómo agregar un módulo nuevo, en pasos cortos.
- Una línea apuntando a `ARCHITECTURE.md` para el razonamiento detrás de las reglas.

### `ARCHITECTURE.md` (raíz) — contexto profundo

- Stack real usado y por qué, sobre las alternativas descartadas.
- Estructura de carpetas real con la razón de la organización modular.
- Convenciones de respuesta/errores con ejemplo real del proyecto.
- Reglas de seguridad activas (dónde vive el middleware de auth, rate limiting configurado).
- Política de N+1 (dónde está el logger de GORM activado).
- Política de testing (qué módulos tienen `testcontainers-go` y por qué).
- Graceful shutdown y probes implementados en este proyecto.
- Diferencias entre `compose.yml` y `compose.prod.yml`, manejo de secrets.
- Decisiones que se desviaron explícitamente del estándar de este skill, para que un agente
  futuro no intente "corregirlas".

Ambos se mantienen vivos: `AGENTS.md` cuando cambian comandos/reglas operativas,
`ARCHITECTURE.md` cuando cambia una decisión de arquitectura. Un módulo complejo puede tener su
propio `AGENTS.md` local en `internal/<modulo>/` (el estándar soporta jerarquía).
