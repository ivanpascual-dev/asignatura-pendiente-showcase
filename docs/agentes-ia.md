# Agentes de IA

La Asignatura Pendiente tiene dos agentes de IA en producción, construidos con el **Vercel AI SDK**. No son chatbots de respuestas fijas: cada uno dispone de un conjunto de *tools* (funciones) que el modelo decide cuándo invocar para consultar o modificar datos reales del negocio.

## Por qué AI-First

El proyecto se diseñó con la IA como capa de gestión, no como un extra. La consecuencia práctica: no hay paneles CRUD ni formularios de administración. La dueña gestiona el negocio entero conversando con un agente, y el visitante recibe atención sin intervención humana.

---

## Agente público: NEBULA.ia

Asistente de atención al visitante, disponible 24/7 en la web pública.

- **Modelo:** Gemini 2.5 Flash (vía Vercel AI SDK).
- **Rol:** recomendar el programa o reto adecuado, explicar la comunidad y el acompañamiento 1:1, y resolver preguntas frecuentes sobre precios, acceso y formato.
- **Tools (5, solo lectura):**
  - `get_programs`: catálogo de programas y retos con precios.
  - `get_faq`: preguntas frecuentes.
  - `get_community_info`: información de la comunidad "El Círculo".
  - `get_module_preview`: vista previa de los módulos de un programa.
  - `get_content_details`: detalle de un contenido concreto.
- **Diseño de seguridad:** todas las tools son de solo lectura. El agente no puede modificar nada.

---

## Agente de gestión: ALBI.ADMIN

El sistema operativo del negocio. La dueña administra todo conversando, desde el panel `/admin` autenticado.

- **Modelo:** Claude Haiku 4.5 (Anthropic, vía Vercel AI SDK).
- **Rol:** ejecutar la operativa completa del negocio a partir de lenguaje natural.
- **Tools (29):** agrupadas por área:
  - **Datos y análisis**: productos, módulos, estadísticas de ventas por periodo, historial de compras, métricas de newsletter, export completo en JSON.
  - **Gestión de usuarios**: búsqueda, historial de compras, conceder o revocar acceso a productos.
  - **Cupones y descuentos**: crear, listar, desactivar, reactivar y modificar cupones.
  - **Precios y productos**: cambiar precios, sincronizar con Stripe.
  - **Contenido**: añadir módulos, activar/desactivar, editar títulos, descripciones y recursos.
  - **Comunicación**: enviar emails a usuarios, gestionar atajos del panel.
  - **Comunidad**: conceder o revocar acceso, ver miembros, cambiar el precio de suscripción.
- **Diseño de seguridad:** solo accesible para sesión autenticada con rol admin. Las acciones con efectos están acotadas por guardarraíles propios, con una suite de pruebas dedicada (ver [seguridad.md](seguridad.md)).

---

## Decisiones de diseño

- **Dos proveedores distintos a propósito.** El agente público usa un modelo rápido y económico (Gemini Flash) porque atiende volumen y solo lee; el de gestión usa Claude por su fiabilidad ejecutando tools con efectos sobre datos reales.
- **Tools como contrato.** Cada capacidad del negocio es una tool con su esquema de entrada validado (Zod), lo que mantiene al modelo dentro de lo permitido y hace el sistema auditable.
- **Separación lectura/escritura por agente.** Es la frontera de seguridad principal: el público nunca toca datos, el de gestión requiere autenticación.
