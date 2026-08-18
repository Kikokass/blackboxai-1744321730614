# CRM SaaS Universal — Arquitectura Base

> Documento de arquitectura previo a cualquier implementación. Cubre los 25 puntos solicitados. Este documento es la referencia de diseño; no sustituye a los ADRs que se abran conforme se tomen decisiones específicas por módulo.

**Estado:** propuesta inicial para discusión y cierre de arquitectura.
**Alcance temporal:** decisiones para MVP (P0/P1) con visión de escalamiento a 10,000+ tenants sin reescritura.

---

## 0. Principio rector

Todo lo que sigue está optimizado para una sola variable dominante: **costo marginal por tenant cercano a cero**, sin sacrificar aislamiento de datos ni seguridad. Esto descarta de entrada: base de datos por cliente, microservicios prematuros, cómputo dedicado por tenant, y llamadas a LLM como mecanismo de cálculo quandoswitch reglas SQL bastan.

---

## 1. Arquitectura completa recomendada

**Monolito modular, multi-tenant, con workers separados y una capa de eventos interna.**

No microservicios. A esta escala (cientos → decenas de miles de tenants pequeños/medianos), microservicios solo añaden latencia de red, complejidad operativa y costo de infraestructura sin beneficio real: no hay equipos separados que necesiten desplegar de forma independiente, y el volumen por tenant es bajo.

Capas:

```
┌─────────────────────────────────────────────────────────┐
│  Cliente (Web PWA responsive, mobile-first)              │
└───────────────┬───────────────────────────────────────────┘
                │ HTTPS / JSON
┌───────────────▼───────────────────────────────────────────┐
│  API / App Server (monolito modular)                     │
│  - Auth & Session                                         │
│  - Módulos de dominio (contacts, deals, tasks, etc.)      │
│  - RBAC + entitlements gate                                │
│  - AI Tool Layer (funciones controladas)                  │
│  - Event Emitter (outbox transaccional)                   │
└───────────────┬─────────────────┬──────────────────────────┘
                │                 │
        ┌───────▼──────┐   ┌──────▼───────────┐
        │  PostgreSQL   │   │  Worker(s) Jobs   │
        │  (RLS multi-  │   │  automations,     │
        │   tenant)     │   │  webhooks, PDFs,  │
        │               │   │  imports, AI      │
        └───────────────┘   └───────────────────┘
                │
        ┌───────▼───────┐
        │ Object Storage │ (archivos, PDFs, adjuntos)
        └────────────────┘
```

Todo el estado vive en Postgres. El monolito modular se organiza internamente por **bounded contexts** (carpetas/paquetes con fronteras claras), de forma que si en el futuro un módulo (p. ej. omnichannel o AI) justifica extraerse como servicio, la frontera ya existe en el código y la extracción es mecánica, no una reescritura.

---

## 2. Stack tecnológico recomendado

| Capa | Elección | Por qué |
|---|---|---|
| Lenguaje | TypeScript en todo el stack | Un solo lenguaje, tipos compartidos cliente/servidor, contratación más fácil, velocidad de desarrollo |
| Frontend | Next.js (React) + Tailwind + shadcn/ui | SSR/PWA, buen soporte mobile-first, ecosistema maduro, componentes accesibles sin licencia |
| Backend | Next.js API routes / Route Handlers o NestJS embebido en el monorepo | Evita mantener dos runtimes; NestJS si el equipo crece y se necesita más estructura (DI, módulos) |
| Base de datos | PostgreSQL (gestionado: Supabase o RDS/Cloud SQL) | RLS nativo, JSONB donde corresponde, extensiones (pg_trgm, pgvector futuro), un solo motor para todo |
| Auth | Supabase Auth o Auth propio sobre Postgres (JWT + refresh) | JWT con claims de tenant_id/rol, integrable con RLS directamente |
| Storage de archivos | Object storage S3-compatible (Supabase Storage / S3 / R2) | Nunca blobs en la base relacional; URLs firmadas, barato por GB |
| Cache | Postgres (vistas materializadas) primero; Redis solo cuando se justifique | No pagar Redis desde el día uno si no hay evidencia de necesidad |
| Colas / Jobs | Cola basada en Postgres (pg-boss / graphile-worker) | Cero infraestructura adicional al inicio; migrar a Redis/BullMQ o SQS solo si el volumen lo exige |
| Búsqueda | Postgres full-text + pg_trgm | Suficiente hasta varios millones de registros; Typesense/Meilisearch como upgrade path documentado, no día uno |
| IA | Anthropic Claude API (modelos por tiers) | Tool-use controlado, prompt caching, buena relación costo/calidad |
| Deploy frontend/API | Vercel o similar (edge + serverless) | Pago por uso, cero servidores que mantener |
| Deploy workers | Contenedor pequeño en Fly.io/Railway/Render, autoescalable | Aislado del request/response path, escala independiente |
| Observabilidad | Sentry (errores) + logs estructurados (Axiom/Better Stack) + Postgres pg_stat_statements | Barato, suficiente para SaaS en esta etapa |
| CDN | El del proveedor de hosting (Vercel/Cloudflare) | Cachea assets estáticos y formularios públicos |

