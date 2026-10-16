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

## Migraciones de base de datos

`golang-migrate/migrate` es la **única fuente de verdad del esquema**, en desarrollo y en
producción. `AutoMigrate` de GORM **no** es el camino por defecto ni en dev: solo se tolera en
prototipos desechables, nunca commiteado (ver `stack-and-structure.md`).

### Desarrollo (local)

- Se aplican los **mismos** archivos SQL versionados que en producción; la DB local vive en
  `compose.yml`.
- Aplicar/revertir: `make migrate-up` / `make migrate-down` (envuelven los comandos de la sección
  de producción con el `DATABASE_URL` local).
- Crear una migración nueva:

  ```bash
  migrate create -ext sql -dir migrations -seq <nombre>
  ```

  Genera el par `NNNNNN_<nombre>.up.sql` y `NNNNNN_<nombre>.down.sql`; **ambos** deben escribirse.
  El número es incremental y no se reutiliza.
- Datos de desarrollo: `migrations/seed.dev.sql` aplicado con `make seed` (idempotente; no es una
  migración versionada).
- Nunca uses `AutoMigrate` para "arreglar" la DB local: si el esquema cambió, crea una migración.

### Producción

Las migraciones son un paso de **despliegue**, no de arranque de la aplicación. En
producción se ejecutan con `golang-migrate/migrate` como SQL versionado (ver
`stack-and-structure.md`); `AutoMigrate` de GORM está prohibido en producción.

**Cómo se ejecuta el paso de migración:**

- Como step explícito de CI/CD o job de contenedor separado (ej. servicio `migrate` en
  `compose.prod.yml` o `initContainer` si usas Kubernetes), ejecutado **antes** del rollout y
  con salida bloqueante si falla.
- Nunca en el `CMD` de la app: dos réplicas arrancando a la vez intentarían migrar en
  paralelo. Si se usa job de contenedor, limitarlo a una sola replica (`deploy.replicas: 1`).
- La versión de la imagen del job de migración debe ser la misma del release que se despliega
  (mismas migraciones), para evitar aplicar el esquema de una versión distinta.

**Comandos core de `golang-migrate`:**

```bash
# Aplicar todas las migraciones pendientes
migrate -path migrations -database "$DATABASE_URL" up

# Revertir la última migración (rollback manual/incidente)
migrate -path migrations -database "$DATABASE_URL" down 1

# Versión aplicada y flag dirty (migración a medio aplicar por un fallo)
migrate -path migrations -database "$DATABASE_URL" version

# Forzar una versión marcada como "sucia" por un fallo parcial
migrate -path migrations -database "$DATABASE_URL" force N
```

`DATABASE_URL` viene de los secrets del entorno, nunca del repo. El `AGENTS.md` debe listar
estos comandos para que el agente no los invente.

**Prácticas seguras:**

- Migraciones **aditivas primero** (patrón *expand/contract*): crear columnas/tablas en un
  release, backfillear datos con `UPDATE` en la misma o siguiente migración, y recién en un
  release posterior eliminar/deprecar lo viejo. Nunca `DROP COLUMN`/`DROP TABLE` en el mismo
  release que escribe el nuevo esquema.
- Usar `IF NOT EXISTS`/`IF EXISTS` y transacciones explícitas (`BEGIN`/`COMMIT`, o los
  separadores `-- +migrate Up`/`-- +migrate Down` de golang-migrate) cuando la migración lo
  permita.
- No editar migraciones ya aplicadas en un entorno: crear una migración nueva que corrija.
  El historial versionado es inmutable una vez aplicado.
- `golang-migrate` usa advisory lock de Postgres: el `up` se serializa solo, pero igual no
  lanzar dos migradores a la vez; el lock evita corrupción, no la confusión operativa.
- Verificar tras el release con `migrate ... version` y chequear que el readiness del app
  valide el esquema esperado (no solo que la DB responde `Ping`).

**Rollback:** las migraciones no son el mecanismo de rollback del código. Si el release falla,
se revierte el despliegue de la app y, si fue una migración destructiva/fallida, se corrige con
`down 1` o una migración nueva que restituya — nunca con edición retroactiva de archivos.

## Testing

- Tests unitarios de `service.go` con mocks del repositorio.
- Tests de integración de endpoints críticos con `httptest`.
- Makefile con target `make test`.

### Convenciones de archivos y naming

`go test` obliga a que los tests vivan en `*_test.go` dentro del mismo directorio que el código
probado, así que la convención es co-locación en el módulo (no hay carpeta `test/` aparte):

- **Nombre:** `<archivo>_test.go` para unitarios (`service_test.go`, `handler_test.go`).
- **Integración:** `<feature>_integration_test.go`, aislados con `testing.Short()` (se saltan con
  `go test -short`, que es lo que corre el ciclo normal). No combinar con build tags.
- **Package de test:** `package <modulo>_test` (black-box) por defecto; `package <modulo>` solo
  cuando el test necesite símbolos no exportados.
