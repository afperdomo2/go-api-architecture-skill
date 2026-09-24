# Seguridad y estandarización de endpoints

## Seguridad (no negociable, aplica siempre)

- **Sanitización de inputs**: validación estricta vía `validator` en el binding de cada
  endpoint (whitelist de campos esperados); nunca bind directo de payload libre a modelo GORM
  sin pasar por DTO validado.
- **Prevención de SQL injection**: solo queries parametrizadas vía GORM; prohibido concatenar
  SQL con input de usuario.
- **Protección contra fuerza bruta**:
  - Rate limiting global por IP (ej. 100 req/min).
  - Rate limiting específico y más estricto en `/login`, `/register`, `/forgot-password`
    (ej. 5 intentos/min por IP + por usuario).
  - Bloqueo temporal incremental tras N intentos fallidos de login (backoff exponencial).
- **Headers de seguridad**: middleware tipo Helmet (CSP, X-Frame-Options,
  X-Content-Type-Options, HSTS si hay TLS terminado en la app).
- **CORS**: whitelist explícita de orígenes (nunca `*` en producción si hay auth con cookies).
- **Passwords**: `bcrypt` con cost ≥ 12, nunca loggear passwords ni tokens.
- **JWT**: expiración corta en access token (ej. 15 min) + refresh token con rotación.
- **Secrets**: nunca en el repo; siempre por variables de entorno / secret manager.

## Estandarización de endpoints

- Versionado explícito: `/api/v1/...`
- Sustantivos en plural, sin verbos: `GET /api/v1/users`, no `/getUsers`.
- Anidación solo un nivel cuando hay relación clara: `/api/v1/users/{id}/orders`.
- Verbos HTTP con semántica correcta: `GET` (leer), `POST` (crear), `PUT` (reemplazar),
  `PATCH` (actualizar parcial), `DELETE` (eliminar).
- **Paginación estándar**: `?page=1&per_page=20`, respuesta con `meta` (ver
  `stack-and-structure.md`).
- **Filtros y ordenamiento estándar**: `?sort=-created_at&status=active`.
- Documentación Swagger obligatoria por endpoint (`@Summary`, `@Param`, `@Success`, `@Failure`
  en comentarios de `swaggo`).

### Filtros booleanos en query params

- Nunca `c.Query("active")` a secas ni `strconv.ParseBool` ignorando el error:
  `c.Query` devuelve `""` tanto si el param **falta** como si está vacío, y `ParseBool("")`
  falla → un param ausente se convierte silenciosamente en `false` y filtras mal.
- Para distinguir "no enviado" de `false`, bindear a un struct de filtros con `*bool`
  (tag `form:"active"`): si el param no viene queda `nil`; si viene `?active=false`, queda
  `false`.

```go
type ListUsersQuery struct {
    Active *bool `form:"active"`
    Page   int   `form:"page"`
}

var q ListUsersQuery
if err := c.ShouldBindQuery(&q); err != nil {
    // 422 con el envelope de error estándar
}
```

- Pasar ese `*bool` al repository, que lo convierte en condición de GORM (ver
  `references/stack-and-structure.md`, sección "Filtros booleanos y valores cero en GORM").
