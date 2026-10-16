# Seguridad

Decisiones de seguridad aplicadas en La Asignatura Pendiente y el porqué de cada una.

## Webhooks

- **Verificación de firma antes de procesar.** Ningún evento de Stripe ni de Resend se trata hasta validar su firma. Un payload sin firma válida, o un aviso de Resend con más de 5 minutos, se rechaza sin tocar la base.
- **Idempotencia por acceso esperado.** El webhook de Stripe no pregunta "¿ya se procesó este pago?", sino "¿existen ya todos los accesos que este pago debía crear?". Así un reintento completa lo que faltara tras un fallo parcial sin duplicar nada, con una restricción única en base como última defensa.
- **Una reentrega no deshace un reembolso.** Los eventos llegan desordenados; el estado solo avanza en el sentido correcto.

## Pagos

- **Un solo cálculo de precio.** Lo que se muestra en el checkout y lo que se cobra salen de la misma función, con tests. Antes divergían en varios sitios.
- **Stripe antes que la base** en altas y bajas de la comunidad: si Stripe falla, no se revoca ni se concede en local, y se dice. Las condiciones de carrera de la suscripción están cerradas a nivel de base de datos.
- **No se vende lo que no se puede entregar.** Una práctica de 21 días sin sus días cargados se rechaza en el checkout.
- **Consentimiento de desistimiento** expreso antes de dar acceso inmediato a contenido digital.

## Autenticación y sesión

- **Tokens separados por riesgo.** Access token de vida corta y refresh token en una cookie httpOnly inaccesible desde JS, guardado como hash en base.
- **Rotación de refresh tokens** en cada renovación de sesión, con revocación global por usuario.
- **Contraseñas con bcrypt** y salt rounds 12.
- **Google OAuth** verificado en servidor con la librería oficial de Google.
- **Verificar una cuenta exige el token del correo y una prueba de haber elegido la contraseña.** Sin esa segunda condición, quien registrara el correo de otra persona con su propia contraseña conservaría el acceso en cuanto ella pulsara el enlace. Salió de una auditoría de seguridad interna.
- **Errores genéricos**: el login no revela si un email existe.

## Autorización de datos

- **Filtrado por identidad del token.** Todas las queries usan el `user_id` del JWT verificado, nunca un id recibido en el body o la query string.
- **Cada endpoint de contenido repite la comprobación completa** (módulo activo, producto, compra completada, día desbloqueado). Una cuenta borrada con el token aún vivo recibe 403.
- **Gates de comunidad.** El acceso a El Círculo se valida en servidor contra la suscripción real y la aceptación de las normas.

## Contenido protegido

- **Ni vídeos ni materiales en el HTML inicial.** Se piden por API tras verificar el acceso; el reproductor se crea al pulsar reproducir.
- **Audios y PDF en un almacén privado**, separado del público de la comunidad, servidos con URL firmada de vida corta.
- **Subidas acotadas**: solo admin, solo tipos de audio y PDF, tamaño máximo y ruta dentro de la carpeta del producto.

## Superficie y configuración

- **Cabeceras de seguridad y CSP por zona.** La zona pública admite lo mínimo para la analítica; la privada (alumno, cuenta, pago, comunidad, admin) no la admite. Un test deriva la lista de rutas privadas del mismo sitio que decide dónde se mide, para que no se desincronicen.
- **Fuentes autoalojadas**, sin CDN de terceros en la CSP.
- **JSON-LD escapado** siempre por una única función, porque nombres y precios vienen de base y pueden cambiarse desde el chat.
- **CORS restringido** al dominio de producción.
- **Rate limiting** persistido en base de datos sobre auth, cuenta, contenido e IA.
- **Crons protegidos por secreto**, comparado en tiempo constante para no filtrar información por temporización, y vigilados: si la purga deja de ejecutarse, el sistema avisa.
- **Secretos solo en el entorno de Vercel**, nunca commiteados.

## Privacidad y RGPD

- **Exportación y borrado de cuenta** autoservicio; el borrado anonimiza y elimina progreso y actividad, conservando solo lo que exige la ley fiscal.
- **Consentimientos separados** (privacidad obligatoria, marketing opcional), exigidos en servidor.
- **Doble confirmación** en lista de espera, newsletter y test; lo no confirmado se purga a los 7 días.
- **Analítica solo tras aceptar**: GA4 no se carga antes del consentimiento, nunca en zona privada, con un catálogo cerrado de eventos y URLs sin parámetros.
- **Logs sin datos personales**: las entregas de correo se guardan sin destinatario ni asunto, y los logs de auditoría del agente no llevan emails ni contenido.

## Agentes de IA

- **Barreras de código, no solo instrucciones.** La confirmación de acciones sensibles se verifica contra el historial que el propio servidor envió; el modelo no puede autoconfirmarse.
- **Protocolo de crisis** determinista en el chat público: sin tools en ese turno y mensaje de recursos garantizado por código.
- **Filtro de enlaces**: nada que no sea del propio sitio llega al visitante.
- **Datos externos tratados como no confiables**: lo que viene de Google, GA4 o de nombres de usuarios pasa por listas cerradas o se marca como dato.
- **Separación de capacidades.** El agente público es de solo lectura; las tools con escritura solo existen en el endpoint de administración.
- **Baterías de pruebas** de guardarraíles y de calidad contra el modelo real.
