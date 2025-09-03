# Bloque 1 - Etapa 1: Desarrollo API de Tracking con Checkpoints

## ADR-001 — Arquitectura de la **Tracking & Checkpoints API**

**Fecha:** 2025-09-02  
**Estado:** Aprobada (para MVP)  
**Alcance:** Servicio REST de tracking que ingesta y persiste checkpoints inmutables por unidad, expone consultas de historial y último estado, y lista unidades por estado.  
**Stack:** NestJS, Azure (Front Door, App Gateway, APIM, App Service/AKS, Key Vault, Monitor), PostgreSQL, Redis (cache), Service Bus (opcional), OpenTelemetry.

---

## 1) Contexto

- El sistema recibe solicitudes desde clientes externos (Courier App, Partner Carrier, Backoffice) para **registrar checkpoints** y **consultar** tracking.
- La API corre en Azure detrás de **Front Door (WAF)** → **App Gateway (WAF)** → **APIM** (validación JWT/OAuth2), alojada en **App Service o AKS**, con conectividad privada a **PostgreSQL** y **Redis**, secretos en **Key Vault**, y **observabilidad centralizada** (Azure Monitor / OpenTelemetry).
- Requisitos clave:
  - Checkpoints **inmutables**; auditoría completa del historial.
  - **Autenticación/autorización** via JWT/OAuth2 (Azure Entra ID) con validación vía JWKS.
  - **Trazas, métricas y logs** estándar (OpenTelemetry).
  - **Disponibilidad** y **seguridad perimetral** (WAFs, TLS extremo a extremo).
  - **Escalabilidad** horizontal de la API y del data store (r/w).
- Estilo arquitectónico interno: **Arquitectura Hexagonal** (Ports & Adapters) en NestJS.

---

## 2) Decisiones

### D1 — Framework y estilo

- **Elegimos NestJS** por su modularidad, inyección de dependencias y soporte nativo para controladores, pipes, guards, interceptors, y testing.
- **Arquitectura Hexagonal** para aislar dominio (use cases, entidades) de detalles de infraestructura (DB, auth, clock, telemetry), mejorando testabilidad y mantenibilidad.

**Consecuencias**

- - Facilita pruebas unitarias/integración y cambios de infraestructura.
- – Curva de aprendizaje para el equipo si no domina ports/adapters.

---

### D2 — Despliegue y perímetro en Azure [Diagrama de despliegue](https://s.icepanel.io/paayESblF9Rgip/kfkV/landscape/diagrams/viewer?diagram=lz9867gxqd&model=d56aw9p8yu&overlay_tab=tags&x1=-634.1&y1=-64&x2=2514.1&y2=1456)

- **Front Door (WAF)** en el borde para CDN/routing global y protección L7.
- **Application Gateway (WAF)** para terminación TLS y protección adicional en la “perimeter VNet”.
- **API Management (APIM)** para publishing, throttling, validación de JWT, cuotas y observabilidad de API.
- **App Service** (PaaS) como opción por defecto; **AKS** si se requiere control fino de runtime o sidecars.
- **VNet + Subnets** (DMZ/App/Data) y **Private Endpoints** para PostgreSQL/Redis/Key Vault.

**Consecuencias**

- - Defense-in-depth, integración nativa con Entra ID y Managed Identity.
- – Mayor complejidad operativa y costo que un despliegue PaaS minimalista.

---

### D3 — Identidad y seguridad

- **Azure Entra ID** como IdP. **APIM** valida **JWT** (RS256) vía **JWKS**.
- La API usa **AuthGuard** en NestJS (scopes/roles) para autorización fina.
- **TLS** extremo a extremo; **Managed Identity** para acceder a Key Vault y otros recursos.
- **Rate limiting** y **quotas** en APIM; **idempotencia** a nivel de dominio para comandos de checkpoint.

**Consecuencias**

- - Seguridad robusta, revocación centralizada, menor manejo de secretos en app.
- – Requiere coordinación con tenancies/consentimiento de apps externas.

---

### D4 — Persistencia

- **PostgreSQL** como sistema transaccional; modelo diseñado para:
  - **Shipment** (guía/paquete), **Unit** (unidad identificada por código de barras), **Checkpoint** (evento inmutable con estado, timestamp, actor, source).
- **Índices** por `trackingId`, `unitBarcode`, y `latest_state` materializado (vista/tabla auxiliar) para consultas rápidas.
- **Transacciones** y **constraints** para asegurar inmutabilidad y consistencia (no actualizaciones de checkpoints; solo inserts).

![Alt text](assets/ME-R.png 'ME-R')

**Consecuencias**

- - ACID, queries flexibles, fácil auditoría.
- – Costo de almacenamiento histórico; se planifica **particionamiento** por tiempo para grandes volúmenes.

---

### D5 — Cache y colas (opcionales según carga)

- **Redis** para cachear “último estado por unidad/tracking” y resultados de consultas frecuentes.
- **Service Bus** para integrar **webhooks firmados** y/o fan-out de eventos a consumidores externos sin acoplar la API síncrona.

**Consecuencias**

- - Latencias menores y desac acoplamiento de integraciones.
- – Inval idación de cache y consistencia eventual requieren disciplina.

---

### D6 — Observabilidad

- **OpenTelemetry** (trazas, métricas, logs) exportado a **Azure Monitor / Application Insights**.
- **Interceptors** de NestJS para logging estructurado, correlación (trace/span), latencias y códigos HTTP.
- Alertas en **Defender for Cloud** para postura de seguridad y en Monitor para SLOs.

