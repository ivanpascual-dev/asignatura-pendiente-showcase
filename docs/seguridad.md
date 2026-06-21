# Seguridad

Decisiones de seguridad aplicadas en La Asignatura Pendiente y el porqué de cada una.

## Webhooks de pago

- **Verificación de firma antes de procesar.** Ningún evento de Stripe se trata hasta validar su firma con el secreto del webhook. Un payload sin firma válida se rechaza de inmediato.
- **Idempotencia.** Antes de conceder un acceso o registrar un pago se comprueba si ese evento ya fue procesado, evitando duplicados si Stripe reintenta la entrega.

## Autenticación y sesión

- **Tokens separados por riesgo.** El access token vive en memoria de JavaScript (corta duración) y el refresh token en una cookie httpOnly inaccesible desde JS, lo que mitiga el robo por XSS.
- **Rotación de refresh tokens** en cada renovación de sesión.
- **Contraseñas con bcrypt** y salt rounds 12.
- **Google OAuth** verificado en servidor con la librería oficial de Google.
- **Verificación de email** obligatoria y flujo de recuperación de contraseña con tokens de un solo uso.

## Autorización de datos

- **Filtrado por identidad del token.** Todas las queries usan el `user_id` extraído del JWT verificado, nunca un id recibido en el body o la query string. Un usuario no puede pedir datos de otro manipulando parámetros.
- **Gates de comunidad.** El acceso a la comunidad y a sus espacios se valida en servidor contra la suscripción real del usuario.

## Contenido protegido

- **URLs de vídeo nunca en el HTML inicial.** Los identificadores de YouTube se sirven mediante fetch autenticado solo a quien ha comprado el contenido, evitando que aparezcan en el código fuente de la página.

## Superficie y configuración

- **CORS restringido** al dominio de producción.
- **Secretos solo en el entorno de Vercel**, nunca commiteados.
- **Rate limiting** persistido en base de datos sobre los endpoints sensibles (auth, pagos, IA).
- **TypeScript estricto** en todo el backend para reducir errores en tiempo de ejecución.

## Agentes de IA

- **Guardarraíles propios.** Las entradas a los agentes se validan y las acciones con efectos (escritura, envío de emails, cambios de precio o accesos) están acotadas, de modo que el modelo no pueda salirse de lo permitido. Existe una suite de pruebas dedicada a estos guardarraíles.
- **Separación de capacidades.** El agente público es de solo lectura; las tools con capacidad de escritura solo están disponibles para el agente de administración autenticado.
