# Pasos finales (lo que queda para cerrar la entrega al 100%)

Todo el diseño, la base de datos, el flujo y la documentación están listos. Quedan
**solo acciones que requieren tus credenciales o tu cuenta** (por seguridad no las puede
hacer un asistente): conectar secretos, autenticar OAuth, hacer público un enlace, subir a
GitHub y grabar el video.

## Estado actual

| Ítem | Estado |
|------|--------|
| Base Airtable (2 tablas relacionadas + estados + RAG + registro de prueba) | ✅ Hecho |
| Campo `Ultima modificacion` (Last modified time) para los triggers | ✅ Hecho |
| Workflow n8n importado (`workflow-propuestas-hitl.json`) | ✅ Hecho |
| Diagrama de arquitectura en PDF (`docs/diagrama-arquitectura.pdf`) | ✅ Hecho |
| README + esquema de la base | ✅ Hecho |
| Repositorio git local (commiteado) | ✅ Hecho |
| Credenciales en n8n (Anthropic + Gmail) | ⛔ Las cargás vos (secretos) |
| Config final de nodos + test de estrés | ⛔ Tras cargar credenciales |
| Enlace público (solo lectura) a la base | ⛔ Lo generás vos (permisos) |
| Repo en GitHub (push) | ⛔ Lo publicás vos |
| Video demo 3 min | ⛔ Lo grabás vos |

---

## 1. Cargar las 2 credenciales que faltan en n8n (secretos)

En n8n → **Credentials → Add credential**:

- **Header Auth** (para Claude): Name `x-api-key`, Value = tu API key de Anthropic
  (console.anthropic.com → API Keys). Luego, en el nodo *IA - Generar Propuesta (Claude)*
  seleccioná esta credencial.
- **Gmail OAuth2**: conectá tu cuenta Google. Luego seleccioná esta credencial en los nodos
  *HITL - Notificar Revisor* y *Enviar Propuesta al Cliente*.

> La credencial de **Airtable** ya la conectaste antes; solo seleccionala en cada nodo Airtable.

## 2. Terminar de mapear los nodos (sin secretos)

En cada nodo con ⚠️:
- **Base**: `Ecosistema IA - Propuestas HITL` (id `app7ClffAV18gZulm`)
- **Table**: `Solicitudes` (id `tbluu65vfvwesKYiB`) en todos menos el RAG;
  `Base_Conocimiento` (id `tblcQ3njQhPEEGxkH`) en *RAG - Buscar Directrices Privadas*.
- **Triggers** (los 2): *Trigger Field* = `Ultima modificacion`.
- En *HITL - Notificar Revisor* poné tu email en **Send To**.

## 3. Test de estrés (5 corridas, incluye camino infeliz)

En la tabla `Solicitudes`, con la fila *Innova Software SA*:
1. Estado → `Generando` (con datos completos) → debe generar borrador + mail. Luego marcá
   `Aprobado` ✓ → debe enviar y pasar a `Enviado`.
2. Otra fila con datos, sin aprobar → queda en `Revision pendiente`, no envía.
3. Fila SIN "Datos del Cliente" → el IF la descarta, no llama a la IA.
4. Poné una API key inválida y dispará → debe caer en `Estado=Error` + `Log Error`.
5. Modificá una fila ya en `Enviado` → no re-dispara.

Sacá **captura de pantalla** de cada corrida (n8n → pestaña *Ejecuciones*) → guardalas en
`docs/screenshots/`.

## 4. Enlace público (solo lectura) a la base

En Airtable → botón **Compartir** (arriba a la derecha) → **Compartir públicamente** →
acceso **solo lectura** → copiá el link. Pegalo en el README (sección "Enlaces") y en el
formulario de entrega.

## 5. Subir a GitHub

El repo ya está inicializado y commiteado localmente en esta carpeta. Para publicarlo:

```bash
# en https://github.com creá un repo vacío (ej. "ecosistema-ia-propuestas-hitl")
git remote add origin https://github.com/TU_USUARIO/ecosistema-ia-propuestas-hitl.git
git push -u origin main
```

Verificá que en GitHub estén: el PDF, el JSON, el schema, el README y los screenshots.

## 6. Video demo (3 min)

Grabá: el trigger (cambiar Estado a `Generando`), el procesamiento en n8n, el mail de
revisión (HITL), la aprobación, y el envío final. **Ocultá tus API keys** en la grabación.
Subí el link (YouTube no listado / Loom) al README y al formulario.

---

## Check de seguridad (ya cubierto en el diseño)
- Sin bucles infinitos: los triggers filtran por estado; `Enviado` no re-dispara.
- Comparación de tipos correcta: el filtro de aprobación compara booleano vs booleano.
- Prompt dinámico: se arma con variables del sistema (RAG + datos del cliente).
- Sin datos hardcodeados: base/credenciales por referencia, email del revisor configurable.