**Regla de oro:** cada pieza adicional de infraestructura (Redis, motor de búsqueda dedicado, cola distribuida) se agrega solo cuando una métrica real lo demanda, nunca por anticipación.

---

## 3. Diagrama conceptual de módulos

```
                         ┌─────────────────────────┐
                         │   CORE (P0 — obligatorio)│
                         │  Auth · Tenancy · RBAC   │
                         │  Contacts · Orgs         │
                         │  Pipelines · Deals        │
                         │  Activities · Tasks       │
                         │  Calendar · Notes · Tags  │
                         │  Custom Fields · Search   │
                         │  Notifications · Audit    │
                         └────────────┬─────────────┘
                                      │ eventos internos
        ┌────────────┬───────────────┼───────────────┬───────────────┐
        ▼            ▼               ▼               ▼               ▼
  ┌──────────┐ ┌───────────┐  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
  │Automation│ │AI Copilot │  │Omnichannel  │ │Commerce      │ │Analytics /  │
  │ Engine   │ │(tool layer│  │(WhatsApp,   │ │(products,    │ │Dashboards / │
  │(P1)      │ │+ chat)(P1)│  │email, etc.) │ │quotes,       │ │Forecast(P1) │
  │          │ │           │  │(P1/P2)      │ │payments)(P1) │ │             │
  └──────────┘ └───────────┘  └─────────────┘ └─────────────┘ └─────────────┘
        │                                                              
        ▼                                                              
  ┌─────────────────────────────────────────────────────────────────┐
  │  Módulos verticales activables por plan/industria (P2/P3)        │
  │  Real Estate (propiedades, match) · Projects · Tickets/Support    │
  │  Customer Success · White-label · Marketplace                     │
  └─────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │  Plataforma transversal: Feature Flags · Entitlements/Billing ·   │
  │  API pública + Webhooks · Import/Export · Super Admin interno      │
  └─────────────────────────────────────────────────────────────────┘
```

Todo módulo fuera del CORE se activa mediante **feature flags por tenant** y depende del CORE, nunca al revés.

---

## 4. Arquitectura multi-tenant

**Modelo elegido: base de datos única, esquema compartido, aislamiento por `tenant_id` + Row Level Security (RLS) en Postgres.**

Descartado:
- Base de datos por tenant → costo operativo y de conexión inmanejable a partir de cientos de tenants, migraciones N veces más lentas.
- Esquema por tenant → mismo problema en menor escala, además complica pooling de conexiones.

Reglas duras:
1. **Toda** tabla de negocio tiene `tenant_id UUID NOT NULL`.
2. RLS habilitado en **todas** las tablas con tenant_id, con política `USING (tenant_id = current_tenant_id())`, donde `current_tenant_id()` lee un claim firmado del JWT de sesión (no un valor enviable por el cliente en el body/query).
3. El backend nunca confía en un `tenant_id` recibido del frontend para lectura/escritura; siempre se deriva de la sesión autenticada.
4. Ningún rol de aplicación usa el `service_role` (bypass de RLS) para servir requests de usuario; ese rol se reserva para jobs internos que ya validan tenant_id explícitamente en cada query.
5. Pool de conexiones compartido (PgBouncer / pooler del proveedor) — el aislamiento vive en RLS, no en conexiones separadas.

Esto hace que una fuga de tenant A → tenant B sea estructuralmente difícil incluso si hay un bug de lógica en el backend (defensa en profundidad).

---

## 5. Arquitectura de base de datos

- Motor: PostgreSQL 15+.
- IDs: UUID v7 (ordenables por tiempo) donde el orden de inserción importa para performance de índices.
- Timestamps: `created_at`, `updated_at` (trigger automático), `deleted_at` (soft delete) en entidades de negocio que el usuario puede "eliminar" pero se auditan.
- Convención: tablas en `snake_case` plural, FKs `*_id`, siempre con `ON DELETE` explícito (`RESTRICT` por defecto, `CASCADE` solo en relaciones de composición real como `quote_items` de `quotes`).
- JSONB: reservado para (a) `custom_field_values` (valores dinámicos), (b) `metadata` de eventos/webhooks, (c) payloads de integraciones externas. **Nunca** como sustituto de columnas relacionales para datos que se filtran, ordenan o agregan con frecuencia.
- Índices: FK siempre indexadas; índices compuestos `(tenant_id, <campo de filtro frecuente>)` en cada tabla de alto volumen (contacts, deals, activities, tasks); `pg_trgm` para búsqueda difusa de nombre/teléfono/email.
- Particionamiento: no en el MVP. Documentado como upgrade path para `audit_logs`, `activities` y `ai_usage_logs` por rango de fecha cuando el volumen lo justifique (>50-100M filas).
- Migraciones: herramienta declarativa versionada (Prisma Migrate, Drizzle Kit o Supabase Migrations) — nunca cambios manuales en ambientes compartidos.

