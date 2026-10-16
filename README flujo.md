# Ecosistema Leads IA: clasificación y contacto de leads con IA

Entrega final: **Construcción de un Ecosistema de Automatización IA**.
Perfil: Arquitecto/a de Flujos IA.

## 1. Caso de uso

Una empresa recibe consultas de potenciales clientes (leads) por formulario web y por referidos. Cada consulta llega en lenguaje natural, por ejemplo: *"Somos una cadena de 50 sucursales y necesitamos una propuesta urgente"*.

El sistema:

1. Registra el lead en la base de datos.
2. Verifica que los datos estén completos.
3. Le pide a **Gemini** que lo clasifique (**VIP**, **Estándar** o **Descartar**), le asigne un **score** del 1 al 10 y escriba un **resumen**.
4. Si el lead es VIP, **pide aprobación a una persona** por Gmail antes de contactarlo.
5. Si la persona aprueba, **contacta al lead por Gmail** y guarda el **Thread ID** de la conversación.

## 2. Stack

| Capa | Herramienta |
|------|-------------|
| Orquestador | n8n (cloud) |
| Base de datos ("cerebro") | Airtable |
| Motor de razonamiento (LLM) | Google Gemini (`gemini-3.6-flash`) |
| Aprobación y salida | Gmail |
| Disparador | Webhook (POST) |

## 3. Base de datos (Airtable)

Base: **Ecosistema Leads IA**, con 4 tablas relacionadas (ver página 2 del PDF).

| Tabla | Para qué sirve | Campos principales |
|-------|----------------|--------------------|
| **Empresas** | Datos de la empresa del lead | Nombre, Sector, Tamaño, Sitio web, Leads (vínculo) |
| **Leads** | Tabla central | Nombre, Mensaje original, Fuente, Empresa (vínculo), Sector y Tamaño (lookups), **Estado**, Categoría IA, Email, Score IA, Resumen IA, Gmail Thread ID, Interacciones (vínculo), Errores (vínculo) |
| **Interacciones** | Borradores y envíos al lead | Asunto borrador, Lead (vínculo), Email (lookup), Cuerpo borrador, Estado, Aprobado por, Fecha de envío |
| **Errores** | Registro de fallas | Mensaje de error, Lead (vínculo), Nodo que falló, Tipo, Fecha, Resuelto |

**Relaciones:** Empresas 1 → N Leads; Leads 1 → N Interacciones; Leads 1 → N Errores.

**Campo Estado (Leads):** `Pendiente`, `Procesado por IA`, `Aprobado por Humano`, `Rechazado`, `Contactado`, `Datos incompletos`, `Error`.

**Nota:** la tabla **Interacciones** y el campo **Slack Thread TS** quedan reservados para ampliaciones futuras (por ejemplo, guardar el borrador enviado o notificar por Slack). El flujo actual no escribe en ellos.

## 4. Flujo en n8n

Diagrama completo en [`diagrama_arquitectura.pdf`](./diagrama_arquitectura.pdf) y flujo exportado en [`flujo_n8n.json`](./flujo_n8n.json).

1. **Webhook nuevo lead** (`POST /nuevo-lead`): trigger inteligente; el flujo solo corre cuando llega un lead.
2. **Registrar lead nuevo**: crea el registro en Airtable con Estado = `Pendiente`.
3. **Buscar leads pendientes**: filtra por Estado = `Pendiente`.
4. **¿Datos completos?**: si falta un dato (por ejemplo, el email) va a **Marcar Datos incompletos** y no se llama a la IA.
5. **Calificar lead con Gemini**: responde solo con un JSON (categoría `VIP`, `Estándar` o `Descartar`, score del 1 al 10 y resumen de 2 frases). El prompt es dinámico: usa el nombre, el sector, el tamaño de la empresa y el mensaje del lead. Tiene el límite *Max Output Tokens* configurado y reintenta automáticamente si falla.
6. **Separar respuesta de IA**: convierte el JSON de Gemini en los campos `categoria`, `score` y `resumen`.
7. **Guardar resultado en Airtable**: Estado = `Procesado por IA`.
8. **¿Es VIP?**: solo los VIP siguen al paso de aprobación. Los leads `Estándar` y `Descartar` quedan en `Procesado por IA`, sin contacto.
9. **Pedir aprobación por Gmail** (*Send message and wait for response*): el flujo se frena hasta que una persona apruebe o rechace.
10. **¿Aprobó?**: compara un valor booleano.
    - Sí → **Marcar Aprobado por Humano** → **Contactar lead por Gmail** → **Guardar Thread ID** (Estado = `Contactado`).
    - No → **Marcar Rechazado**.

