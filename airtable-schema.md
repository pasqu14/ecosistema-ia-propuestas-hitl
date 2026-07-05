# Esquema de la base de datos (Airtable) — "Cerebro" del sistema

La base tiene **dos tablas relacionadas** para evitar datos aislados: `Solicitudes` (registro de
cada propuesta y su estado) y `Base_Conocimiento` (memoria privada para RAG).

## Tabla 1 — `Solicitudes` (centro de comando)

| Campo | Tipo | Uso |
|-------|------|-----|
| Cliente | Single line text | Nombre del prospecto |
| Sector | Single select | Sector del cliente (usado por el RAG y el prompt) |
| Email Cliente | Email | Destino final de la propuesta |
| Datos del Cliente | Long text | Input dinámico: necesidades, contexto, notas del vendedor |
| **Estado** | Single select | `Pendiente` / `Generando` / `Revisión pendiente` / `Aprobado por Humano` / `Enviado` / `Error` |
| Borrador Propuesta | Long text | Donde la IA escribe la propuesta |
| **Aprobado** | Checkbox | Semáforo HITL. Por defecto `false` |
| Revisor | Collaborator | Quién valida |
| Log Error | Long text | Se rellena si la API de IA falla (ruta de error) |
| Gmail Thread ID | Single line text | Para mantener el hilo de correo organizado |
| Última modificación | Last modified time | Campo que usan los triggers de n8n |
| Directrices aplicadas | Link to `Base_Conocimiento` | **Relación** entre tablas |

### Ciclo de estados
`Pendiente → Generando → Revisión pendiente → (Aprobado) → Enviado`
(rama de error: cualquier estado → `Error`)

## Tabla 2 — `Base_Conocimiento` (RAG — información atómica)

Un tema por registro (no un manual kilométrico), para que la recuperación sea precisa.

| Campo | Tipo | Uso |
|-------|------|-----|
| Título | Single line text | Nombre del fragmento |
| Categoria | Single select | `Voz de marca` / `Precios` / `Casos de éxito` / `Servicios` |
| Sector | Single select | Sector al que aplica (para filtrar por relevancia) |
| Contenido | Long text | El fragmento de conocimiento validado |
| Solicitudes | Link to `Solicitudes` | **Relación** inversa |

## Relaciones

`Solicitudes.Directrices aplicadas  <-->  Base_Conocimiento.Solicitudes`

Esto conecta cada propuesta con los fragmentos de conocimiento que la fundamentaron —
evita datos aislados y da trazabilidad de qué directrices usó la IA.

## Vista en modo lectura para la entrega

Crear una vista compartida (Share → "Anyone with the link", solo lectura) y pegar el enlace
en el README y en el formulario de entrega.
