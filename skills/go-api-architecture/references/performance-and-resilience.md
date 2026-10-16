# Performance y resiliencia

Reglas para que la API sea rápida bajo carga y se recupere de fallos. Los valores concretos
dependen del tamaño del proyecto (ver calibración en `SKILL.md`).

## Pool de conexiones (GORM / `database/sql`)

GORM no tiene pool propio: usa el `*sql.DB` subyacente. **Sin configurarlo, el pool es ilimitado**
y una ráfaga de tráfico puede tumbar Postgres. Configúralo al abrir la conexión:

```go
sqlDB, err := db.DB() // *sql.DB subyacente
if err != nil {
	return err
}
sqlDB.SetMaxOpenConns(25)                  // conexiones abiertas máximas
sqlDB.SetMaxIdleConns(25)                  // ociosas (suele igualar a MaxOpen)
sqlDB.SetConnMaxLifetime(time.Hour)        // reciclar conexiones
sqlDB.SetConnMaxIdleTime(10 * time.Minute) // Go 1.15+
```

- `MaxOpenConns` debe ser ≤ `max_connections` de Postgres **÷ número de réplicas de la app**, con
  margen para migraciones y herramientas externas.
- `SetConnMaxLifetime` es clave detrás de **PgBouncer / load balancer**: evita usar conexiones ya
  cerradas por ellos.
- Monitoriza la **saturación** (conexiones en uso vs. `MaxOpenConns`) — ver
  `references/observability-and-ci.md`.
- Cierra el pool en el shutdown (`sqlDB.Close()`), ver `references/production-and-docs.md`.

## Timeouts en tres capas

Una petición no debe poder colgar recursos indefinidamente. Configura timeouts en **todas** las
capas:

1. **Servidor HTTP** (`http.Server`): sin `ReadHeaderTimeout`, un cliente lento agota conexiones
   (slowloris).
   ```go
   srv := &http.Server{
       Addr:              ":8080",
       Handler:           router,
       ReadHeaderTimeout: 5 * time.Second,
       ReadTimeout:       10 * time.Second,
       WriteTimeout:      15 * time.Second,
       IdleTimeout:       60 * time.Second,
       MaxHeaderBytes:    1 << 20, // 1 MiB
   }
   ```
2. **Contexto por request → DB**: propaga el `context` y llévalo a GORM con `db.WithContext(ctx)`
   para que la query se cancele al expirar o si el cliente se desconecta.
   ```go
   ctx, cancel := context.WithTimeout(c.Request.Context(), 3*time.Second)
   defer cancel()
   db.WithContext(ctx).Find(&users)
   ```
3. **Cliente HTTP saliente**: nunca uses `http.DefaultClient` sin timeout.
   ```go
   client := &http.Client{Timeout: 5 * time.Second}
   ```

## Idempotencia

Los endpoints de **creación crítica** (pagos, órdenes, registros) deben tolerar reintentos sin
duplicar. Dos capas:

- **Clave de idempotencia**: el cliente manda `Idempotency-Key` (UUID) en el header. Guardas
  `(key, endpoint, respuesta)` en una tabla con `UNIQUE(key)`; si la clave ya existe, devuelves la
  respuesta guardada en vez de reprocesar.
- **Dedup en DB**: `UNIQUE` sobre la clave natural + `ON CONFLICT DO NOTHING` (o `DO UPDATE`) para
  que el reintento sea un no-op en vez de un error 500.

```go
// repository, dentro de db.Transaction(...)
tx.Clauses(clause.OnConflict{
	Columns:   []clause.Column{{Name: "idempotency_key"}},
	DoNothing: true,
}).Create(&payment)
```

- Los `GET` son idempotentes por definición; no necesitan clave.
- Los reintentos van con **backoff** (abajo), nunca en bucle apretado.

## Concurrencia y transacciones

### Transacciones

- Agrupa varias escrituras que deban ser atómicas en `db.Transaction(...)`: devolver un error hace
  `Rollback`, devolver `nil` hace `Commit`.
- No compartas entre goroutines el `*gorm.DB` de una transacción.
- Para rollbacks parciales usa `SavePoint`/`RollbackTo` o transacciones anidadas.

### Concurrencia segura

- Trabajo paralelo (varias llamadas externas, etc.): usa `errgroup.WithContext` para cancelar todo
  si una parte falla, en vez de goroutines sueltas.
  ```go
  g, ctx := errgroup.WithContext(ctx)
  g.Go(func() error { return fetchA(ctx) })
  g.Go(func() error { return fetchB(ctx) })
  err := g.Wait()
  ```
- Nunca lances goroutines sin límite por request (agotan memoria); si necesitas paralelismo alto,
  usa un worker pool con canal.
- Protege estado compartido con `sync.Mutex`/`sync.RWMutex` o, mejor, evítalo.
- Corre la suite con **`go test -race`** en CI (ver `references/observability-and-ci.md`).

### Bloqueo optimista

Para evitar "lost updates" en recursos editables por varios usuarios, añade una columna `version`
y actualiza condicionalmente:

```go
res := db.Model(&doc).
	Where("id = ? AND version = ?", doc.ID, doc.Version).
	Updates(map[string]any{"body": doc.Body, "version": doc.Version + 1})
if res.RowsAffected == 0 {
	return ErrConflictoConcurrente // → HTTP 409
}
```

## Reintentos y resiliencia

- Reintenta **solo errores transitorios** (timeouts, conexión reset, `503`), con **backoff
  exponencial + jitter** y máximo de intentos. No reintentes errores de negocio (400/422) ni
  operaciones no idempotentes sin clave.
- Para llamadas a servicios externos, considera un **circuit breaker** (p. ej. `sony/gobreaker`):
  tras N fallos seguidos deja de llamar un tiempo y falla rápido.
- **Timeouts agresivos** en dependencias externas (arriba) y degradación elegante: devuelve lo que
  puedas, no cuelgues el request.

## Caché

- Redis (ya presente por el rate limiter) para **lecturas calientes** y datos que cambian poco.
- Claves con **TTL explícito**; invalida al escribir el recurso afectado.
- No caches por defecto: primero mide (observabilidad), luego cachea solo lo que haga falta.
  Cachear sin necesidad añade complejidad y datos obsoletos.

## Base de datos: índices y consultas

- **Índices** en: todas las FK, columnas de `WHERE`/`ORDER BY` frecuentes, y compuestos para
  filtros+orden que se usan juntos. Cada índice encarece la escritura: no indexar todo.
- **Paginación keyset** (cursor) en listados grandes, en vez de `OFFSET` alto (que escanea y
  descarta filas): filtra por el último valor visto (`WHERE id > ? ORDER BY id LIMIT ?`). Deja
  `page/per_page` solo para listados pequeños.
- Selecciona **solo las columnas** necesarias (`.Select("id", "email")`) en listados.
- Evita N+1 (ver `references/stack-and-structure.md`).

## Ajustes de rendimiento de GORM

- `SkipDefaultTransaction: true` en `gorm.Config`: GORM envuelve cada escritura en una transacción
  por defecto; desactívalo si no necesitas atomicidad por operación (menos overhead).
- `PrepareStmt: true`: cachea los prepared statements (menos parseo con queries repetidas).
- Logger de GORM solo en desarrollo (ver `references/stack-and-structure.md`).