### Detalle de nodos

El flujo tiene **18 nodos**. Hay dos formas de entrada: el **Webhook** (producción) y el **Inicio manual** (solo para pruebas).

| # | Nodo | Tipo | Qué hace y configuración clave |
|---|------|------|--------------------------------|
| 1 | **Webhook nuevo lead** | Webhook | `POST /nuevo-lead`. Recibe `nombre`, `email` y `mensaje` dentro de `body`. |
| 2 | **Inicio manual (prueba)** | Manual Trigger | Dispara el flujo a mano para pruebas. Va directo a *Buscar leads pendientes*. |
| 3 | **Registrar lead nuevo** | Airtable (create) | Crea el lead en la tabla Leads con `Nombre`, `Email` y `Mensaje original` tomados de `body`, y Estado = `Pendiente`. |
| 4 | **Buscar leads pendientes** | Airtable (search) | Fórmula `{Estado} = 'Pendiente'`. Solo se procesan los leads pendientes (anti-bucles). |
| 5 | **¿Datos completos?** | If (booleano) | Verdadero si `Email` y `Mensaje original` no están vacíos (se recortan los espacios con `trim`). *true* → Gemini. *false* → Marcar Datos incompletos. |
| 6 | **Marcar Datos incompletos** | Airtable (update) | Estado = `Datos incompletos`. No se llama a la IA. |
| 7 | **Calificar lead con Gemini** | Google Gemini | Modelo `gemini-3.6-flash`. Prompt dinámico con nombre, sector y tamaño de la empresa y mensaje del lead. `maxOutputTokens` = 5000. **Reintenta** si falla (espera 2 s entre intentos) y, si sigue fallando, sale por la rama *Error* (`continueErrorOutput`). |
| 8 | **Separar respuesta de IA** | Set | Convierte el JSON de Gemini (`content.parts[0].text`) en tres campos: `categoria` (texto), `score` (número) y `resumen` (texto). |
| 9 | **Guardar resultado en Airtable** | Airtable (update) | Guarda Categoría IA, Score IA y Resumen IA, y cambia Estado a `Procesado por IA`. |
| 10 | **¿Es VIP?** | If (texto) | Compara `categoria` con `VIP` (texto contra texto). *true* → pedir aprobación. *false* → el flujo termina. |
| 11 | **Pedir aprobación por Gmail** | Gmail (*Send and wait for response*) | Envía el mail al aprobador con nombre, email, mensaje, score y resumen. Tipo de aprobación: *Approve / Disapprove*. **El flujo se frena hasta recibir la respuesta.** |
| 12 | **¿Aprobó?** | If (booleano) | Evalúa `{{ $json.data.approved }}` como booleano. *true* → aprobado. *false* → rechazado. |
| 13 | **Marcar Aprobado por Humano** | Airtable (update) | Estado = `Aprobado por Humano`. |
| 14 | **Marcar Rechazado** | Airtable (update) | Estado = `Rechazado`. No se contacta al lead. |
| 15 | **Contactar lead por Gmail** | Gmail (send) | Envía el mail al lead (`Email` del registro) con asunto "Hola {Nombre}, recibimos tu consulta" y propone coordinar una demo. |
| 16 | **Guardar Thread ID** | Airtable (update) | Guarda `threadId` del mail en el campo **Gmail Thread ID** y cambia Estado a `Contactado`. |
| 17 | **Registrar error en log** | Airtable (create) | Crea un registro en la tabla **Errores**: mensaje de error, nodo que falló, lead afectado (vínculo), fecha (`$now`), Tipo = `API IA` y Resuelto = falso. |
| 18 | **Marcar lead con Error** | Airtable (update) | Estado = `Error`. |

### Gestión de errores (resiliencia)

Si falla la API de Gemini, la salida *Error* del nodo registra el problema en la tabla **Errores** (mensaje, nodo que falló, tipo, fecha, lead) y marca el lead con Estado = `Error`. El sistema no se cae y la falla queda documentada.

### Human-in-the-loop

