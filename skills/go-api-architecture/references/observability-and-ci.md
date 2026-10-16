# Observabilidad y CI

Cómo saber qué pasa en producción y cómo mantener la calidad. El nivel de observabilidad se elige
**al inicio** (pregunta 7 de `SKILL.md`); aquí está qué implica cada uno.

## Los tres pilares

| Señal | Qué responde | Librería en la app |
|---|---|---|
| **Logs** | "¿qué pasó?" (eventos) | `rs/zerolog` (JSON, ya en el stack) |
| **Métricas** | "¿cuántas? ¿qué tan rápido?" (números) | `prometheus/client_golang` |
| **Trazas** | "¿por dónde fue esta petición?" (recorrido) | `go.opentelemetry.io/otel` |

**Siempre** se generan logs + `/health` + `/ready`. Las **métricas** desde el nivel `LIGERA`. Las
**trazas** son **opcionales** (útiles en sistemas multi-servicio): en una API de un solo servicio
aportan poco sobre logs + métricas y añaden coste (SDK, colector, almacenamiento).

## Niveles (elegir uno; default `LIGERA`)

| Nivel | Backend | RAM en el VPS | Para quién |
|---|---|---|---|
| `NINGUNA` | `docker logs` / Dokploy | 0 | Proyecto personal mínimo |
| `LIGERA` | **Uptime Kuma** | ~150 MB | VPS chico: uptime + alertas |
| `CLOUD` | **Grafana Cloud** (la app empuja) | ~0 | Dashboards sin gastar RAM |
| `COMPLETA` | Grafana + Prometheus + Loki (overlay) | 1-2 GB | Server con holgura / multi-servicio |

Regla: `NINGUNA`/`LIGERA` para VPS chico; `CLOUD` si quieres dashboards gratis sin RAM; `COMPLETA`
solo con holgura. `COMPLETA` es un **overlay opt-in**, nunca en el compose base.

## `NINGUNA`

- Logs JSON a stdout; se ven con `docker logs -f <contenedor>` o en la UI de Dokploy.
- Configura **rotación** en `compose.yml` para acotar disco:
  ```yaml
  services:
    api:
      logging:
        driver: json-file
        options: { max-size: "10m", max-file: "3" }
  ```
- `/health` y `/ready` para chequeos manuales o un monitor externo gratuito.

## `LIGERA` (Uptime Kuma)

- Añade el monitor como **overlay** (`compose.observability.yml`) y levántalo con
  `docker compose -f compose.yml -f compose.observability.yml up -d` (o `make obs-up`):
  ```yaml
  services:
    uptime-kuma:
      image: louislam/uptime-kuma:1
      restart: unless-stopped
      volumes: ["uptime-kuma:/app/data"]
      ports: ["3001:3001"]
  volumes:
    uptime-kuma: {}
  ```
- En su panel, crea monitores HTTP a `GET /health` (espera `200`) y configura una **notificación**
  (Telegram, Discord, email, webhook) para saber de una caída en menos de un minuto.

## `CLOUD` (Grafana Cloud)

- **No corre nada en el VPS**: la app expone `/metrics` y **empuja** métricas/logs por
  OTLP/remote-write a Grafana Cloud (tier gratuito). Requiere token + endpoint por variables de
  entorno (nunca en el repo).
- Ideal cuando no hay RAM para contenedores: dashboards y alertas viven en la nube.

## `COMPLETA` (stack LGTM self-hosted)

- Overlay `compose.observability.yml` con **Grafana + Prometheus + Loki**: Prometheus scrapea
  `/metrics`, Grafana es el **panel único** para las tres señales.
- Incluye dashboards de **golden signals** y alertas (incluida la **saturación del pool**).

## Instrumentación en la app

- **Métricas**: expón `/metrics` con `promhttp.Handler()`; añade latencia por ruta y contador de
  errores. **El endpoint NO debe ser público**: bíndalo a red interna o protégelo con auth.
- **Correlación**: incluye `request_id` (y `trace_id` si hay trazas) en cada log.
- **Trazas** (opcional): inicializa el SDK de OTel y exporta por OTLP al colector.

### Golden signals y alertas mínimas

- **Latencia** (p50/p95/p99), **tráfico** (RPS), **errores** (tasa 5xx) y **saturación**
  (conexiones del pool vs. `MaxOpenConns`, CPU/RAM).
- Alerta si: la tasa de 5xx supera un umbral, p99 de latencia alta, o el pool se satura.

## CI

Pipeline mínimo que debe pasar antes de mergear (`.github/workflows`):

- `golangci-lint run`
- `go test ./... -race`
- `go build ./...`
- `govulncheck ./...` (vulnerabilidades)
- (si aplica) job de migraciones contra una DB efímera.

Documenta estos comandos en `AGENTS.md`.
