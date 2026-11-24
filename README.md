# CoderHouse · AI Automation — Proyecto integrador

Agente autónomo en **n8n** para una **agencia de viajes**: conversa con personas interesadas en viajar, releva sus datos, califica el lead (CALIENTE / TIBIO / FRÍO) y lo registra para el equipo comercial.

El proyecto crece módulo a módulo sobre el mismo workflow.

**Repositorio público:** https://github.com/DiegoCostanzo/coderhouse-ai-automation

| Módulo | Contenido | Carpeta |
|---|---|---|
| M1 | Agente base y motor de razonamiento | [`m1/`](m1/) |
| M2 | Orquestación multi-agente (Manager-Worker con sub-workflows) | [`m2/`](m2/) |
| M3 | Memoria persistente en Airtable por Session_ID y resumen agéntico | [`m3/`](m3/) |

---

## M1 · Agente base y motor de razonamiento

Archivo de entrega: [`m1/checkpoint1_diego_costanzo.json`](m1/checkpoint1_diego_costanzo.json)

### Arquitectura

```
[When chat message received] ──► [AI Agent] ──► [Log_Gmail]
                                     │
                                     ├─ Chat Model: Google Gemini Chat Model
                                     ├─ Memory:     Simple Memory (ventana de 10 mensajes)
                                     ├─ Tool:       Buscar_Lead    (Google Sheets · Get Rows por Email)
                                     └─ Tool:       Registrar_Lead (Google Sheets · Append Row)
```

| Componente | Configuración |
|---|---|
| Trigger | Chat Trigger, Response Mode = Streaming |
| AI Agent | Tools Agent · **Max Iterations = 7** · Return Intermediate Steps = ON |
| System Message | Modular: Rol → Ámbito (puede / no puede) → Objetivo → Reglas de herramientas → Escalamiento → Estilo |
| Tools | Conectadas lateralmente al puerto Tool, con *Tool Description* manual que indica cuándo usarlas |
| Parámetros de tools | `$fromAI()` para datos que el agente interpreta; `$now` para la fecha de registro |
| Observabilidad | Gmail (texto plano) con Session ID, mensaje, respuesta y herramientas usadas con sus entradas/resultados |

### Comportamiento esperado

- Pide los datos faltantes (nombre, contacto, destino como mínimo) antes de registrar.
- Antes de registrar, busca por email para **no duplicar** leads.
- **No** da precios, reservas, descuentos ni gestiona pagos/reclamos: escala a revisión humana.
- Rechaza temas ajenos a viajes sin usar herramientas.

### Cómo probarlo

1. Importar `m1/checkpoint1_diego_costanzo.json` en n8n (*Import from File*).
2. Configurar credenciales: Google Gemini (API key), Google Sheets OAuth2 y Gmail OAuth2.
3. Crear una planilla `Leads Agencia Viajes`, hoja `Leads`, con los encabezados:
   `Fecha_Registro | Nombre | Email | Telefono | Destino | Fecha_Viaje | Pasajeros | Presupuesto | Tipo_Viaje | Calificacion | Notas`
   y seleccionarla en los nodos `Buscar_Lead` y `Registrar_Lead`.
4. En `Log_Gmail`, reemplazar `tu_email@ejemplo.com` por el destinatario del log.
5. **Open chat** y probar:

| Caso | Mensaje | Resultado esperado |
|---|---|---|
| Lead nuevo | `Hola, soy Martín Pérez, martin.perez@mail.com. Queremos ir a Cancún en febrero, somos 2, presupuesto 1500 USD por persona, luna de miel.` | `Buscar_Lead` → `Registrar_Lead`, fila nueva calificada |
| Duplicado | Repetir el mensaje anterior | Solo `Buscar_Lead`; avisa que ya está registrado |
| Datos incompletos | `Hola, quiero viajar a Brasil` | Pide datos, no usa herramientas |
| Escalamiento | `¿Cuánto me sale Cancún para 2 en marzo?` | Frase de transferencia a revisión humana |
| Fuera de alcance | `¿Me ayudás con mi tarea de matemática?` | Declina, no usa herramientas |

> Nota: las credenciales no se incluyen en el export (solo su nombre/ID); hay que crearlas en la instancia que importe el workflow.