---

## 6. Esquema inicial de tablas (P0)

Resumen de entidades núcleo con sus relaciones clave. (DDL completo se genera junto con las migrations, no aquí.)

**Tenancy & Identidad**
- `tenants` (id, name, slug, industry_template, plan_id, status, created_at)
- `users` (id, email, password_hash, name, avatar_url, mfa_enabled, created_at) — un usuario puede pertenecer a varios tenants
- `memberships` (id, tenant_id, user_id, role_id, status, invited_by, joined_at) — puente users↔tenants
- `roles` (id, tenant_id NULL para roles de sistema, name, is_system)
- `permissions` (id, code, module, description)
- `role_permissions` (role_id, permission_id)

**CRM Core**
- `organizations` (id, tenant_id, name, custom fields…, owner_id)
- `contacts` (id, tenant_id, organization_id NULL, first_name, last_name, phone, whatsapp, email, city, state, country, source_id, owner_id, lifecycle_stage, temperature, last_interaction_at, next_interaction_at, created_at, deleted_at)
- `lead_sources` (id, tenant_id, name, source, medium, campaign, ad, adset)
- `tags` (id, tenant_id, name, color)
- `entity_tags` (id, tenant_id, entity_type, entity_id, tag_id)
- `notes` (id, tenant_id, entity_type, entity_id, author_id, body, created_at)
- `files` (id, tenant_id, entity_type, entity_id, storage_path, filename, mime_type, size, uploaded_by, created_at)

**Pipeline & Ventas**
- `pipelines` (id, tenant_id, name, entity_type, is_default)
- `pipeline_stages` (id, tenant_id, pipeline_id, name, order, probability, color)
- `deals` (id, tenant_id, pipeline_id, stage_id, contact_id, organization_id, name, value, currency, probability, owner_id, source_id, expected_close_date, status, next_activity_id NULL, stage_changed_at, created_at)

**Actividad & Productividad**
- `activities` (id, tenant_id, entity_type, entity_id, type, subject, body, owner_id, occurred_at, created_at) — timeline unificado (llamadas, mensajes, cambios de etapa, etc.)
- `tasks` (id, tenant_id, title, description, owner_id, related_entity_type, related_entity_id, due_at, priority, status, recurrence_rule, created_at)
- `events` (id, tenant_id, title, type, owner_id, related_entity_type, related_entity_id, starts_at, ends_at, location, status, recurrence_rule, created_at) — calendario

**Personalización**
- `custom_fields` (id, tenant_id, entity_type, key, label, field_type, options JSONB, is_required)
- `custom_field_values` (id, tenant_id, custom_field_id, entity_id, value JSONB)

**Plataforma**
- `notifications` (id, tenant_id, user_id, type, title, body, read_at, entity_type, entity_id, created_at)
- `audit_logs` (id, tenant_id, actor_id, action, entity_type, entity_id, metadata JSONB, created_at)
- `domain_events` (id, tenant_id, event_type, entity_type, entity_id, payload JSONB, processed_at NULL, created_at) — outbox, ver §8
- `feature_flags` (id, tenant_id NULL=global, key, enabled, config JSONB)
- `entitlements` (id, tenant_id, plan_id, limits JSONB) — límites por plan (usuarios, contactos, AI tokens/mes, storage)

Entidades de P1+ (automation_workflows, automation_runs, messages, conversations, forms, form_submissions, products, quotes, quote_items, payments, api_keys, webhooks, properties, projects, tickets, ai_usage_logs) se diseñan al abrir cada módulo, siguiendo el mismo patrón (`tenant_id` obligatorio, RLS, auditoría donde toque).

---

## 7. Sistema RBAC

Modelo: **rol → permisos, membership → rol por tenant, permisos con scope de datos.**

- Roles de sistema (no editables): `owner`, `admin`, `manager`, `sales`, `support`, `viewer`.
- Roles personalizados: mismo modelo, `tenant_id` no nulo en `roles`, permisos asignables desde UI en fases posteriores (P2), pero el esquema lo soporta desde P0.
- Permisos con formato `module.action` (ej. `deals.read`, `deals.update`, `contacts.export`, `billing.manage`).
- **Scope de datos**, no solo de acción — cada permiso relevante lleva un scope: `own` (solo lo asignado a mí), `team` (mi equipo), `all` (todo el tenant). Esto resuelve el caso "vendedor solo ve sus leads, gerente ve su equipo, admin ve todo" sin lógica ad-hoc dispersa: una función central `can(user, action, resource)` que resuelve rol + scope + RLS.
- La verificación de permisos ocurre **siempre en backend** antes de tocar la base de datos, y la RLS actúa como segunda barrera (defensa en profundidad) filtrando por tenant y, donde aplique, por scope vía columnas indexadas (`owner_id`, `team_id`).
- El frontend nunca es la fuente de verdad de permisos: solo oculta/muestra UI en base a lo que el backend ya declaró que el usuario puede hacer (entitlements + permisos entregados en la sesión).

