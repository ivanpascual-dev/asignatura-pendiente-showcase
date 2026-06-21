# Patrones de código

Ejemplos **ilustrativos** de los patrones técnicos usados en el proyecto. No son el código fuente real (privado por confidencialidad del cliente), sino versiones genéricas que muestran cómo están resueltas las decisiones clave.

## 1. Webhook con verificación de firma e idempotencia

Ningún evento de pago se procesa hasta validar su firma. Después se comprueba que no se haya tratado ya, para que un reintento de Stripe no duplique el efecto.

```ts
export async function POST({ request }: APIContext) {
  const signature = request.headers.get('stripe-signature');
  const payload = await request.text();

  // 1. Verificar firma ANTES de tocar nada
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(payload, signature!, WEBHOOK_SECRET);
  } catch {
    return new Response('Firma inválida', { status: 400 });
  }

  // 2. Idempotencia: si ya procesamos este evento, salimos
  const yaProcesado = await db.query.processedEvents.findFirst({
    where: eq(processedEvents.id, event.id),
  });
  if (yaProcesado) return new Response('OK (duplicado)', { status: 200 });

  // 3. Procesar y registrar el evento como tratado
  await procesarEvento(event);
  await db.insert(processedEvents).values({ id: event.id });

  return new Response('OK', { status: 200 });
}
```

## 2. Tool de un agente de IA con esquema validado

Cada capacidad del negocio es una tool con su entrada validada por Zod. El modelo decide cuándo invocarla, pero nunca puede salirse del esquema.

```ts
import { tool } from 'ai';
import { z } from 'zod';

const getProgramas = tool({
  description: 'Devuelve el catálogo de programas con sus precios.',
  inputSchema: z.object({
    soloActivos: z.boolean().default(true),
  }),
  execute: async ({ soloActivos }) => {
    const programas = await db.query.products.findMany({
      where: soloActivos ? eq(products.active, true) : undefined,
    });
    return programas.map((p) => ({
      slug: p.slug,
      nombre: p.name,
      precio: p.price,
    }));
  },
});
```

## 3. Autorización: filtrar siempre por la identidad del token

El `userId` se extrae del JWT verificado, nunca de un parámetro que envíe el cliente. Así un usuario no puede pedir datos de otro manipulando la petición.

```ts
export async function GET({ request }: APIContext) {
  const userId = await getUserIdFromToken(request); // del JWT verificado
  if (!userId) return new Response('No autorizado', { status: 401 });

  // La query SIEMPRE se acota al userId del token, no a un id del body/query
  const compras = await db.query.purchases.findMany({
    where: eq(purchases.userId, userId),
  });

  return Response.json(compras);
}
```

## 4. Tokens separados por nivel de riesgo

El access token (corta vida) viaja al cliente; el refresh token (larga vida) se guarda en una cookie httpOnly inaccesible desde JavaScript, lo que mitiga el robo por XSS.

```ts
function setRefreshCookie(headers: Headers, refreshToken: string) {
  headers.append(
    'Set-Cookie',
    [
      `refresh_token=${refreshToken}`,
      'HttpOnly',         // inaccesible desde JS del navegador
      'Secure',           // solo por HTTPS
      'SameSite=Strict',  // no se envía en peticiones cross-site
      'Path=/api/auth',
      `Max-Age=${60 * 60 * 24 * 30}`,
    ].join('; '),
  );
}
```
