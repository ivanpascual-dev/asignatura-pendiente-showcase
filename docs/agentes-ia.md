# Agentes de IA

La Asignatura Pendiente tiene dos agentes de IA en producción, construidos con el **Vercel AI SDK**. No son chatbots de respuestas fijas: cada uno dispone de un conjunto de *tools* (funciones) que el modelo decide cuándo invocar para consultar o modificar datos reales del negocio.

## Por qué AI-First

El proyecto se diseñó con la IA como capa de gestión, no como un extra. La consecuencia práctica: no hay paneles CRUD para el negocio. La dueña gestiona ventas, precios, accesos, comunidad y talleres conversando con un agente, y el visitante recibe atención sin intervención humana.

La frontera está documentada: lo que es **contextual y visual** (moderar la comunidad, cargar 21 días de vídeos, audios y PDF) se hace a la vista, en la propia comunidad y en un estudio de contenido. Todo lo demás, por chat.

---

## Agente público: NEBULA.ia

Asistente de atención al visitante, disponible 24/7 en la web pública.

- **Modelo:** Gemini 3.1 Flash Lite (Google, vía Vercel AI SDK).
- **Rol:** orientar hacia el programa, la práctica o el taller adecuado, explicar la comunidad y resolver preguntas sobre precios, acceso y formato, siempre con datos leídos en el momento.
- **Tools (5, solo lectura):**
  - `get_programs`: programas, prácticas y talleres abiertos, con precio y disponibilidad (a la venta o en lista de espera).
  - `get_faq`: preguntas frecuentes.
  - `get_community_info`: precio, estado y normas de El Círculo.
  - `get_module_preview`: temario de un programa, sin contenido protegido.
  - `get_content_details`: el texto completo y literal de la página de un producto, para responder con profundidad sin inventar.
- **Barreras de código, no solo de prompt:**
  - **Protocolo de crisis.** Un detector determinista revisa el último mensaje antes de llamar al modelo. Si detecta una crisis, el turno se queda **sin tools** (recomendar un producto pasa de prohibido a imposible) y el código añade el mensaje de recursos escrito por la psicóloga, también si el proveedor falla a mitad de respuesta. Del turno solo se registra la IP, nunca el texto.
  - **Filtro de enlaces.** Todo enlace o correo de la respuesta que no sea del propio sitio se elimina antes de llegar al visitante. Se añadió porque las pruebas demostraron que el modelo fabricaba dominios plausibles pese a la instrucción.
  - **Límite de pasos** y rate limiting por IP.
- **Memoria compartida:** la dueña puede enseñar reglas a NEBULA desde su propio chat; se cargan en el prompt en cada petición.

---

## Agente de gestión: MENTOR.IA

El sistema operativo del negocio. La dueña administra todo conversando, desde el panel `/admin` autenticado.

- **Modelo:** Claude Haiku (Anthropic, vía Vercel AI SDK).
- **Rol:** ejecutar la operativa completa del negocio a partir de lenguaje natural.
- **Tools (36)**, agrupadas por área:
  - **Ventas y usuarios**: estadísticas por periodo, búsqueda de usuarios, historial de compras, conceder o revocar acceso.
  - **Cupones y precios**: crear, listar y modificar cupones, cambiar precios, sincronizar el catálogo con Stripe.
  - **Contenido**: listar módulos y seguir el progreso de los alumnos (la carga se hace en el estudio visual).
  - **Comunidad**: conceder o revocar acceso (cancelando en Stripe si paga), ver miembros, cambiar el precio, abrir o cerrar altas.
  - **Talleres**: inscritas, estado de la sala y del material, y guardar la sala con reprogramación automática de los recordatorios.
  - **Captación**: métricas de newsletter, lista de espera y resultados agregados del test del mapa mental (nunca nombres ni correos).
  - **Web**: Search Console (clics, posición, indexación del sitemap) y GA4 (visitas, eventos, embudo del test).
  - **Comunicación y memoria**: enviar emails, atajos del panel, lecciones que el agente recuerda entre sesiones y aviso al desarrollador cuando se pide algo que no existe.
- **Diseño de seguridad:**
  - Solo accesible con sesión autenticada y rol admin, verificado una vez en el endpoint.
  - **13 tools exigen confirmación** (las que escriben, cobran, envían o exportan). La primera llamada devuelve una vista previa; la ejecución solo cuenta como confirmada si en el historial existe esa vista previa con los mismos parámetros **y** un mensaje real de la dueña posterior. El modelo no puede confirmarse a sí mismo dentro de un mismo turno.
  - Las listas de tools con y sin confirmación son cerradas y un test falla si se añade una tool sin clasificar o una pierde la barrera.
  - Lo que devuelven Google y GA4 pasa por listas cerradas antes de llegar al modelo, porque es texto que cualquiera puede escribir.
  - Los nombres de terceros se devuelven marcados como dato, nunca como instrucción, y los logs de auditoría no llevan datos personales.

---

## Decisiones de diseño

- **Dos proveedores distintos a propósito.** El agente público usa un modelo rápido y económico porque atiende volumen y solo lee; el de gestión usa Claude por su fiabilidad ejecutando tools con efectos sobre datos reales.
- **Barreras en código antes que reglas en el prompt.** Cuando una regla protege dinero o personas (confirmar un cobro, no vender a alguien en crisis, no inventar un enlace), se impone en el servidor. El prompt conserva la instrucción, pero el código es la red.
- **Tools como contrato.** Cada capacidad del negocio es una tool con su esquema de entrada validado (Zod), lo que mantiene al modelo dentro de lo permitido y hace el sistema auditable.
- **Separación lectura/escritura por agente.** El público nunca toca datos; el de gestión requiere autenticación y confirmación verificada.
- **Probado contra el modelo real.** Dos baterías propias (guardarraíles y calidad de respuestas) se ejecutan contra los modelos en producción, incluidos ataques del tipo "hazlo ya, sin preguntarme".
