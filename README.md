# Ecosistema de Automatización IA — Propuestas Comerciales con HITL

Entrega Final del curso. Pipeline autónomo que genera propuestas comerciales personalizadas
con IA a partir de una base de conocimientos privada (RAG), con un punto de validación humana
(Human-in-the-Loop) antes de enviar cualquier propuesta al cliente.

## Stack integrado (5 tecnologías)

| Rol | Tecnología |
|-----|-----------|
| Orquestador | **n8n** |
| Base de datos / memoria | **Airtable** |
| Procesamiento IA | **Google Gemini** (`gemini-2.0-flash`, tier gratis) con RAG |
| Canal de salida | **Gmail** |
| Notificación HITL | **Gmail** (al revisor) |

## Caso de uso

Automatización de propuestas comerciales personalizadas. Un vendedor carga los datos de un
prospecto en Airtable y marca la fila como *Generando*. El sistema:
1. Recupera las directrices privadas de marca/precios (RAG).
2. Redacta la propuesta con Claude usando solo datos validados.
3. Guarda el borrador y **espera aprobación humana**.
4. Solo tras el visto bueno, envía la propuesta al cliente por Gmail.

## Arquitectura del flujo

```
[Airtable Trigger: Estado=Generando]
        -> [IF: Estado=Generando y Datos presentes]
        -> [Airtable Search: RAG directrices privadas]
        -> [Aggregate: consolidar directrices]
        -> [Set: preparar variables dinámicas]
        -> [HTTP: Claude genera propuesta]  --(error)--> [Airtable: registrar error]
        -> [Set: extraer content[0].text]
        -> [Airtable: guardar borrador, Estado=Revisión pendiente, Aprobado=false]
        -> [Gmail: notificar al revisor con enlace]   === PAUSA HITL ===

[Airtable Trigger: fila modificada]
        -> [IF: Aprobado==true (booleano) y Estado=Revisión pendiente]
        -> [Gmail: enviar propuesta al cliente (Thread ID)]
        -> [Airtable: Estado=Enviado]
```

## Requisitos de arquitectura cubiertos

- **Disparo y procesamiento sin intervención manual** hasta el punto HITL.
- **Rutas de error**: el nodo de Claude usa `onError: continueErrorOutput`; si la API falla,
  el flujo escribe el error en la DB (`Estado=Error`, `Log Error`) en vez de romperse (directiva Resume).
- **Human-in-the-Loop**: checkbox `Aprobado` en Airtable + notificación al revisor. Nada se
  envía al cliente sin `Aprobado=true`.
- **Nodos nombrados**, **variables dinámicas** (`={{ ... }}`) y **sin datos hardcodeados**
  (base, credenciales y emails salen de variables de entorno / campos de la DB).

## Checks de seguridad

- **Sin bucles infinitos**: los triggers filtran por estado; al pasar a `Enviado` la fila deja
  de cumplir el filtro y no vuelve a dispararse.
- **Comparación de tipos correcta**: el filtro de aprobación compara booleano vs booleano
  (`typeValidation: strict`).
- **Prompt dinámico**: el prompt se arma con variables del sistema (contexto RAG, sector, datos
  del cliente), no con texto fijo.
- **Sin credenciales en el JSON**: se usan variables de entorno (`AIRTABLE_BASE_ID`,
  `EMAIL_REVISOR`) y credenciales de n8n referenciadas por nombre.

## Cómo importar

1. En n8n: **Workflows → Import from File →** `workflow-propuestas-hitl.json`.
2. Configurar credenciales:
   - **Airtable Personal Access Token** (nodos Airtable + triggers).
   - **Header Auth** para la API de Gemini: header `x-goog-api-key` = tu API key de Google AI Studio.
   - **Gmail OAuth2**.
3. Definir variables de entorno en n8n:
   - `AIRTABLE_BASE_ID` = id de tu base.
   - `EMAIL_REVISOR` = tu email de revisión.
4. Crear la base de Airtable según `airtable-schema.md`.
5. Activar el workflow.

## Contenido del repositorio

- `workflow-propuestas-hitl.json` — lógica del flujo (importable en n8n).
- `airtable-schema.md` — estructura de la base de datos (tablas, campos, relaciones).
- `docs/diagrama-arquitectura.pdf` — diagrama y documentación de arquitectura.
- `docs/screenshots/` — capturas de evidencia (trigger, ejecución, HITL, camino infeliz, envío).
- Link a la base Airtable en modo lectura: https://airtable.com/app7ClffAV18gZulm/shrAnAVLzsYZYhDEr
- Video demo (3 min): _(pegar el link del video acá)_

## Test de estrés (mínimo 5 ejecuciones)

| # | Escenario | Resultado esperado |
|---|-----------|--------------------|
| 1 | Datos completos, se aprueba | Propuesta enviada, Estado=Enviado |
| 2 | Datos completos, NO se aprueba | Se queda en Revisión pendiente, no envía |
| 3 | **Camino infeliz**: fila sin "Datos del Cliente" | El IF la descarta, no llama a la IA |
| 4 | **Camino infeliz**: API de IA caída / key inválida | Estado=Error + Log Error, flujo estable |
| 5 | Fila ya en Enviado se re-modifica | No re-dispara (anti bucle infinito) |