---

## 8. Arquitectura del sistema de eventos

**Patrón: Transactional Outbox + worker despachador.**

1. Cuando una operación de negocio cambia estado relevante (ej. `deal.stage_changed`), se inserta una fila en `domain_events` **dentro de la misma transacción** que el cambio de datos. Esto garantiza que nunca hay un evento sin el cambio correspondiente, ni un cambio sin su evento (consistencia atómica sin necesidad de un message broker externo).
2. Un worker (o `LISTEN/NOTIFY` de Postgres + polling de respaldo) lee `domain_events` no procesados y los despacha a los **consumidores internos**: motor de automatizaciones, servicio de notificaciones, agregador de analítica, dispatcher de webhooks.
3. Cada consumidor procesa de forma idempotente (clave `event_id` + `consumer_name` en tabla de control) para tolerar reintentos sin duplicar efectos.
4. Eventos de catálogo inicial: `contact.created`, `contact.updated`, `organization.created`, `deal.created`, `deal.stage_changed`, `deal.won`, `deal.lost`, `task.created`, `task.completed`, `task.overdue`, `activity.created`, `event.created`, `event.cancelled`, `message.received` (P1), `form.submitted` (P1).
5. Esto desacopla CRM ↔ Automations ↔ Notifications ↔ Analytics sin necesitar Kafka/SQS desde el día uno. Si el volumen de eventos crece órdenes de magnitud, el mismo contrato (outbox) permite migrar el despachador a un stream real (NATS/SQS) sin tocar a los productores de eventos.

---

## 9. Arquitectura de automatizaciones

Modelo **Trigger → Conditions → Actions**, persistido como definición declarativa (JSON) por `automation_workflows`, ejecutado por el worker en reacción a `domain_events`.

- `automation_workflows` (id, tenant_id, name, trigger_event, conditions JSONB, actions JSONB, is_active)
- `automation_runs` (id, tenant_id, workflow_id, trigger_event_id, status, started_at, finished_at, error, steps_log JSONB) — trazabilidad completa para depurar por qué algo pasó (o no pasó).

Reglas de robustez, no negociables:
- **Idempotencia por paso**: cada acción lleva una idempotency key derivada de (`workflow_id`, `run_id`, `step_index`) para que un retry no duplique tareas/mensajes.
- **Límite de profundidad/recursión**: una acción que dispara un evento que a su vez podría re-disparar el mismo workflow debe cortar en N iteraciones (ej. máx. 5) para evitar loops infinitos.
- **Rate limiting por tenant**: tope de ejecuciones/mensajes salientes por minuto, para que un workflow mal configurado no dispare miles de WhatsApp de golpe.
- **Delays** (`esperar 24 horas`) se implementan como jobs programados (`run_at`), no como procesos en memoria.
- El builder visual (UI) es P1/P2; el motor de ejecución y el esquema de datos se construyen desde P1 aunque la UI inicial sea plantillas predefinidas simples.

---

## 10. Estrategia de IA y control de costos

**Principio:** la IA es una capa de interpretación/generación sobre datos que ya calculamos con SQL/reglas. Nunca sustituye cálculo determinístico.

- **AI Tool Layer**: conjunto cerrado de funciones (`search_contacts`, `get_contact`, `search_deals`, `create_task`, `create_note`, `schedule_activity`, `get_pipeline_metrics`, etc.) que el modelo puede invocar vía tool-use. Cada función:
  - recibe `tenant_id` y `user_id` inyectados por el backend (nunca provistos por el modelo),
  - aplica el mismo RBAC que cualquier request humano,
  - nunca ejecuta SQL arbitrario generado por el modelo.
- **Model tiering**: tareas de clasificación/extracción simples → modelo pequeño/barato; redacción, resúmenes y chat conversacional → modelo medio; razonamiento complejo (poco frecuente) → modelo top, solo bajo demanda explícita del usuario.
- **Prompt caching** para contexto repetido (system prompt, esquema de herramientas, perfil del tenant).
- **Resultados cacheados** cuando el dato base no cambió (ej. resumen diario se recalcula una vez, no en cada apertura de dashboard).
- **AI asistida, no autónoma en P0/P1**: el asistente de seguimiento sugiere ("¿quieres enviar este seguimiento?"), no dispara mensajes reales sin confirmación humana, hasta que haya suficiente confianza/datos para ofrecerlo como opción avanzada.
- **Tracking de costos**: tabla `ai_usage_logs` (tenant_id, user_id, feature, model, input_tokens, output_tokens, estimated_cost_usd, created_at). Se agrega a `entitlements` un límite mensual de tokens/costo por plan; al acercarse al límite se notifica, al superarlo se degrada a funciones no-IA (nunca se corta el CRM, solo la capa IA).
- Cosas que **no** deben resolverse con IA (van con SQL/reglas): detectar lead abandonado, calcular SLA, lead scoring base, forecast ponderado, alertas de "sin próxima actividad". La IA entra para explicar, redactar o conversar sobre esos resultados, no para calcularlos.

