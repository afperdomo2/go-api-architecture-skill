# Stack, respuestas estandarizadas y estructura de carpetas

## Stack tecnológico estándar

- **Framework HTTP:** Gin (`gin-gonic/gin`)
- **ORM:** GORM (`gorm.io/gorm`) + driver `gorm.io/driver/postgres`
- **Migraciones:** `golang-migrate/migrate` (SQL versionado) es la **única fuente de verdad del
  esquema**, en dev y en producción. `AutoMigrate` de GORM no se usa (solo tolerado en prototipos
  desechables, nunca commiteado). Ver `references/production-and-docs.md`.
- **Validación de entrada:** `go-playground/validator/v10` vía binding de Gin
- **Documentación:** `swaggo/swag` + `gin-swagger` → `/swagger/index.html`
- **Autenticación:** JWT con `golang-jwt/jwt/v5`, refresh tokens, passwords con `bcrypt`
- **Rate limiting / fuerza bruta:** `ulule/limiter` (memoria o Redis según escala)
- **Logging estructurado:** `rs/zerolog` (JSON, con request ID por petición)
- **Configuración:** `spf13/viper` + `.env` (nunca credenciales hardcodeadas)
- **Testing:** `testify` + `net/http/httptest`
- **Contenedor:** Docker multi-stage build (imagen final `scratch` o `distroless`)

## Estandarización de respuestas (RFC 9457)

Envelope consistente en toda la API. Para errores seguimos RFC 9457 (Problem Details for HTTP
APIs, sucesor de RFC 7807):

**Éxito:**
```json
{
  "success": true,
  "data": { ... },
  "meta": { "page": 1, "per_page": 20, "total": 134 }
}
```

**Error:**
```json
{
  "success": false,
  "error": {
    "type": "https://api.dominio.com/errors/validation-error",
    "title": "Error de validación",
    "status": 422,
    "detail": "El campo 'email' no tiene un formato válido",
    "instance": "/api/v1/users",
    "code": "VALIDATION_ERROR"
  }
}
```

- Códigos HTTP semánticamente correctos (`400`, `401`, `403`, `404`, `409`, `422`, `429`, `500`).
- Error handler centralizado (middleware de Gin) que captura panics/errores; nunca stack
  traces al cliente.
- `code` interno estable para que el frontend haga `switch` sin parsear el mensaje.

## Estructura de carpetas

Modular por feature/negocio dentro de `internal/` — NO por tipo técnico:

```
proyecto/
├── cmd/api/main.go
├── internal/
│   ├── users/
│   │   ├── handler.go       # capa HTTP (Gin)
│   │   ├── service.go       # lógica de negocio
│   │   ├── repository.go    # acceso a datos (GORM)
│   │   ├── model.go         # struct de dominio + tags GORM/json/validate
│   │   └── dto.go           # request/response (si difieren del modelo)
│   ├── auth/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── middleware.go    # JWT auth middleware
│   │   └── jwt.go
│   ├── shared/
│   │   ├── response/        # helpers del envelope estándar
│   │   ├── errors/          # tipos de error + mapeo a HTTP status
│   │   ├── middleware/       # logging, recovery, CORS, rate limit
│   │   ├── validator/
│   │   └── db/               # conexión GORM, health check
│   └── config/config.go
├── migrations/
├── docs/                      # swagger autogenerado
├── pkg/                        # reutilizable fuera del dominio (si aplica)
├── .env.example
├── Dockerfile
├── compose.yml
├── Makefile
├── AGENTS.md
├── ARCHITECTURE.md
└── go.mod
```

Modelo de datos: struct único por entidad ("Opción A") con tags GORM/json/validate en el mismo
struct, salvo que el request/response necesite forma distinta al modelo de dominio (ahí sí DTO
explícito).

## Convenciones de modelado

- **Clave primaria**: `UUID v7` por defecto (`github.com/google/uuid` → `uuid.NewV7()`): ordenable
  por tiempo, no revela el volumen de datos ni permite enumerar recursos. Alternativa
  `BIGSERIAL` para tablas internas de alto volumen donde el tamaño importa.
- **Timestamps**: `created_at` y `updated_at` en toda entidad (GORM los autogestiona).
- **Soft delete**: `gorm.DeletedAt` solo si necesitas conservar registros borrados; si no, borrado
  físico. Un soft delete "invisible" complica los `UNIQUE` (un email borrado sigue ocupando la
  clave) — decídelo a consciencia.
- **Optimistic locking**: columna `version` en recursos editables por varios usuarios (ver
  `references/performance-and-resilience.md`).
- **Naming**: tablas y columnas en `snake_case`, tablas en plural.

## Prevención de consultas N+1

- Usar `.Preload("Relacion")` (o anidado, `.Preload("Orders.Items")`) en vez de acceder a la
  relación de forma perezosa dentro de un loop.
- Para listados con relaciones simples de solo lectura, preferir un `JOIN` explícito
  (`.Joins(...)`) mapeado a un DTO plano.
- Nunca hacer queries dentro de un `for` sobre resultados ya obtenidos (antipatrón clásico:
  `for _, u := range users { db.Where("user_id = ?", u.ID).Find(&orders) }`).
- En desarrollo, activar el logger de GORM (`Logger: logger.Default.LogMode(logger.Info)`) para
  detectar ráfagas de queries repetidas por endpoint.
- Endpoints de listados con relaciones llevan test de integración que valida también el número
  de queries ejecutadas (ver `references/production-and-docs.md`).

## Filtros booleanos y valores cero en GORM

- `Where` con **struct ignora los valores cero** (`false`, `0`, `""`, `nil`): 
  `db.Where(&User{Active: false})` **no** genera `active = false`, así que un filtro tipo
  `?active=false` devuelve toda la colección sin filtrar.
- Para booleanos y cualquier filtro cuyo valor pueda ser cero, usar **map** (que incluye todas
  sus claves) o **condición explícita**:

```go
db.Where(map[string]any{"active": false}) // ... WHERE active = false
// o
db.Where("active = ?", false)
```

- **Patrón completo**: el handler trae el booleano como `*bool` (ver
  `references/security-and-endpoints.md`, sección "Filtros booleanos en query params"). En el
  repository se arma un `map[string]any` con **solo las claves presentes** — añadir `"active"`
  únicamente si `q.Active != nil` — y se pasa a `db.Where(filters)`:

```go
filters := map[string]any{}
if q.Active != nil {
    filters["active"] = *q.Active
}
db.Where(filters).Find(&users)
```

- Los endpoints de listado con filtro booleano llevan test de integración obligatorio que
  verifica que `?active=false` filtra de verdad y no devuelve la colección completa.
