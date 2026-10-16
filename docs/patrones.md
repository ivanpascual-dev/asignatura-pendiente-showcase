# Patrones de código

Ejemplos **ilustrativos** de los patrones técnicos usados en el proyecto. No son el código fuente real (privado por confidencialidad del cliente), sino versiones genéricas que muestran cómo están resueltas las decisiones clave.

## 1. Webhook con verificación de firma e idempotencia por acceso esperado

Ningún evento se procesa hasta validar su firma. Después, en vez de preguntar "¿ya procesé este pago?", se pregunta "¿qué accesos debía crear este pago y cuáles faltan?". Si un intento anterior falló a medias, el reintento de Stripe completa solo lo que falta. La decisión es una función pura, testeada sin base de datos.

```ts
// Lógica pura: qué filas faltan por crear
export function accesosQueFaltan(esperados: string[], yaCreados: string[]): string[] {
  const creados = new Set(yaCreados);
  return esperados.filter((productId) => !creados.has(productId));
}

export async function POST({ request }: APIContext) {
  const payload = await request.text(); // body crudo: la firma se calcula sobre él
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(payload, request.headers.get('stripe-signature')!, WEBHOOK_SECRET);
  } catch {
    return new Response('Firma inválida', { status: 400 });
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    const esperados = productosDeLaSesion(session);
    const yaCreados = await accesosExistentes(session.id);
    const faltan = accesosQueFaltan(esperados, yaCreados);

    // payment_id único por sesión y producto: la base es la última defensa contra duplicados
    await db.insert(purchases).values(
      faltan.map((productId) => ({ paymentId: `${session.id}_${productId}`, productId, userId: session.metadata!.userId })),
    ).onConflictDoNothing();
  }

  return new Response('OK', { status: 200 });
}
```

## 2. Confirmación de acciones del agente impuesta en código

Pedirle al modelo "confirma antes de cobrar" en el prompt no basta: bajo presión ("hazlo ya, sin preguntarme") puede autoconfirmarse. La barrera mira el historial que el propio servidor envió: solo se ejecuta si antes hubo una vista previa de **esa misma acción con los mismos parámetros** y, después, un mensaje real del usuario. Dentro de un bucle de varios pasos el modelo solo produce mensajes de asistente y de tool, nunca de usuario, así que no puede fabricar la segunda condición.

```ts
type Mensaje = { role: 'user' | 'assistant' | 'tool'; toolName?: string; args?: unknown; result?: unknown };

const firma = (args: Record<string, unknown>) =>
  JSON.stringify(Object.fromEntries(
    Object.entries(args).filter(([k, v]) => k !== 'confirmado' && v !== undefined).sort(),
  ));

export function fueConfirmadoDeVerdad(historial: Mensaje[], tool: string, args: Record<string, unknown>): boolean {
  const i = historial.findLastIndex(
    (m) => m.role === 'tool' && m.toolName === tool && firma(m.args as any) === firma(args)
      && (m.result as any)?.needs_confirmation === true,
  );
  if (i === -1) return false;                                   // nunca hubo vista previa
  return historial.slice(i + 1).some((m) => m.role === 'user'); // y el usuario respondió después
}

// Dentro de la tool
execute: async (args, { messages }) => {
  if (!args.confirmado || !fueConfirmadoDeVerdad(messages, 'grant_access', args)) {
    return { needs_confirmation: true, preview: `Dar acceso GRATUITO a ${args.slug} para ${args.email}` };
  }
  return concederAcceso(args);
}
```

Las tools que necesitan la barrera están en una lista cerrada, y un test falla si alguien añade una tool nueva sin clasificarla.

## 3. Tool de un agente de IA con esquema validado

Cada capacidad del negocio es una tool con su entrada validada por Zod. El modelo decide cuándo invocarla, pero nunca puede salirse del esquema, y la tool solo devuelve los campos que el modelo necesita.

```ts
import { tool } from 'ai';
import { z } from 'zod';

const getProgramas = tool({
  description: 'Programas y prácticas activos, con precio y disponibilidad.',
  inputSchema: z.object({}),
  execute: async () => {
    const productos = await db.query.products.findMany({ where: eq(products.active, true) });
    return productos.map((p) => ({
      nombre: p.name,
      precio: p.price,
      disponibilidad: p.enListaEspera ? 'lista_espera' : 'venta',
      url: p.enListaEspera ? null : `/pago/${p.slug}`, // sin url de compra si no se vende
    }));
  },
});
```

## 4. Autorización: filtrar siempre por la identidad del token

El `userId` se extrae del JWT verificado, nunca de un parámetro que envíe el cliente. Así un usuario no puede pedir datos de otro manipulando la petición.

```ts
export async function GET({ request }: APIContext) {
  const auth = await verifyAuth(request); // JWT verificado
  if (isAuthError(auth)) return auth;     // 401

  // La query SIEMPRE se acota al userId del token, no a un id del body/query
  const compras = await db.query.purchases.findMany({
    where: eq(purchases.userId, auth.userId),
  });

  return Response.json(compras);
}
```

## 5. Tokens separados por nivel de riesgo

El access token (vida corta) viaja al cliente; el refresh token (larga vida) se guarda en una cookie httpOnly inaccesible desde JavaScript, se rota en cada uso y en base solo se guarda su hash.

```ts
function setRefreshCookie(headers: Headers, refreshToken: string) {
  headers.append(
    'Set-Cookie',
    [
      `refresh_token=${refreshToken}`,
      'HttpOnly',      // inaccesible desde JS del navegador
      'Secure',        // solo por HTTPS
      'SameSite=Lax',  // no viaja en peticiones cross-site que modifican estado
      'Path=/',
      `Max-Age=${60 * 60 * 24 * 30}`,
    ].join('; '),
  );
}
```