---

## 11. Estrategia de almacenamiento

- Archivos (documentos, fotos, PDFs generados) → **object storage** (S3-compatible), nunca en Postgres.
- La tabla `files` guarda solo metadata + `storage_path`; el contenido vive en el bucket.
- Acceso vía **URLs firmadas de corta duración**, generadas server-side, nunca credenciales de storage expuestas al cliente.
- Buckets separados lógicamente por tipo (`attachments`, `avatars`, `generated-pdfs`, `imports`) con políticas de retención distintas (ej. imports temporales se purgan a los 30 días).
- Límite de tamaño/tipo de archivo validado en el backend antes de emitir la URL de subida (previene abuso de storage).
- Cuotas de storage por tenant como parte de `entitlements`, visibles en el panel de super admin.

---

## 12. Estrategia de caché

Capas, de la más barata a la más cara — se sube de nivel solo si hay evidencia de necesidad:

1. **Cliente**: cache de queries (React Query) con invalidación por mutación — evita refetch innecesario.
2. **Edge/CDN**: assets estáticos y formularios públicos embebibles cacheados en el borde.
3. **Base de datos**: vistas materializadas / tablas de agregación (`deal_stage_summary`, `dashboard_daily_metrics`) refrescadas por jobs programados (cada pocos minutos u horas según la métrica), en vez de recalcular sobre millones de filas en cada request.
4. **Redis** (opcional, introducido cuando el tráfico lo justifique): sesiones de alta frecuencia, rate limiting distribuido, resultados de IA cacheados con TTL.

No se introduce Redis en el MVP — Postgres + cache de cliente cubre el volumen esperado en los primeros miles de tenants.

---

## 13. Estrategia de jobs/workers

- Motor: cola respaldada en Postgres (pg-boss / graphile-worker) — reutiliza la misma base de datos, sin infraestructura adicional.
- Tipos de job iniciales: `dispatch_domain_event`, `run_automation_step`, `send_message` (email/WhatsApp), `generate_pdf`, `import_csv_batch`, `ai_task`, `webhook_delivery`, `notification_fanout`, `analytics_rollup`.
- Garantías: reintentos con backoff exponencial, `idempotency_key` obligatoria por job, cola de **dead letter** para jobs que agotan reintentos (con alerta), métricas de jobs fallidos visibles en super admin.
- El worker corre como proceso separado del API server (contenedor propio), escalable de forma independiente por número de jobs pendientes, no por tráfico HTTP.
- Jobs programados (cron-like) para: recálculo de scoring, detección de leads abandonados/estancados, refresco de vistas materializadas, resumen diario, purga de imports temporales.

---

## 14. Estrategia de observabilidad

- **Errores**: Sentry (o equivalente) en frontend, API y worker, con `tenant_id` y `request_id` como tags para poder filtrar por cliente afectado.
- **Logs estructurados** (JSON) con `request_id`, `tenant_id`, `user_id`, `duration_ms` — centralizados en un servicio barato de logs (Axiom/Better Stack) con retención razonable (ej. 30 días).
- **Métricas de negocio internas**: jobs fallidos, automatizaciones fallidas, errores de API por tenant, latencia P95 de endpoints críticos (pipeline, dashboard).
- **Postgres**: `pg_stat_statements` para detectar queries lentas antes de que se vuelvan un incidente; alertas sobre conexiones cerca del límite del pool.
- **Uptime checks** externos sobre endpoints críticos (login, API pública).
- Todo esto se conecta al **Super Admin interno** (§64 del brief) para tener visibilidad proactiva por tenant antes de que el cliente reporte el problema.

---

## 15. Estrategia de backups

- Backups automáticos diarios completos + **WAL/PITR** (point-in-time recovery) provistos por el motor gestionado de Postgres (Supabase/RDS/Cloud SQL ya lo incluyen).
- Retención: 30 días rodantes como mínimo (ajustable por plan/contrato).
- **Restore drills trimestrales**: restaurar un backup a un ambiente aislado y validar integridad — un backup nunca probado no cuenta como backup confiable.
- Runbook documentado de restauración (quién, cómo, tiempo estimado — RTO/RPO objetivo definidos explícitamente, ej. RPO ≤ 15 min vía PITR, RTO ≤ 2 h).
- Exportaciones de storage (archivos) con la misma política de retención que su bucket.

---

## 16. División P0 / P1 / P2 / P3