**Consecuencias**

- - Triaging rápido de incidentes; base para SLO/Error Budgets.
- – Costos de ingestión y necesidad de muestreo en picos.

---

### D7 — Entrega y operaciones

- **CI/CD** con pipelines (build, unit/integration tests, lint, SCA, IaC plan/apply).
- **Infra as Code** (Bicep/Terraform) para VNet, App Service/AKS, APIM, Key Vault, PostgreSQL, Redis.
- **Blue-green/Canary** (App Service slots o AKS + Ingress) detrás de APIM.

**Consecuencias**

- - Deploys previsibles y reversibles.
- – Requiere madurez del pipeline e IaC.

---

## 3) Diseño lógico (Hexagonal en NestJS) [C4 Model diagrams](https://s.icepanel.io/paayESblF9Rgip/kfkV)

**Inbound adapters**

- `TrackingController`, `ShipmentsController`, `CheckpointsController`.
- `AuthGuard (JWT)`; `ValidationPipe/DTOs`; `Logging & Tracing Interceptor`.

**Application (Use cases)**

- `CheckpointCommandService` (registro idempotente de checkpoint).
- `TrackingQueryService` (historial y último estado).
- `ShipmentQueryService` (listar unidades por estado).

**Domain**

- Entidades/Aggregates: `Shipment`, `Unit`, `Checkpoint`.
- Dom. services: `StatusValidator`, `IdempotencyService`.
- **Ports**: `UnitRepository`, `ShipmentRepository`, `CheckpointRepository`, `AuthPort` (opcional), `ClockPort`, `IdempotencyRepository`.

**Outbound adapters**

- `PgClient` / repositorios PostgreSQL.
- `ConfigService` (Key Vault via Managed Identity).
- `JwtAuthProvider` (validaciones delegadas cuando aplique).
- `TelemetryService` (OpenTelemetry).
- `ClockSystem`.
- (Opcional) `Webhook adapter` y `Service Bus adapter`.

---

## 4) Atributos de calidad y cómo se cumplen

| Atributo           | Decisiones que lo soportan                                                                                                                     |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Seguridad**      | WAF (Front Door/App Gateway), APIM con validación JWT/JWKS, TLS E2E, Private Endpoints, Managed Identity, Key Vault, RBAC, Defender for Cloud. |
| **Disponibilidad** | App Service/AKS con escalado, APIM con throttling y health checks, PostgreSQL flexible server HA, Redis réplica, despliegues blue-green.       |
| **Escalabilidad**  | Escalado horizontal de la API, cache Redis para hot paths, particionamiento en DB, colas para trabajos asíncronos.                             |
| **Observabilidad** | OpenTelemetry + Azure Monitor, logs estructurados, métricas SLI/SLO, alertas.                                                                  |
| **Mantenibilidad** | Hexagonal + NestJS modular, separación de dominio/infra, pruebas unitarias/integración, IaC.                                                   |
| **Auditoría**      | Checkpoints inmutables, historial completo, trazas con correlación.                                                                            |
| **Rendimiento**    | Índices adecuados, cache de último estado, DTOs ligeros, compresión en Front Door/APIM.                                                        |

---

## 5) API (MVP, alto nivel)

- **POST** `/checkpoints` — registra checkpoint inmutable (idempotencia por `(unitId,eventId|clientToken)`).
- **GET** `/tracking/{trackingId}` — historial completo y último estado.
- **GET** `/units?state={state}` — lista unidades por estado (paginado).
- **Seguridad:** `Authorization: Bearer <JWT>`; scopes/roles aplicados por guard.

---

## 6) Datos y reglas clave

- **Unit** debe existir para aceptar un `Checkpoint`.
- `Checkpoint` **no se actualiza ni borra**; solo **insert**.
- “Último estado” se resuelve por **max(timestamp)** por `unitId` (o vista materializada/campo derivado mantenido por trigger/job).
- Integridad referencial estricta entre `Shipment` → `Unit` → `Checkpoint`.

---

## 7) Alternativas consideradas

1. **Monolito sin APIM ni doble WAF**
   - - Menor costo/operación.
   - – Menor seguridad/observabilidad/gestión de ciclo de vida de API. **Rechazada**.

2. **NoSQL (Cosmos DB) para eventos**
   - - Escala de ingesta muy alta, TTLs.
   - – Complejidad para queries relacionales del tracking; consistencia y JOINS. **Rechazada para MVP**.

3. **Serverless Functions** en lugar de App Service
   - - Cost-effective en burst, rápido de iterar.
   - – Latencia fría y manejo de conexiones a DB; necesidad de APIM igual. **Rechazada para paths síncronos críticos**.

---

## 8) Riesgos y mitigaciones

- **Crecimiento de historial** → particionamiento temporal y archivado a Storage; índices compuestos; VACUUM/ANALYZE.
- **Hot key en “último estado”** → caching con TTL, invalidación en write path.
- **Rotación de claves/certificados** → Key Vault + versiones + alertas; Managed Identity.
- **Backpressure** en picos de ingesta → cuotas APIM, colas (Service Bus) para offload, autoscaling.

---

**Aprobado por:** Lina Beltrán  
**Vigencia:** Este ADR aplica al MVP y será revisado después de la primera release GA o ante cambios de requisitos no funcionales.