Antes de cualquier contacto con el cliente, el flujo envía un mail de aprobación con nombre, email, mensaje, score y resumen del lead, y espera la respuesta. Así se evita el "efecto metralleta".

### Cómo importar y probar el flujo

1. En n8n, creá un workflow nuevo y elegí **Import from file** con [`flujo_n8n.json`](./flujo_n8n.json).
2. Conectá tus propias credenciales de **Airtable**, **Gmail** y **Google Gemini**. El archivo no incluye claves.
3. En el nodo **Pedir aprobación por Gmail**, reemplazá `tu-email@ejemplo.com` por el mail de quien aprueba.
4. En **Registrar lead nuevo**, **Buscar leads pendientes** y el resto de los nodos de Airtable, elegí tu base y tus tablas (`Leads` y `Errores`).
5. Abrí el nodo **Webhook nuevo lead**, hacé clic en **Listen for test event** y enviá un `POST` a la *Test URL* con este cuerpo:

```json
{
  "nombre": "Carlos Pérez",
  "email": "carlos@ejemplo.com",
  "mensaje": "Somos una cadena de 50 sucursales y necesitamos una propuesta urgente para esta semana."
}
```

Para probar un lead existente, ponelo en estado `Pendiente` en Airtable y ejecutá desde **Inicio manual (prueba)**.

## 5. Pruebas realizadas

| # | Caso | Lead de prueba | Resultado esperado | Evidencia |
|---|------|----------------|--------------------|-----------|
| 1 | VIP aprobado | Carlos Pérez, Martín Rodríguez | Contactado + Thread ID guardado | `evidencias/` |
| 2 | VIP rechazado | Carolina Méndez | Estado `Rechazado` | `evidencias/` |
| 3 | No VIP | Diego Sosa | Queda en `Procesado por IA`, sin mail | `evidencias/` |
| 4 | Datos incompletos (camino infeliz) | Ana Torres, Patricia Pintos | Estado `Datos incompletos`, sin llamar a la IA | `evidencias/` |
| 5 | Falla de la IA (camino infeliz) | Gabriel Pérez | Registro en tabla Errores + Estado `Error` | `evidencias/` |

## 6. Check de seguridad

- [x] **Anti-bucles:** el flujo solo toma leads en `Pendiente` y cada rama cambia el Estado, así que un lead no se reprocesa.
- [x] **Tipos de datos correctos en filtros:** `¿Aprobó?` compara un booleano; `¿Es VIP?` compara texto con texto.
- [x] **Prompt dinámico:** usa variables del lead (nombre, sector, tamaño y mensaje), no texto fijo.
- [x] **Costos controlados:** *Max Output Tokens* limitado, y el filtro `¿Datos completos?` evita llamar a la IA con leads incompletos.
- [x] **Credenciales protegidas:** el JSON exportado no incluye API keys (solo el nombre de las credenciales) y el video las oculta.

## 7. Limitaciones conocidas y mejoras futuras

- El Webhook no tiene autenticación (suficiente para la demo). En producción convendría agregar *Header Auth*.
- La categoría `Descartar` todavía no tiene una rama propia: hoy termina igual que `Estándar`.
- La tabla `Interacciones` no se usa todavía; se podría guardar ahí el mail enviado al lead.
- Un lead en estado `Error` no se reintenta solo: hay que revisarlo en la tabla `Errores` y volver a ponerlo en `Pendiente`.

## 8. Entregables

- [`diagrama_arquitectura.pdf`](./diagrama_arquitectura.pdf): diagrama de arquitectura y modelo de datos.
- [`flujo_n8n.json`](./flujo_n8n.json): lógica del flujo.
- **Base de datos (solo lectura):** `https://airtable.com/appEaA6OqswO6At2e/shrBtGe60FPoXXPL5`
- **Video demo (3 min):** `https://drive.google.com/file/d/1CC-BTXfiDuNKarMjl8jCCHx5e8o_DXlX/view?usp=sharing`
- **Capturas de evidencia:** carpeta [`evidencias/`](./evidencias).

## 9. Estructura del repositorio

```
.
├── README.md
├── diagrama_arquitectura.pdf
├── flujo_n8n.json
└── evidencias/
    ├── 01_vip_aprobado.png
    ├── 02_vip_rechazado.png
    ├── 03_no_vip.png
    ├── 04_datos_incompletos.png
    ├── 05_error_ia.png
    └── airtable_tablas.png
```