**P0 — Imprescindible para operar (MVP funcional):**
Auth, multi-tenancy + RLS, RBAC básico, Contacts, Organizations, Pipelines, Deals, Activities, Tasks, Calendar, Notes, Tags, Custom Fields, Search básico, Notifications, Dashboard/Command Center, Anti-lead-loss (reglas), CSV import, Audit logs, Event system (outbox), Jobs/workers base.

**P1 — Importante, cierra el diferenciador comercial:**
Automatizaciones (motor + plantillas), WhatsApp (Cloud API), Forms, AI Copilot (tool layer + chat), Products, Quotes (con PDF), Payments (registro simple), Atribución de marketing, Lead scoring por reglas, API pública + API keys, Webhooks, Billing/entitlements, Feature flags operativos.

**P2 — Crecimiento:**
Analítica avanzada/forecast con explicación IA, Omnichannel completo (email/SMS/Instagram/Messenger unificado), Telefonía (capa de integración), Real Estate module, Projects module, Tickets/Support, Detección de duplicados + merge, Roles personalizados vía UI, Vistas guardadas avanzadas, Resúmenes automáticos IA.

**P3 — Futuro:**
Marketplace de integraciones/plantillas de terceros, White-label, Benchmarking agregado anónimo, Modelos de scoring con ML, App móvil nativa (si el negocio lo justifica), Integraciones contables externas, Multi-región.

---

## 17. Roadmap técnico de desarrollo

**Fase 0 — Fundación (semanas 1-3):** proyecto monorepo, auth, tenancy + RLS, RBAC, esquema base, CI/CD, observabilidad mínima, entorno de staging.

**Fase 1 — CRM Core (semanas 4-9):** Contacts, Organizations, Pipelines/Deals, Activities/Tasks/Calendar, Notes/Tags, Custom Fields, Search, Dashboard v1.

**Fase 2 — Retención y confiabilidad (semanas 10-13):** Anti-lead-loss, Notifications, Import CSV + de-dup, Audit logs, Event system + Automations motor (con 3-5 plantillas fijas), primeras pruebas de aislamiento multi-tenant automatizadas.

**Fase 3 — Comunicación y ventas (semanas 14-19):** WhatsApp Cloud API, Forms + captura de leads, Products/Quotes/PDF, Payments básico, Atribución de fuente.

**Fase 4 — IA y billing (semanas 20-24):** AI Copilot (tool layer), AI cost tracking + límites, Billing/entitlements reales, API pública + webhooks, Super Admin interno.

**Fase 5 — Verticalización (post-lanzamiento comercial):** Real Estate, Projects, Tickets, Omnichannel ampliado, Onboarding por plantilla de industria, mejoras de analítica/forecast.

Cada fase se cierra con: schema + RLS + backend + permisos + frontend + estados de carga/error/vacío + auditoría + tests + responsive, antes de pasar a la siguiente (Definition of Done, §25).

---

## 18. Dependencias entre módulos

- **RLS/RBAC/Tenancy** es prerequisito de absolutamente todo — nada se construye antes.
- **Event system (outbox)** es prerequisito de Automations, Notifications y Analytics — todos son consumidores de eventos, no deben acoplarse directamente a las tablas de dominio.
- **Automations** depende de: Event system, Tasks, Notifications, y (para acciones de mensajería) del módulo de comunicación correspondiente (WhatsApp/Email).
- **Anti-lead-loss** depende del modelo de datos de Activities/Deals/Tasks (necesita `next_activity`, `last_interaction_at`) — es una capa de reglas sobre datos ya existentes, no una entidad nueva.
- **AI Copilot** depende del AI Tool Layer, que a su vez depende de que Search, Deals, Tasks y Pipeline Metrics ya expongan funciones de consulta reutilizables — no se construye antes que esas funciones existan.
- **Billing/Entitlements** y **Feature Flags** son transversales: todo módulo P1+ debe verificar entitlement antes de activarse; se diseñan junto con el CORE aunque el cobro real llegue después.
- **Módulos verticales** (Real Estate, Projects, Tickets) dependen del CORE + Custom Fields, nunca al revés — no deben filtrarse conceptos de un vertical dentro del core.
- **Omnichannel** depende de que exista un modelo de `conversations`/`messages` genérico antes de conectar el primer canal (WhatsApp), para no rehacer el modelo de datos con cada canal nuevo.

---

