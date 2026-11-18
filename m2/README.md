# M2 · Orquestación multi-agente (Manager-Worker con sub-workflows)

Evolución del agente de M1: la lógica que antes vivía en un solo AI Agent con tools se separa en un **Manager** (router de triaje) y dos **Workers** especialistas en lienzos independientes.

| Archivo | Workflow | Rol |
|---|---|---|
| `manager_modulo2_costanzo_diego.json` | M2 - Manager Agente Leads Viajes | Recibe el chat, clasifica la intención y delega |
| `worker1_modulo2_costanzo_diego.json` | M2 - Worker_Registro_Lead | Valida, califica, deduplica y registra el lead en Google Sheets |
| `worker2_modulo2_costanzo_diego.json` | M2 - Worker_Escalamiento_Humano | Redacta con IA el correo de derivación y alerta al asesor por Gmail |

## Arquitectura

```
MANAGER
[Chat Trigger] → [Router_Triage (AI Agent + Gemini + Memory + Structured Output Parser)]
                    │ éxito                                  │ error del router
                    ▼                                        ▼
             [Switch_Intencion] ──Escalar────────────► [Payload_Escalamiento] → [Ejecutar_Worker_Escalamiento] → [Resp_Escalamiento]
                    │──Registrar─► [Payload_Registro] → [Ejecutar_Worker_Registro] → [Resp_Registro]
                    │──Responder─► [Resp_Directa]                    │ error / timeout de cualquier worker
                    └──Fallback_Humano ──► [Payload_Escalamiento]    └──────────► [Fallback_Worker_Caido]
                                                         ↓ (las 4 ramas)
                              [Consolidar_Respuesta] → [Log_Trazabilidad (Gmail)] → [Salida_Chat]

WORKER 1 · Registro de lead
[Execute Workflow Trigger] → [Validar_Datos_Minimos] ─no─► [Salida_Error_Validacion]
                                   └─sí─► [Calificar_Lead] → [Buscar_Duplicado] → [Resultado_Busqueda (Switch)]
                                                                     ├─Error──► [Salida_Error_Sheets]
                                                                     ├─Existe─► [Salida_Duplicado]
                                                                     └─Nuevo──► [Registrar_Lead] ─ok─► [Salida_Exito]
                                                                                              └─error─► [Salida_Error_Sheets]

WORKER 2 · Escalamiento humano
[Execute Workflow Trigger] → [Redactar_Correo_Asesor (LLM)] ─error─► [Salida_Error_LLM]
                                   └─► [Enviar_Alerta_Asesor (Gmail)] ─ok─► [Salida_Exito]
                                                                     └─error─► [Salida_Error_Gmail]
```

## Criterio de enrutamiento

El `Router_Triage` es un AI Agent **sin herramientas** (no resuelve nada, solo clasifica) con un Structured Output Parser que fuerza esta salida:

```json
{
  "intent": "NUEVO_LEAD",
  "confidence": 0.92,
  "riesgo": "MEDIO",
  "motivo": "Quiere viajar y ya dio nombre, email y destino.",
  "respuesta_cliente": "",
  "lead": { "nombre": "...", "email": "...", "telefono": "...", "destino": "...",
            "fecha_viaje": "...", "pasajeros": "...", "presupuesto": "...", "tipo_viaje": "..." }
}
```

Taxonomía cerrada de negocio y ruta que toma cada intención en el `Switch_Intencion` (reglas deterministas, cero tokens; gana la primera que coincide):

| Intención | Riesgo | Regla del Switch | Destino |
|---|---|---|---|
| `ESCALAR_HUMANO` (precio, reserva, pago, reembolso, reclamo, molestia) | ALTO | intent = ESCALAR_HUMANO | Worker 2 (humano siempre) |
| `NUEVO_LEAD` (quiere viajar y dio nombre + contacto + destino) | MEDIO | intent = NUEVO_LEAD **y** confidence ≥ 0.7 | Worker 1 |
| `DATOS_INCOMPLETOS` / `CONVERSACION` / `FUERA_DE_ALCANCE` | BAJO | intent ∈ {…} | Respuesta directa del Router (sin Worker) |
| Cualquier otro caso (NUEVO_LEAD con confianza < 0.7, etiqueta desconocida) | — | Fallback_Humano | Worker 2 con motivo "Intención dudosa" |
| El Router falla (error del modelo o JSON inválido) | ALTO | salida de error del AI Agent | Worker 2 con motivo "Falla técnica del router" |

Así, ninguna acción que escribe datos se ejecuta con baja confianza, y lo de alto riesgo nunca lo resuelve la IA: siempre termina en una persona.

## Contrato de datos (Manager ⇄ Worker)

**Entrada (Manager → Worker).** Un nodo Set (`Payload_Registro` / `Payload_Escalamiento`) inmediatamente antes de cada `Execute Workflow` arma solo los campos que ese Worker necesita (mínimo producto viable); no viajan la memoria, los metadatos del chat ni la salida cruda del modelo. Los mismos campos están declarados en el `Execute Workflow Trigger` de cada Worker (*Define using fields below*), así que el nodo `Execute Workflow` del Manager los muestra mapeados uno por uno.

