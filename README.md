# 🤖 HelpDeskBot — Documentación Técnica

Bot de gestión de solicitudes de soporte construido con **n8n**, **Telegram** y **Google Sheets** como base de datos.

---

## 📌 Descripción General

HelpDeskBot es un chatbot de Telegram que permite a los usuarios crear, consultar y gestionar tickets de soporte directamente desde la aplicación de mensajería. Toda la información se almacena en una hoja de cálculo de Google Sheets que actúa como base de datos liviana.

El bot mantiene un registro de navegación (LOGS) para rastrear en qué pantalla se encuentra cada usuario y qué opciones ha seleccionado, lo que permite una experiencia de flujo paso a paso (wizard).

---

## 🗂️ Estructura de la Base de Datos

La base de datos es una hoja de cálculo de Google Sheets (`HelpDeskBot_DB`) con tres pestañas:

### 📋 SOLICITUDES
| Campo | Descripción |
|---|---|
| `id_ticket` | Identificador único del ticket (formato `yyyyMMddHHmmss`) |
| `tipo` | Tipo de solicitud: Soporte técnico / Solicitud administrativa / Consulta general |
| `prioridad` | Alta / Media / Baja |
| `descripcion` | Texto libre con el detalle del problema |
| `estado` | Abierto / Cancelado |
| `creado_por` | Username de Telegram del usuario |
| `fecha_creacion` | Fecha y hora de confirmación (zona horaria Bogotá) |

### 👥 USUARIOS
| Campo | Descripción |
|---|---|
| `telegram_user` | Username de Telegram |
| `nombre` | Primer nombre del usuario |
| `rol` | Rol asignado (por defecto: `Empleado`) |
| `activo` | `si` / `no` — indica si tiene sesión activa |

### 📝 LOGS
| Campo | Descripción |
|---|---|
| `timestamp` | Fecha y hora del evento (zona horaria Bogotá) |
| `telegram_user` | Username del usuario |
| `pantalla` | Pantalla actual en la que se encontraba el usuario |
| `opcion` | Opción seleccionada por el usuario |
| `resultado` | Resultado de la acción ejecutada |

---

## 🧭 Menú Principal

Al iniciar una conversación o volver al menú, el usuario ve las siguientes opciones:

```
0. Ayuda
1. Crear solicitud
2. Consultar estado de solicitud
3. Mis solicitudes
4. Reportes
5. Configuración
6. Terminar
```

---

## 🔄 Flujo General del Bot

### 1. Recepción del mensaje (Telegram Trigger)
Cada mensaje enviado al bot dispara el workflow. El primer paso siempre es consultar la tabla **USUARIOS** para verificar si el usuario ya está registrado.

### 2. Registro de usuario
- **Usuario nuevo:** se registra automáticamente con rol `Empleado` y `activo = si`, y se le da la bienvenida.
- **Usuario existente con sesión cerrada (`activo = no`):** se reactiva su sesión y se le saluda de vuelta.
- **Usuario existente con sesión activa:** se consulta el log más reciente para determinar en qué pantalla estaba y continuar desde ahí.

### 3. Manejo de pantallas (wizard stateful)
El bot usa los LOGS para recordar en qué paso del proceso se encuentra el usuario. El nodo **"Code in JavaScript"** filtra el último registro con `opcion = "en espera"` del usuario para determinar la pantalla activa.

Las pantallas del wizard de creación de solicitudes son:

| Pantalla (en LOGS) | Descripción |
|---|---|
| `menu_principal` | Menú de inicio |
| `ayuda` | Vista de ayuda |
| `tipo_solicitud` | Selección de tipo (Soporte / Admin / Consulta) |
| `solicitud_prioridad` | Selección de prioridad (Alta / Media / Baja) |
| `solicitud_descripcion` | Ingreso de descripción libre |
| `solicitud_confirmar` | Confirmación (Sí / No) |
| `consultar_ticket` | Búsqueda de un ticket por ID |

---

## 🛠️ Opciones del Menú

### 0 — Ayuda
Muestra información sobre cómo usar el bot, tipos de solicitudes disponibles y recomendaciones de uso. Luego regresa al menú principal.

### 1 — Crear solicitud (Wizard de 4 pasos)