## 19. Riesgos técnicos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Falla en política RLS deja ver datos de otro tenant | Crítico (fuga de datos, pérdida de confianza) | RLS obligatoria + tests automatizados específicos "Tenant A no puede leer/escribir Tenant B" en cada PR que toque el schema |
| Automatización en loop / mensajería masiva accidental | Alto (costo, reputación en WhatsApp) | Límite de profundidad, rate limiting por tenant, dry-run antes de activar workflow nuevo |
| Costo de IA fuera de control | Alto (margen del negocio) | Tracking por request, límites por plan, model tiering, alertas tempranas |
| Custom fields como EAV degradando performance | Medio | Índices sobre `custom_field_values`, límite razonable de campos por entidad, evaluar materialización para reportes pesados |
| Búsqueda lenta a gran volumen | Medio | Postgres FTS + trigram cubre el rango esperado; path de migración a Typesense documentado, no urgente |
| Rate limits / cambios de política de WhatsApp Cloud API | Medio | Aislar la integración detrás de una interfaz propia, no acoplar el dominio a la API externa |
| Migraciones de schema riesgosas en producción multi-tenant | Alto | Migraciones reversibles, columnas nuevas con default/nullable primero, despliegue sin downtime, staging obligatorio |
| Importaciones CSV generando duplicados masivos | Medio | De-dup por teléfono/email antes de insertar, previsualización + confirmación del usuario |
| Crecimiento de `audit_logs`/`domain_events` sin control | Medio | Particionamiento por fecha + purga/archivado planificado desde el diseño de la tabla |
| Dependencia de un solo proveedor gestionado (Postgres/Storage) | Medio | Elegir proveedor con export/restore estándar (pg_dump, S3-compatible) para evitar lock-in real |

---

## 20. Formas de reducir costos de infraestructura

- Un solo Postgres multi-tenant compartido en vez de N bases — el mayor ahorro estructural.
- Serverless/edge para el API (pago por uso) en vez de servidores siempre encendidos.
- Cola de jobs sobre Postgres en vez de Redis/SQS hasta que el volumen lo exija.
- Vistas materializadas y agregaciones periódicas en vez de recalcular todo en cada request.
- Modelos de IA pequeños por defecto, grandes solo bajo demanda explícita; prompt caching.
- Storage por metadata + object storage, nunca blobs en la base relacional (evita inflar el costo de backups de la DB).
- CDN para todo lo estático y para formularios/páginas públicas embebibles.
- Connection pooling (pgbouncer/pooler gestionado) para no pagar por miles de conexiones activas.
- Purga/archivado programado de datos de bajo valor a largo plazo (logs viejos, imports temporales).
- Autoscaling de workers por cola pendiente, no por reloj — se apagan solos en horas valle.

---

## 21. Estimación conceptual de recursos por escala

Cifras orientativas para planeación, no cotización — dependen del proveedor elegido y patrón de uso real.

**100 tenants** (asumiendo ~20 usuarios y algunos miles de contactos/deals por tenant en promedio):
- Postgres: instancia pequeña/mediana gestionada, low-tens de GB.
- Workers: 1 instancia pequeña, baja concurrencia.
- IA: gasto dominado por adopción del Copilot, tiering mantiene el costo bajo por tenant.
- Orden de magnitud de infraestructura base (sin IA ni mensajería): rango bajo, de decenas a pocos cientos de USD/mes.

**1,000 tenants:**
- Postgres: instancia mediana con pooler, cientos de GB, posible réplica de lectura para reportes/analítica.
- Workers: 2-3 instancias con autoscaling por cola.
- Necesidad de vistas materializadas para dashboards ya es real, no opcional.
- Orden de magnitud: rango medio, cientos a bajo-miles de USD/mes.

**10,000 tenants:**
- Postgres: instancia grande + réplicas de lectura, particionamiento activo en tablas de alto volumen (activities, audit_logs, domain_events), monitoreo de queries obligatorio.
- Considerar motor de búsqueda dedicado si la UX de búsqueda global lo exige a ese volumen.
- Workers con autoscaling agresivo, posible separación de colas por prioridad (mensajería vs. reportes).
- Orden de magnitud: miles a bajo-decenas de miles de USD/mes — aun así, muy por debajo de cualquier esquema de infraestructura dedicada por tenant, que a este volumen sería inviable.

En los tres escenarios, el costo por tenant **decrece** con la escala porque la infraestructura compartida se amortiza — validación directa de la decisión de arquitectura multi-tenant sobre base compartida.

---

## 22. Qué NO deberíamos desarrollar inicialmente

- Infraestructura de telefonía propia (usar proveedores vía integración).
- Apps móviles nativas (PWA responsive cubre el caso de uso hasta que haya evidencia de necesidad real).
- Motor de BI/cubos analíticos complejos — dashboards + reportes predefinidos bastan al inicio.
- Contabilidad/ERP completo — solo registro de ingresos/pagos suficiente para indicadores.
- Marketplace de integraciones de terceros.
- White-label completo (multi-dominio, branding dinámico) — se deja preparado conceptualmente, no construido.
- Scoring por Machine Learning — reglas primero, ML cuando haya suficiente dato histórico propio para entrenar algo mejor que las reglas.
- Motor de búsqueda dedicado (Typesense/Meilisearch) — Postgres FTS/trigram basta al inicio.
- Multi-región activo-activo.
- RBAC con permisos 100% granulares por campo — el modelo rol+scope del §7 cubre los casos reales del MVP.
- Roles personalizados vía UI (el esquema los soporta, pero la UI de creación es P2).