| Campo | Tipo | Worker 1 | Worker 2 | Descripción |
|---|---|:-:|:-:|---|
| `session_id` | string | ✔ | ✔ | Sesión del chat (trazabilidad) |
| `intent` | string | ✔ | ✔ | Etiqueta de la taxonomía |
| `confidence` | number | ✔ | ✔ | Confianza del Router (0–1) |
| `original_input` | string | ✔ | ✔ | Mensaje original del cliente (contexto compartido en el handoff) |
| `riesgo` | string | | ✔ | ALTO / MEDIO |
| `motivo` | string | | ✔ | Por qué se deriva a una persona |
| `nombre`, `email`, `telefono`, `destino` | string | ✔ | ✔ | Datos del lead |
| `fecha_viaje`, `pasajeros`, `presupuesto`, `tipo_viaje` | string | ✔ | | Datos para registrar y calificar |

**Salida (Worker → Manager).** Todas las ramas terminales de los Workers devuelven la misma forma, haya éxito o error:

```json
{ "status": "success", "worker": "registro_lead",
  "message": "Lead registrado en la planilla comercial.",
  "data": { "accion": "REGISTRADO", "calificacion": "CALIENTE", "nombre": "Martín Pérez", "destino": "Cancún" } }
```

```json
{ "status": "error", "worker": "registro_lead",
  "message": "No se pudo acceder a la planilla comercial.",
  "data": { "error_code": "SHEETS_ERROR", "detalle": "..." } }
```

Códigos de error: `DATOS_INSUFICIENTES`, `SHEETS_ERROR` (Worker 1); `LLM_ERROR`, `GMAIL_ERROR` (Worker 2). El Manager solo lee `status` y `data` para armar la respuesta al cliente.

## Guardrails y tolerancia a fallos

- **Wait for child to finish = ON** en ambos `Execute Workflow`: el Manager se pausa hasta recibir el JSON del Worker.
- **Timeout**: cada Worker tiene *Timeout Workflow* de 60 s (Settings del workflow); el Manager, 180 s.
- **Reintentos**: `Execute Workflow`, nodos de Google Sheets y LLM con *Retry On Fail* (2 intentos).
- **Salida de error** en cada nodo que llama a una API externa: `Registrar_Lead`, `Redactar_Correo_Asesor` y `Enviar_Alerta_Asesor` usan *On Error → Continue (using error output)*; `Buscar_Duplicado` usa *Continue (regular output)* + *Always Output Data* y el Switch `Resultado_Busqueda` separa Error / Existe / Nuevo (así un fallo de lectura nunca termina registrando el lead). Todos devuelven el JSON de contingencia en vez de cortar la ejecución.
- Si el Worker igual muere (timeout, error no controlado), la salida de error del `Execute Workflow` lleva a `Fallback_Worker_Caido`, que responde al cliente y deja registro.
- **Max Iterations = 5** en el Router.
- **Mínimo privilegio**: el Router no tiene tools. Worker 1 solo lee y agrega filas (no borra ni actualiza). Worker 2 solo envía un correo a una dirección fija.
- **Calificación determinista** en Worker 1 (sin tokens): CALIENTE si hay fecha y presupuesto, TIBIO si hay uno de los dos, FRÍO si no hay ninguno.

## Cómo probarlo

1. Importar **primero los dos Workers** (*Import from File*), guardarlos y revisar credenciales (Google Sheets, Gmail, Gemini) y la planilla `Leads Agencia Viajes` / hoja `Leads` de M1.
2. En Worker 2, nodo `Enviar_Alerta_Asesor`: reemplazar `asesor@ejemplo.com` por el mail del asesor.
3. Importar el Manager. En `Ejecutar_Worker_Registro` y `Ejecutar_Worker_Escalamiento` elegir el Worker correspondiente en **Workflow → From list**.
4. En `Log_Trazabilidad`, reemplazar `tu_email@ejemplo.com`.
5. **Open chat** en el Manager:

| Caso | Mensaje | Ruta esperada |
|---|---|---|
| Lead nuevo | `Hola, soy Martín Pérez, martin.perez@mail.com. Queremos ir a Cancún en febrero, somos 2, presupuesto 1500 USD por persona, luna de miel.` | Registrar → Worker 1 → `REGISTRADO` |
| Duplicado | Repetir el mensaje anterior en un chat nuevo | Registrar → Worker 1 → `DUPLICADO` |
| Incompleto | `Hola, quiero viajar a Brasil` | Responder (pide datos, sin Worker) |
| Escalamiento | `¿Cuánto me sale Cancún para 2 en marzo?` | Escalar → Worker 2 → mail al asesor |
| Fuera de alcance | `¿Me ayudás con mi tarea de matemática?` | Responder (declina, sin Worker) |
| Fallo de Worker | En Worker 1, `Buscar_Duplicado` → Document **By ID** = `ID_INVALIDO`, guardar y repetir el caso 1 (después volver a elegir la planilla) | Worker 1 → `Resultado_Busqueda` (Error) → `Salida_Error_Sheets` → `status: error` → el Manager responde igual |
