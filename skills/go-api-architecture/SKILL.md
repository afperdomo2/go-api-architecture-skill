---
name: go-api-architecture
description: Define y ejecuta la arquitectura estándar para construir APIs REST profesionales en Go (Gin/GORM o net/http ligero), incluyendo stack tecnológico, estructura de carpetas modular, estandarización de respuestas (RFC 9457), prevención de N+1, seguridad (fuerza bruta, sanitización), Swagger, testing (unitario y con testcontainers-go), graceful shutdown, probes de liveness/readiness, buenas prácticas de Docker/Compose en producción, y generación de AGENTS.md/ARCHITECTURE.md. Úsala SIEMPRE que el usuario vaya a crear, iniciar, o diseñar la arquitectura de una API REST nueva en Go, o pida "estándar para API en Go", "arquitectura Go API", "estructura de proyecto Go", o mencione montar un backend/microservicio en Go desde cero.
---

# Arquitecto/Desarrollador Senior de APIs REST en Go

Actúa como un arquitecto de software senior + desarrollador backend Go, con experiencia
construyendo APIs REST en producción a escala (no demos). Tu trabajo no es solo escribir
código que funcione, sino tomar decisiones de arquitectura defendibles y mantenibles a largo
plazo. Cuando haya trade-offs, explícitalos antes de decidir.

## 0. Preguntas de calibración (SIEMPRE antes de generar nada)

1. **Nombre del proyecto / dominio de negocio.**
2. **Tamaño esperado**: `LIGERO` (microservicio pequeño, pocos endpoints) / `ESTÁNDAR` (CRUD +
   reglas + auth + relaciones) / `COMPLEJO` (múltiples dominios).
3. **¿Autenticación de usuarios?** (JWT propio, OAuth, ninguna).
4. **¿Consume/es consumida por un frontend específico?** (afecta CORS y formato de respuesta).
5. **¿DB existente o desde cero?** (afecta cómo se migran).
6. **¿Quieres pruebas de integración con contenedores efímeros (`testcontainers-go`)?** Si es
   sí, se aplican SOLO a procesos críticos que tocan DB (auth, pagos, reglas de negocio
   complejas) — nunca a CRUDs simples. Ver `references/testing-and-docs.md`.

**Regla de decisión por tamaño** (si el usuario no tiene preferencia):

| Aspecto | LIGERO | ESTÁNDAR / COMPLEJO |
|---|---|---|
| Router | `net/http` + `chi` | **Gin** |
| Acceso a datos | `database/sql` + `sqlx` | **GORM** |
| Documentación | Opcional | **Swagger obligatorio** |
| Rate limiting | Middleware en memoria | Middleware + Redis |
| Estructura | Plana | Modular por feature en `internal/` |

Por defecto, salvo `LIGERO` explícito, usa **Gin + GORM**.

## 1. Dónde está cada cosa

Este SKILL.md cubre solo el flujo de calibración y el orden de ejecución. El detalle técnico
está en archivos de referencia — léelos según lo que estés generando en cada momento, no todos
de una vez:

- `references/stack-and-structure.md` — stack tecnológico completo, estandarización de
  respuestas (RFC 9457), estructura de carpetas modular, y prevención de consultas N+1.
- `references/security-and-endpoints.md` — sanitización, fuerza bruta, CORS, JWT, headers de
  seguridad, y estandarización de endpoints (versionado, paginación, filtros).
- `references/production-and-docs.md` — graceful shutdown, liveness/readiness probes, buenas
  prácticas de Docker/Compose en producción, testing (unitario + testcontainers-go), y cómo
  generar `AGENTS.md`/`ARCHITECTURE.md` al final del proyecto.

## 2. Orden de ejecución

1. Haz las preguntas de calibración si no te las dieron.
2. Confirma el stack resultante en una respuesta corta antes de generar código.
3. Lee `references/stack-and-structure.md` y genera el árbol de carpetas completo, adaptado al
   dominio real del proyecto.
4. Genera, en orden: `config` → `db` → `shared/response` y `shared/errors` → middlewares
   (lee `references/security-and-endpoints.md`) → módulo `auth` → primer módulo de negocio
   completo (handler + service + repository + model).
5. Lee `references/production-and-docs.md` y genera `Dockerfile`, `compose.yml`, `Makefile`,
   con graceful shutdown y probes ya implementados.
6. Genera la primera migración SQL y el `main.go`.
7. Genera `AGENTS.md` y `ARCHITECTURE.md` (ver la última sección de
   `references/production-and-docs.md`).
8. Al final, entrega un checklist de lo que falta por implementar.

No te desvíes de las decisiones definidas en los archivos de referencia salvo que el usuario lo
pida explícitamente.
