# Arquitectura

Visión técnica de cómo está construida La Asignatura Pendiente.

## Visión general

Aplicación monolítica desplegada en Vercel con Astro 7 en modo SSR. El frontend y el backend viven en el mismo repositorio: las páginas son componentes Astro y la API son rutas de servidor bajo `src/pages/api/`. La base de datos es PostgreSQL gestionada en Neon, accedida mediante Drizzle ORM con el driver serverless.

```
Navegador
   │
   ▼
Astro 7 (SSR en Vercel, Node 24)
   ├── Páginas .astro            ← frontend, HTML por defecto
   ├── API Routes (93)           ← backend TypeScript
   │     └── lib/server/         ← lógica de negocio y utilidades
   ├── Vercel Cron (3 diarios)   ← purgas, avisos de lista de espera, talleres
   ▼
Neon PostgreSQL (Drizzle ORM, 43 tablas)

Servicios externos: Stripe · Resend · Anthropic · Google (Gemini, OAuth, GA4, Search Console) · Vercel Blob
```

## Organización del backend

Las API routes están agrupadas por dominio mediante prefijos en el nombre del fichero:

| Prefijo             | Rutas | Dominio                                                         |
| ------------------- | ----- | --------------------------------------------------------------- |
| `auth-*`            | 11    | Registro, login, Google OAuth, verificación, refresh tokens     |
| `payments-*`        | 5     | Sesiones de pago, vista previa de precio, cupones, suscripción  |
| `webhooks-*`        | 2     | Eventos de Stripe y de entregas de Resend                       |
| `products-*` / `bundles-*` | 3 | Catálogo y packs                                            |
| `content-*`         | 4     | Entrega de contenido al alumno: módulo, material, progreso, eventos |
| `community-*`       | 38    | Toda la comunidad: posts, comentarios, DM, encuestas, push, moderación |
| `taller-*` / `talleres-*` | 2 | Inscripción y material de los talleres en directo            |
| `waitlist-*` / `newsletter-*` / `test-mapa-mental-*` | 6 | Captación con doble confirmación |
| `ai-*`              | 2     | Endpoints de los dos agentes de IA                              |
| `admin-*`           | 9     | Estudio de contenido (subida, YouTube, seguimiento), estadísticas y export |
| `user-*`            | 8     | Perfil, compras, talleres, lista de espera, exportación y borrado de cuenta |
| `cron-*`            | 3     | Trabajos programados, protegidos por secreto                    |

## Capa de servidor (`lib/server/`)

La regla de la casa: **la lógica que decide algo de dinero, acceso o seguridad se extrae a una función pura con tests**, y el endpoint solo la orquesta. Algunos ejemplos:

- `webhook-compras.ts`: qué accesos debe crear un pago y cuáles ya existen (idempotencia por acceso esperado).
- `contenido-acceso.ts` / `contenido-venta.ts`: desbloqueo de días por calendario y regla de "solo se vende completa".
- `confirmacion.ts` / `confirmacion-admin.ts`: la barrera que decide si una acción del agente fue confirmada de verdad.
- `crisis.ts`: detector determinista para el chat público.
- `filtro-urls.ts`: red que elimina enlaces y correos que no son del sitio.
- `circulo-stripe.ts`: orden de operaciones entre Stripe y la base en altas y bajas de la comunidad.
- `correo-entregas.ts`: estado de entregas de Resend sin guardar destinatario ni asunto.

Y las utilidades transversales: `db.ts` (Neon con pooling), `jwt.ts`, `password.ts`, `cors.ts`, `ratelimit.ts`, `middleware.ts`, `stripe.ts`, `resend.ts`, `web-push.ts`, `email-templates.ts` y `schema.ts`.

## Base de datos

- **43 tablas** que cubren usuarios y tokens, catálogo (productos, módulos, packs, talleres), compras y cupones, progreso y eventos de contenido, comunidad (espacios, posts, comentarios, reacciones, encuestas, DM, notificaciones, suscripciones push, bloqueos), captación (lista de espera, newsletter, confirmaciones por correo, entregas), memoria del agente de gestión, ejecuciones de cron y rate limits.
- **36 migraciones** versionadas con Drizzle Kit. Un guardarraíl propio escanea las migraciones en cada despliegue: bloquea las que suponen pérdida de datos irreversible si no están marcadas como revisadas, y avisa de las que podrían bloquear una tabla en producción.
- Entornos develop y producción como ramas de Neon. Las migraciones de producción se aplican en el build de Vercel solo cuando el entorno es `production`, seguidas de la sincronización del catálogo con Stripe.
- Las migraciones de datos delicadas se ensayan antes contra una copia de producción.

## Frontend

- Astro renderiza HTML por defecto; el JavaScript de cliente es opt-in (Island Architecture), lo que mantiene el peso bajo y favorece Core Web Vitals.
- Tailwind CSS para utilidades, CSS custom properties para la paleta de marca y fuentes autoalojadas con la API de fuentes de Astro (sin CDN de terceros).
- Zonas diferenciadas: web pública indexable, área de alumno (`/aprende`), cuenta (`/mi-cuenta`), comunidad (`/el-circulo/app`) y panel de gestión (`/admin`), cada una protegida por middleware.
- SEO técnico: canónicas, sitemap generado, JSON-LD escapado, migas de pan, imagen para compartir y `llms.txt`.
- La comunidad funciona como PWA instalable con notificaciones push.

## Calidad y proceso

- **TypeScript estricto** y `astro check` en todo el proyecto.
- **42 ficheros de test** con `node:test`, centrados en la lógica pura donde un fallo cuesta dinero o expone datos.
- **Baterías propias para los agentes**: guardarraíles (comprobaciones duras de seguridad) y calidad de respuestas, ejecutadas contra el modelo real.
- **CI en GitHub Actions**: typecheck, Biome sin avisos, tests, build sin variables locales y guardarraíl de migraciones.
- Desarrollo por fases con especificación de criterios de aceptación, decisiones registradas como ADR y verificación al cierre de cada fase.

## Despliegue

- Vercel con Node 24.
- Build personalizado: migraciones de producción, sincronización con Stripe, build y aviso de URLs nuevas a buscadores (IndexNow).
- Tres trabajos de Vercel Cron diarios.
- Variables de entorno gestionadas en el dashboard de Vercel, nunca en el repositorio.