```
Paso 1: Tipo de solicitud
  1. Soporte técnico
  2. Solicitud administrativa
  3. Consulta general
  0. Volver al menú principal

Paso 2: Prioridad
  1. Alta
  2. Media
  3. Baja

Paso 3: Descripción
  (texto libre)

Paso 4: Confirmación
  1. Sí → Ticket creado ✅
  2. No → Ticket cancelado ❌
```

Al confirmar, el ticket queda registrado en la hoja **SOLICITUDES** con estado `Abierto` y se notifica al usuario con el número de ticket asignado.

### 2 — Consultar estado de solicitud
El usuario ingresa el número de ticket. El bot busca en **SOLICITUDES** filtrando por `creado_por` + `id_ticket`. Devuelve el estado actual o un mensaje de error si no se encuentra.

### 3 — Mis solicitudes
Lista todas las solicitudes del usuario con los campos: ticket, tipo, prioridad, descripción y fecha. Si no tiene solicitudes, informa al usuario.

### 4 — Reportes
Genera un resumen estadístico de las solicitudes del usuario agrupadas por tipo, prioridad y estado.

### 5 — Configuración
Muestra el perfil del usuario: username de Telegram, nombre y rol asignado.

### 6 — Terminar
Cierra la sesión del usuario (`activo = no` en USUARIOS), registra el evento en LOGS y muestra un mensaje de despedida.

---

## ⚙️ Tecnologías Utilizadas

| Tecnología | Uso |
|---|---|
| **n8n** | Motor de automatización y orquestación del flujo |
| **Telegram Bot API** | Canal de comunicación con los usuarios |
| **Google Sheets** | Base de datos (SOLICITUDES, USUARIOS, LOGS) |
| **JavaScript (Code nodes)** | Lógica de negocio y procesamiento de datos |

---

## 🔐 Credenciales Requeridas

Para desplegar este workflow en n8n se necesitan configurar:

1. **Telegram API** — Token del bot creado con BotFather.
2. **Google Sheets OAuth2** — Cuenta de Google con acceso a la hoja `HelpDeskBot_DB`.

---

## 📦 Nodos Principales del Workflow

| Nodo | Tipo | Función |
|---|---|---|
| `Telegram Trigger` | Trigger | Recibe mensajes entrantes |
| `Get row(s) in sheet` | Google Sheets | Busca al usuario en USUARIOS |
| `If` | Condicional | ¿El usuario ya existe? |
| `Append row in sheet` | Google Sheets | Registra nuevo usuario |
| `If2` | Condicional | ¿El usuario está inactivo? |
| `Update row in sheet` | Google Sheets | Reactiva usuario inactivo |
| `Get row(s) in sheet1` | Google Sheets | Obtiene logs del usuario |
| `Code in JavaScript` | Código | Extrae el último log pendiente |
| `If1` | Condicional | ¿Está en menú principal? |
| `If Tipo Valido1` | Condicional | Valida opción del menú |
| `Switch` | Router | Dirige al módulo seleccionado |
| `Switch Wizard Pantalla` | Router | Dirige según pantalla activa del wizard |
| `Code in JavaScript8` | Código | Genera lista de solicitudes del usuario |
| `Code in JavaScript9` | Código | Genera reporte estadístico |
| `Code in JavaScript10` | Código | Genera vista de perfil de usuario |

---

## 🌐 Zona Horaria

Todos los timestamps se registran en la zona horaria **America/Bogotá** (`UTC-5`) usando la expresión:

```javascript
DateTime.fromSeconds($node["Telegram Trigger"].json.message.date)
  .setZone('America/Bogota')
  .toFormat('yyyy-MM-dd HH:mm:ss')
```

---

## 📎 Notas Adicionales

- El bot valida las entradas del usuario en cada pantalla. Si se ingresa una opción inválida, muestra un mensaje de advertencia y repite la pregunta.
- El estado de la sesión se gestiona mediante el campo `activo` en la tabla USUARIOS y los registros de LOGS. No se usa ninguna variable de sesión en n8n.
- El ID del ticket se genera al momento de crear el registro inicial en SOLICITUDES usando el timestamp en formato `yyyyMMddHHmmss`.
- Las opciones **Reportes** y **Configuración** son de solo lectura; no modifican solicitudes.
