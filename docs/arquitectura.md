# Arquitectura

Visión técnica de cómo está construida La Asignatura Pendiente.

## Visión general

Aplicación monolítica desplegada en Vercel con Astro en modo SSR. El frontend y el backend viven en el mismo repositorio: las páginas son componentes Astro y la API son rutas de servidor bajo `src/pages/api/`. La base de datos es PostgreSQL gestionada en Neon, accedida mediante Drizzle ORM con el driver serverless.

```
Navegador
   │
   ▼
Astro (SSR en Vercel, Node 24)
   ├── Páginas .astro            ← frontend, HTML por defecto
   ├── API Routes (~70)          ← backend TypeScript
   │     └── lib/server/         ← utilidades reutilizables
   ▼
Neon PostgreSQL (Drizzle ORM, 35 tablas)

Servicios externos: Stripe · Resend · Anthropic · Google · Vercel Blob
```

## Organización del backend

Las API routes están agrupadas por dominio mediante prefijos en el nombre del fichero:

| Prefijo        | Dominio                                                  |
| -------------- | ------------------------------------------------------- |
| `auth-*`       | Registro, login, OAuth, verificación, refresh tokens    |
| `payments-*`   | Creación de pagos, validación de cupones, precios        |
| `webhooks-*`   | Recepción de eventos de Stripe                           |
| `products-*`   | Catálogo de programas, retos y meditaciones              |
| `bundles-*`    | Packs de productos                                       |
| `community-*`  | Toda la comunidad: posts, comentarios, DM, encuestas... |
| `ai-*`         | Endpoints de los dos agentes de IA                       |
| `admin-*`      | Estadísticas y export para el panel de administración    |
| `user-*`       | Perfil, compras, exportación y borrado de cuenta         |
| `newsletter-*` | Alta y baja de la newsletter                             |

## Capa de utilidades (`lib/server/`)

Lógica compartida que evita duplicación entre rutas:

- `db.ts`: conexión a Neon (siempre vía pooling).
- `jwt.ts`: firma y verificación de tokens con jose.
- `password.ts`: hashing y verificación con bcrypt.
- `cors.ts`: cabeceras CORS restringidas al dominio de producción.
- `ratelimit.ts`: rate limiting persistido en base de datos.
- `middleware.ts`: resolución de sesión y protección de rutas.
- `stripe.ts` / `resend.ts` / `web-push.ts`: clientes de servicios externos.
- `schema.ts`: esquema completo de Drizzle (35 tablas).
- `community*.ts`: lógica de la comunidad (gates de acceso, DM, notificaciones).
- `ai-*.ts`: prompts y tools de los agentes (ver [agentes-ia.md](agentes-ia.md)).

## Base de datos

- **35 tablas** que cubren usuarios, productos, módulos, compras, cupones, comunidad (espacios, posts, comentarios, reacciones, encuestas, mensajes directos, notificaciones), newsletter y rate limits.
- Migraciones versionadas con Drizzle Kit.
- Entornos separados develop y producción como ramas de Neon; las migraciones de producción se aplican automáticamente en el build de Vercel solo cuando el entorno es `production`.

## Frontend

- Astro renderiza HTML por defecto; el JavaScript de cliente es opt-in (Island Architecture), lo que mantiene el peso bajo y favorece Core Web Vitals.
- Tailwind CSS para utilidades, con CSS custom properties para la paleta de marca.
- La comunidad funciona como PWA instalable con notificaciones push.

## Despliegue

- Vercel con Node 24 y Fluid Compute.
- Build personalizado que ejecuta las migraciones de producción antes de compilar.
- Variables de entorno gestionadas en el dashboard de Vercel, nunca en el repositorio.