---

## 23. Propuesta detallada del MVP

**Objetivo del MVP:** un negocio puede registrarse, invitar a su equipo, configurar su pipeline, gestionar contactos y oportunidades, y **nunca perder un seguimiento**, con un dashboard que le diga qué hacer hoy — sin ninguna dependencia de IA ni de integraciones externas de mensajería todavía.

Incluye exactamente el set **P0** del §16:
1. Auth + Multi-tenancy + RLS
2. Users + Memberships + RBAC básico (roles de sistema)
3. Contacts + Organizations
4. Pipelines + Pipeline Stages + Deals
5. Activities (timeline) + Tasks + Calendar/Events
6. Notes + Tags
7. Custom Fields (texto, número, moneda, fecha, selección)
8. Búsqueda global (Postgres FTS)
9. Notifications (in-app)
10. Command Center / Dashboard ("Mi día", "Negocio", "Prioridades" con reglas simples)
11. Anti-lead-loss por reglas (sin próxima actividad, sin contactar, estancado)
12. Import CSV con mapeo de columnas y de-dup básico por teléfono/email
13. Audit logs
14. Event system (outbox) + worker base
15. Onboarding guiado (organización → giro/plantilla → pipeline → invitar equipo → importar)

Explícitamente **fuera** del MVP: WhatsApp, IA, automatizaciones visuales (se puede lanzar con 2-3 automatizaciones fijas de sistema, no un builder), cotizaciones/pagos, módulos verticales, API pública.

Criterio de éxito del MVP: un equipo comercial real puede migrarse desde Excel/WhatsApp suelto y trabajar su semana completa (prospectar, dar seguimiento, agendar, cerrar) sin salir de la plataforma.

---

## 24. Estructura recomendada del repositorio

Monorepo, para compartir tipos y reducir fricción de coordinación siendo un equipo pequeño:

```
/apps
  /web            → Next.js (frontend + API routes del monolito)
  /worker         → proceso de jobs (automations, webhooks, PDFs, imports, AI tasks)
/packages
  /db             → schema, migrations, políticas RLS, seeds
  /core           → lógica de dominio compartida (servicios, casos de uso), independiente de framework HTTP
  /rbac           → definición de roles/permisos/scope, función can()
  /events         → tipos de domain events + contratos de payload
  /ai-tools       → AI Tool Layer (funciones invocables por el modelo)
  /ui             → design system (componentes, tokens, tema claro/oscuro)
  /config         → constantes compartidas, entitlements por plan, feature flags
/infra
  /migrations-ci  → pipeline de validación de migraciones
  /iac            → definición de infraestructura (si aplica: Terraform/Pulumi)
/docs
  ARCHITECTURE.md → este documento
  adr/            → Architecture Decision Records por decisión relevante
  runbooks/       → restauración de backups, incident response
```

Reglas: `core` y `db` no dependen de `apps/web` (dirección de dependencia única, hacia adentro); `worker` y `web` consumen `core`/`db`/`events`, nunca al revés.

---

## 25. Definition of Done global del proyecto

Una funcionalidad **no está terminada** hasta que cumple, cuando aplique:

- [ ] Modelo de datos con migration versionada y reversible.
- [ ] `tenant_id` presente y política RLS activa en toda tabla nueva.
- [ ] Permisos RBAC definidos (acción + scope) y verificados en backend.
- [ ] Test explícito de aislamiento: "Tenant A no puede leer/modificar datos de Tenant B".
- [ ] Validación de entrada y manejo de errores en backend (no solo en frontend).
- [ ] Emite/consume los domain events correspondientes si cambia estado relevante.
- [ ] Frontend con estados de carga, vacío y error (no solo el "happy path").
- [ ] Responsive / usable desde celular.
- [ ] Auditoría (`audit_logs`) si la acción es sensible (financiera, permisos, eliminación, exportación).
- [ ] Respeta feature flags / entitlements del plan del tenant.
- [ ] Sin TODOs, datos mock permanentes, IDs hardcodeados ni bypass de seguridad.
- [ ] Tests automatizados de la lógica crítica (no solo verificación visual manual).
- [ ] Documentado donde aporte valor real (no documentación por documentar).

---

## Siguiente paso propuesto

Con la arquitectura cerrada, el siguiente entregable natural es el **ADR de Fase 0**: setup del monorepo, elección final de proveedor gestionado (Supabase vs. RDS/Cloud SQL + Auth propio) con comparación de costo/control, y el primer schema migrable (`tenants`, `users`, `memberships`, `roles`, `permissions`) con RLS desde el primer commit. No se escribe código de producto hasta cerrar esa decisión de proveedor, porque condiciona Auth, Storage y RLS simultáneamente.
