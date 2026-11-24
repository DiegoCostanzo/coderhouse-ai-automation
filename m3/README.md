# M3 · Memoria persistente y resumen agéntico (Airtable + Session_ID)

Evolución del Manager de M2: se le suma la capa de memoria de largo plazo. Los Workers de M2 no cambian (el Manager los sigue llamando por ID).

| Archivo | Contenido |
|---|---|
| `manager_modulo3_costanzo_diego.json` | Workflow `M3 - Manager Agente Leads Viajes (Memoria)` |
| `manager_modulo3_costanzo_diego_prompt_resumen.txt` | Prompt del modelo mini (summarization) |

## Arquitectura

```
LECTURA (antes del agente)
[Chat Trigger] → [Buscar_Memoria (Airtable Search · {Session_ID} = sessionId · límite 1)]
              → [Usuario_Recurrente (IF: ¿vino un registro?)]
                   ├─ true  → [Cargar_Contexto (Set: user_name, status, last_summary, datos_clave)] ──────────┐
                   └─ false → [Registrar_Usuario_Nuevo (Airtable Create)] → [Contexto_Usuario_Nuevo (Set)] ──┤
                                                                                                             ▼
INYECCIÓN                                         [Router_Triage (Tools Agent) · System Prompt con
                                                   [INICIO DE CONTEXTO COMPARTIDO] … [FIN DEL CONTEXTO COMPARTIDO]]
                                                                     ▼
ORQUESTACIÓN M2 (sin cambios)        Switch_Intencion → Workers → Consolidar_Respuesta → Log_Trazabilidad (Gmail)
                                                                     ▼
ESCRITURA + SUMMARIZATION  [Preparar_Memoria] → [Leer_Historial (Chat Memory Manager)] → [Unificar_Historial]
                           → [Supera_5_Mensajes (IF)]
                                ├─ true  → [Resumir_Conversacion (Gemini Flash-Lite + parser JSON)] → [Memoria_Con_Resumen] ─┐
                                └─ false (o error del LLM) → [Memoria_Sin_Resumen (conserva el resumen previo)] ─────────────┤
                                                                                                                             ▼
                                                        [Guardar_Memoria (Airtable Create or Update · match Session_ID)] → [Salida_Chat]
```

## Base de memoria en Airtable

Base `Memoria del Agente — Leads Viajes`, tabla `Memoria`. Una fila por Session_ID.

| Campo | Tipo Airtable | Quién lo escribe | Variable inyectada |
|---|---|---|---|
| `Session_ID` (campo primario) | Single line text | Create (usuario nuevo) / Upsert (clave de matcheo) | — (filtro de la búsqueda) |
| `Nombre del usuario` | Single line text | Upsert: nombre extraído por el Router, o el guardado | `user_name` |
| `Fecha de actualización` | Date (con hora, ISO) | Create / Upsert: `$now` | — |
| `Resumen consolidado` | Long text | Upsert: JSON `{asunto_principal, puntos_clave, accion_requerida}` | `last_summary` |
| `Estado del caso` | Single select | Create: `NUEVO` · Upsert: según la ruta | `status` |
| `Datos clave` | Long text | Upsert: JSON con los datos del lead (merge previo + nuevo, sin vacíos) | `datos_clave` |

Opciones de `Estado del caso`: `NUEVO`, `EN_CONVERSACION`, `DATOS_INCOMPLETOS`, `LEAD_REGISTRADO`, `ESCALADO_HUMANO`, `ERROR_TECNICO` (con *typecast* activado, si falta una opción Airtable la crea).

Reglas de la capa:
- **Aislamiento por Session_ID**: la lectura filtra con `filterByFormula` y la escritura matchea por `Session_ID`, así nunca se mezclan dos clientes.
- **Idempotencia**: `Guardar_Memoria` es *Create or Update*: siempre sobreescribe la misma fila del Session_ID, nunca duplica.
- **Solo lo importante**: se guarda el resumen estructurado y los datos clave. Nunca la conversación completa, logs ni HTML.
- **Sin variables vacías**: el usuario nuevo recibe textos por defecto ("Sin registrar", "primera conversación…").
- **Tolerancia a fallos**: los nodos de Airtable tienen *Retry* + *Continue on error*; si el LLM de resumen falla, se conserva el resumen anterior y el chat responde igual.

## Summarization

`Supera_5_Mensajes` cuenta los mensajes del historial de la sesión (cliente + agente) que devuelve `Leer_Historial`. Si son más de 5, `Resumir_Conversacion` manda el resumen previo, los datos clave y el historial reciente a Gemini 3.5 Flash-Lite (temperatura 0), con un Structured Output Parser que obliga el esquema:

```json
{ "asunto_principal": "string", "puntos_clave": ["string"], "accion_requerida": "string" }
```

## Cómo probarlo

1. Airtable: crear la base y la tabla con los 6 campos de arriba. Crear un *Personal Access Token* (scopes `data.records:read`, `data.records:write`, `schema.bases:read`, acceso a esa base).
2. n8n: *Import from File* → `manager_modulo3_costanzo_diego.json`.
3. En `Buscar_Memoria`, `Registrar_Usuario_Nuevo` y `Guardar_Memoria`: crear/elegir la credencial de Airtable y elegir Base y Table **From list**.
4. En `Log_Trazabilidad`, poner el mail real. Verificar que los dos `Ejecutar_Worker_*` apunten a los Workers de M2.
5. Open chat:
   - Mensaje 1 (`Hola, soy Martín Pérez, quiero ir a Cancún`) → fila nueva en Airtable, Estado `NUEVO` → `DATOS_INCOMPLETOS`.
   - Mensajes 2 y 3 (email, fecha, pasajeros, presupuesto) → al pasar los 5 mensajes se completa `Resumen consolidado` con el JSON.
   - Siguiente mensaje en el mismo chat → en la ejecución, `Usuario_Recurrente` sale por *true* y el System Prompt del Router muestra el bloque de contexto con los datos de Airtable.