- **Mocks:** `testify/mock` (ya está en el stack, sin dependencias nuevas), definidos en un
  `mocks_test.go` dentro del paquete de test del módulo; los `_test.go` no entran al binario.

### Contenedores efímeros (`testcontainers-go`)

Uso condicionado a la respuesta de la pregunta 6 del SKILL.md. Librería:
`github.com/testcontainers/testcontainers-go` (módulo `testcontainers-go/modules/postgres`).

**A qué código se aplica** (NO a todo el proyecto):
- Solo a procesos críticos que dependen fuertemente de la DB: autenticación, pagos, reglas de
  negocio complejas entre varias tablas, endpoints propensos a N+1.
- CRUDs simples se quedan con tests unitarios + mocks.

**Flujo:** levanta un contenedor real de Postgres → corre migraciones → ejecuta el
repositorio/servicio real contra esa DB → destruye el contenedor al terminar con
`testcontainers.CleanupContainer(t, ctr)`. Esa es la forma recomendada actual: se apoya en
`t.Cleanup` y tolera `nil`, por eso puede registrarse antes de comprobar errores. Requiere
Docker disponible en la máquina/CI (Ryuk actúa como red de seguridad si algo quedara colgado).

### Ejemplo canónico

Tests de integración **por test** (simple y estable). Aplica solo a los procesos críticos
definidos arriba; si el volumen de tests crece y arrancar un contenedor por test se vuelve
lento, evalúa arrancar uno solo desde `TestMain`. La reutilización entre tests con
`WithReuseByName` es experimental y no se recomienda como base.

```go
// internal/auth/auth_integration_test.go
package auth_test

import (
	"context"
	"errors"
	"testing"

	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
	"gorm.io/driver/postgres" // mismo driver que usa la app
	"gorm.io/gorm"
)

// setupDB levanta un Postgres real, aplica migraciones y devuelve la *gorm.DB lista.
// Se usa en tests de procesos críticos (auth, pagos), NUNCA en CRUDs simples.
func setupDB(t *testing.T) *gorm.DB {
	t.Helper()
	if testing.Short() {
		t.Skip("test de integración: omitido con -short")
	}

	ctx := context.Background()
	ctr, err := postgres.Run(ctx, "postgres:16-alpine",
		postgres.WithDatabase("app_test"),
		postgres.WithUsername("test"),
		postgres.WithPassword("test"),
		postgres.BasicWaitStrategies(),
	)
	if err != nil {
		t.Fatalf("no se pudo levantar postgres: %v", err)
	}
	// Se registra el cleanup ANTES de comprobar errores: CleanupContainer tolera nil.
	testcontainers.CleanupContainer(t, ctr)

	connStr, err := ctr.ConnectionString(ctx, "sslmode=disable")
	if err != nil {
		t.Fatalf("connection string: %v", err)
	}

	db, err := gorm.Open(postgres.Open(connStr), &gorm.Config{})
	if err != nil {
		t.Fatalf("gorm.Open: %v", err)
	}
	sqlDB, err := db.DB() // *sql.DB subyacente que necesita golang-migrate
	if err != nil {
		t.Fatalf("db.DB(): %v", err)
	}
	if err := runMigrations(sqlDB); err != nil { // golang-migrate sobre migrations/ (mismos SQL que prod); NO AutoMigrate
		t.Fatalf("migraciones: %v", err)
	}
	return db
}

func TestLogin_CredencialesInvalidas(t *testing.T) {
	db := setupDB(t)
	svc := NewService(NewRepository(db)) // servicio REAL, no mock

	_, err := svc.Login(context.Background(), "noexiste@test.com", "mala")
	if !errors.Is(err, ErrCredencialesInvalidas) {
		t.Fatalf("se esperaba ErrCredencialesInvalidas, got %v", err)
	}
}
```

## Generación de AGENTS.md y ARCHITECTURE.md

`AGENTS.md` es un estándar abierto (Linux Foundation, soportado por Claude Code, Cursor,
Copilot, Codex, Gemini CLI, OpenCode) que se lee en cada interacción del agente — por eso debe
ser corto y operativo, no un documento de arquitectura completo.

### `AGENTS.md` (raíz) — operativo

- Comandos: instalar, correr, testear, migrar, generar Swagger, observabilidad (`make obs-up`) y CI.
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
- Configuración del pool de conexiones (valores) y timeouts aplicados.
- Nivel de observabilidad elegido y dónde mirar (logs / métricas / alertas).
- Política de testing (qué módulos tienen `testcontainers-go` y por qué).
- Graceful shutdown y probes implementados en este proyecto.
- Diferencias entre `compose.yml` y `compose.prod.yml`, manejo de secrets.
- Decisiones que se desviaron explícitamente del estándar de este skill, para que un agente
  futuro no intente "corregirlas".

Ambos se mantienen vivos: `AGENTS.md` cuando cambian comandos/reglas operativas,
`ARCHITECTURE.md` cuando cambia una decisión de arquitectura. Un módulo complejo puede tener su
propio `AGENTS.md` local en `internal/<modulo>/` (el estándar soporta jerarquía).
