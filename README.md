# ☕ Chat Bot — Chatbot de Pedidos para Cafetería (n8n + Telegram + Google Sheets)

Chatbot conversacional para gestionar pedidos de una cafetería a través de **Telegram**, construido con **n8n** como motor de automatización y **Google Sheets** como base de datos.

Los clientes pueden ver el menú por categorías, armar un carrito, confirmar pedidos y consultar su estado, mientras que el administrador recibe notificaciones, puede actualizar el estado de los pedidos y recibe un reporte diario automático.

**GOOGLE SHEETS LINK**
*Link* : https://docs.google.com/spreadsheets/d/1oNcrGeUedjJGDc1VaT0hH7tAQ-2eBNVu4EIAZiSSeuw/edit?usp=sharing

Chatbot conversacional para gestionar pedidos de una cafetería a través de **Telegram**, construido con **n8n** como motor de automatización y **Google Sheets** como base de datos.

Los clientes pueden ver el menú por categorías, armar un carrito, confirmar pedidos y consultar su estado, mientras que el administrador recibe notificaciones, puede actualizar el estado de los pedidos y recibe un reporte diario automático.

---

## 📋 Índice

- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Estructura de Google Sheets](#-estructura-de-google-sheets)
- [Flujo de conversación](#-flujo-de-conversación)
- [Comandos disponibles](#-comandos-disponibles)
- [Requisitos previos](#-requisitos-previos)
- [Instalación](#-instalación)
- [Configuración](#-configuración)
- [Panel de administrador](#-panel-de-administrador)
- [Reporte diario automático](#-reporte-diario-automático)
- [Notas técnicas](#-notas-técnicas)

---

## ✨ Características

- 🤖 Bot de Telegram con manejo de sesión por usuario (máquina de estados).
- 📖 Menú dinámico leído desde Google Sheets, organizado por categorías (Bebidas, Comidas, Snacks).
- 🛒 Carrito de compras temporal por usuario.
- ✅ Validación de stock antes de confirmar un pedido.
- 📦 Registro automático de pedidos y descuento de stock en tiempo real.
- 🔔 Notificación al administrador por cada nuevo pedido.
- 🛠️ Comando de administrador para actualizar el estado de un pedido (`/estado`).
- 📊 Reporte diario automático de ventas enviado por Telegram.
- 🗂️ Persistencia total de usuarios, sesiones, menú y pedidos en Google Sheets (sin base de datos externa).

---

## 🏗️ Arquitectura

El proyecto es un único workflow de **n8n** dividido en bloques funcionales:

```
Telegram Trigger
      │
      ▼
Leer Sesión (Google Sheets)
      │
      ▼
Router (Code) ── interpreta el mensaje según el texto y la pantalla actual del usuario
      │
      ▼
Enrutar Acción (Switch) ── deriva el flujo según la acción detectada:
      │
      ├── start        → Registrar Usuario / Iniciar Sesión → Enviar Bienvenida
      ├── category      → Leer MENU (categoría) → Armar Lista Productos → Enviar Productos
      ├── pickproduct    → Leer MENU (elegir) → Elegir Producto → Enviar Pedir Cantidad
      ├── setqty        → Leer MENU (cantidad) → Validar Cantidad → Guardar Estado
      ├── cart          → Formatear Carrito → Enviar Carrito
      ├── clear         → Vaciar Carrito → Enviar Carrito Vaciado
      ├── confirm       → Validar y Generar Pedido → ¿Stock OK? → Guardar Pedido /
      │                    Descontar Stock / Notificar Pedido / Enviar Error Stock
      ├── status        → Leer PEDIDOS (usuario) → Formatear Pedidos → Enviar Pedidos
      ├── admin         → Leer PEDIDOS (admin) → Procesar Cambio Estado → ¿Pedido Existe? →
      │                    Actualizar Estado → Notificar Estado Usuario / Confirmar al Admin
      └── help          → Enviar Ayuda

Schedule Trigger (20:00) ── Reporte Diario
      │
      ▼
Leer PEDIDOS (reporte) → Armar Reporte → Enviar Reporte Admin
```

El bot funciona como una **máquina de estados**: cada usuario tiene una "pantalla actual" guardada en la hoja `sesiones`, que determina cómo se interpreta su siguiente mensaje (por ejemplo, si está eligiendo una categoría, un producto o una cantidad).

---

## 🗂️ Estructura de Google Sheets

El workflow utiliza una única hoja de cálculo (`deliverybot_ch`) con 4 pestañas:

### `usuarios`
| Columna | Descripción |
|---|---|
| `id telegram` | ID único del usuario en Telegram |
| `nombre completo` | Nombre del usuario (tomado de Telegram) |

### `sesiones`
| Columna | Descripción |
|---|---|
| `id telegram` | ID del usuario (clave de coincidencia) |
| `pantalla actual` | Estado actual de la conversación (home, cat:X, qty:X, etc.) |
| `carrito temporal` | Carrito en formato JSON (string) |
| `ultimo cambio` | Fecha/hora de la última interacción |

### `menu`
| Columna | Descripción |
|---|---|
| `id producto` | Identificador único del producto |
| `nombre` | Nombre del producto |
| `descripcion` | Descripción del producto |
| `precio` | Precio unitario |
| `categoria` | Bebidas / Comidas / Snacks |
| `stock` | Unidades disponibles |

### `pedidos`
| Columna | Descripción |
|---|---|
| `id pedido` | Identificador único del pedido |
| `id usuario` | ID de Telegram del cliente |
| `detalles pedido` | Detalle de productos y cantidades |
| `total pago` | Monto total del pedido |
| `estado` | Estado del pedido (pendiente, en preparación, entregado, etc.) |
| `fecha` | Fecha del pedido |
| `hora` | Hora del pedido |

---

## 💬 Flujo de conversación

1. El usuario escribe `/start`, `hola` o `menu` → se registra (si es nuevo) y se le da la bienvenida.
2. Elige una categoría (**Bebidas**, **Comidas** o **Snacks**) → el bot muestra los productos disponibles.
3. Selecciona un producto por número → el bot pide la cantidad deseada.
4. Ingresa la cantidad → se valida contra el stock disponible y se agrega al carrito.
5. Escribe `carrito` para revisar los productos agregados.
6. Escribe `confirmar` para generar el pedido:
   - Se valida el stock en tiempo real.
   - Si hay stock: se guarda el pedido, se descuenta el stock, se notifica al administrador y se limpia la sesión.
   - Si no hay stock: se informa el error al usuario.
7. Escribe `pedidos` para consultar el estado de sus pedidos anteriores.

---

## ⌨️ Comandos disponibles

| Comando / texto | Acción |
|---|---|
| `/start`, `hola`, `menu`, `inicio` | Inicia o reinicia la conversación |
| `Bebidas` / `Comidas` / `Snacks` | Muestra el menú de esa categoría |
| *(número)* | Selecciona un producto o define una cantidad, según el contexto |
| `carrito` / `ver carrito` | Muestra el carrito actual |
| `confirmar` / `confirmo` | Confirma el pedido |
| `vaciar` / `vaciar carrito` | Vacía el carrito |
| `pedidos` / `mis pedidos` | Consulta el historial/estado de pedidos |
| `/estado <id_pedido> <nuevo_estado>` | *(admin)* Actualiza el estado de un pedido |
| Cualquier otro texto | Muestra el mensaje de ayuda |

---

## ✅ Requisitos previos

- Cuenta de **n8n** (self-hosted o n8n Cloud).
- Un **bot de Telegram** creado con [@BotFather](https://t.me/BotFather) y su token.
- Una cuenta de **Google** con acceso a Google Sheets y credenciales OAuth2 configuradas en n8n.
- Una hoja de cálculo de Google Sheets con las 4 pestañas descritas arriba (`usuarios`, `sesiones`, `menu`, `pedidos`), con sus respectivas cabeceras de columnas ya creadas.

---

## 🚀 Instalación

1. Clona o descarga este repositorio.
2. Abre tu instancia de n8n.
3. Ve a **Workflows → Import from File** e importa el archivo `chatorduz.json`.
4. Configura las credenciales:
   - **Telegram API**: agrega el token de tu bot en el nodo `Telegram Trigger` y en los nodos de tipo Telegram (envío de mensajes).
   - **Google Sheets OAuth2**: conecta tu cuenta de Google en todos los nodos de Google Sheets.
5. En cada nodo de Google Sheets, actualiza el `documentId` para apuntar a tu propia hoja de cálculo.
6. Carga tu menú inicial en la pestaña `menu` (productos, precios, categorías y stock).
7. Activa el workflow (`Active`).

---

## ⚙️ Configuración

- **ID del administrador**: revisa el nodo `Router` / `Procesar Cambio Estado` para configurar qué usuario(s) de Telegram tienen permisos de administrador (comando `/estado`) y a dónde se envían las notificaciones de nuevos pedidos y el reporte diario.
- **Categorías del menú**: definidas en el nodo `Router` (`Bebidas`, `Comidas`, `Snacks`). Pueden ampliarse editando el código del nodo.
- **Horario del reporte diario**: configurado en el nodo `Reporte Diario` (Schedule Trigger), por defecto a las **20:00 h**.

---

## 🛠️ Panel de administrador

El administrador interactúa con el bot mediante el comando:

```
/estado <id_pedido> <nuevo_estado>
```

El workflow:
1. Verifica que el pedido exista (`¿Pedido Existe?`).
2. Actualiza el estado en la hoja `pedidos`.
3. Notifica automáticamente al cliente sobre el cambio de estado de su pedido.
4. Confirma la operación al administrador (o informa el error si el pedido no existe).

---

## 📊 Reporte diario automático

Todos los días a las **20:00 h**, el workflow:
1. Lee todos los pedidos registrados en la hoja `pedidos`.
2. Genera un resumen (`Armar Reporte`) con el total de pedidos y ventas del día.
3. Envía el reporte por Telegram al administrador.

---

## 🧩 Notas técnicas

- La conversación se maneja completamente **sin base de datos externa**: el estado de cada usuario (pantalla actual + carrito) se guarda y lee de la hoja `sesiones` en cada mensaje.
- El carrito se almacena como una cadena JSON dentro de la celda `carrito temporal`.
- El control de stock se realiza en tiempo real al momento de confirmar el pedido, evitando sobreventa.
- Todo el enrutamiento de la conversación se resuelve con nodos **Code** (JavaScript) y un nodo **Switch**, sin depender de servicios externos de NLP.

---

## 📄 Licencia

Este proyecto puede adaptarse libremente para otras cafeterías o negocios de venta de productos por catálogo vía Telegram.

---

## 📋 Índice

- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Estructura de Google Sheets](#-estructura-de-google-sheets)
- [Flujo de conversación](#-flujo-de-conversación)
- [Comandos disponibles](#-comandos-disponibles)
- [Requisitos previos](#-requisitos-previos)
- [Instalación](#-instalación)
- [Configuración](#-configuración)
- [Panel de administrador](#-panel-de-administrador)
- [Reporte diario automático](#-reporte-diario-automático)
- [Notas técnicas](#-notas-técnicas)

---

## ✨ Características

- 🤖 Bot de Telegram con manejo de sesión por usuario (máquina de estados).
- 📖 Menú dinámico leído desde Google Sheets, organizado por categorías (Bebidas, Comidas, Snacks).
- 🛒 Carrito de compras temporal por usuario.
- ✅ Validación de stock antes de confirmar un pedido.
- 📦 Registro automático de pedidos y descuento de stock en tiempo real.
- 🔔 Notificación al administrador por cada nuevo pedido.
- 🛠️ Comando de administrador para actualizar el estado de un pedido (`/estado`).
- 📊 Reporte diario automático de ventas enviado por Telegram.
- 🗂️ Persistencia total de usuarios, sesiones, menú y pedidos en Google Sheets (sin base de datos externa).

---

## 🏗️ Arquitectura

El proyecto es un único workflow de **n8n** dividido en bloques funcionales:

```
Telegram Trigger
      │
      ▼
Leer Sesión (Google Sheets)
      │
      ▼
Router (Code) ── interpreta el mensaje según el texto y la pantalla actual del usuario
      │
      ▼
Enrutar Acción (Switch) ── deriva el flujo según la acción detectada:
      │
      ├── start        → Registrar Usuario / Iniciar Sesión → Enviar Bienvenida
      ├── category      → Leer MENU (categoría) → Armar Lista Productos → Enviar Productos
      ├── pickproduct    → Leer MENU (elegir) → Elegir Producto → Enviar Pedir Cantidad
      ├── setqty        → Leer MENU (cantidad) → Validar Cantidad → Guardar Estado
      ├── cart          → Formatear Carrito → Enviar Carrito
      ├── clear         → Vaciar Carrito → Enviar Carrito Vaciado
      ├── confirm       → Validar y Generar Pedido → ¿Stock OK? → Guardar Pedido /
      │                    Descontar Stock / Notificar Pedido / Enviar Error Stock
      ├── status        → Leer PEDIDOS (usuario) → Formatear Pedidos → Enviar Pedidos
      ├── admin         → Leer PEDIDOS (admin) → Procesar Cambio Estado → ¿Pedido Existe? →
      │                    Actualizar Estado → Notificar Estado Usuario / Confirmar al Admin
      └── help          → Enviar Ayuda

Schedule Trigger (20:00) ── Reporte Diario
      │
      ▼
Leer PEDIDOS (reporte) → Armar Reporte → Enviar Reporte Admin
```

El bot funciona como una **máquina de estados**: cada usuario tiene una "pantalla actual" guardada en la hoja `sesiones`, que determina cómo se interpreta su siguiente mensaje (por ejemplo, si está eligiendo una categoría, un producto o una cantidad).

---

## 🗂️ Estructura de Google Sheets

El workflow utiliza una única hoja de cálculo (`deliverybot_ch`) con 4 pestañas:

### `usuarios`
| Columna | Descripción |
|---|---|
| `id telegram` | ID único del usuario en Telegram |
| `nombre completo` | Nombre del usuario (tomado de Telegram) |

### `sesiones`
| Columna | Descripción |
|---|---|
| `id telegram` | ID del usuario (clave de coincidencia) |
| `pantalla actual` | Estado actual de la conversación (home, cat:X, qty:X, etc.) |
| `carrito temporal` | Carrito en formato JSON (string) |
| `ultimo cambio` | Fecha/hora de la última interacción |

### `menu`
| Columna | Descripción |
|---|---|
| `id producto` | Identificador único del producto |
| `nombre` | Nombre del producto |
| `descripcion` | Descripción del producto |
| `precio` | Precio unitario |
| `categoria` | Bebidas / Comidas / Snacks |
| `stock` | Unidades disponibles |

### `pedidos`
| Columna | Descripción |
|---|---|
| `id pedido` | Identificador único del pedido |
| `id usuario` | ID de Telegram del cliente |
| `detalles pedido` | Detalle de productos y cantidades |
| `total pago` | Monto total del pedido |
| `estado` | Estado del pedido (pendiente, en preparación, entregado, etc.) |
| `fecha` | Fecha del pedido |
| `hora` | Hora del pedido |

---

## 💬 Flujo de conversación

1. El usuario escribe `/start`, `hola` o `menu` → se registra (si es nuevo) y se le da la bienvenida.
2. Elige una categoría (**Bebidas**, **Comidas** o **Snacks**) → el bot muestra los productos disponibles.
3. Selecciona un producto por número → el bot pide la cantidad deseada.
4. Ingresa la cantidad → se valida contra el stock disponible y se agrega al carrito.
5. Escribe `carrito` para revisar los productos agregados.
6. Escribe `confirmar` para generar el pedido:
   - Se valida el stock en tiempo real.
   - Si hay stock: se guarda el pedido, se descuenta el stock, se notifica al administrador y se limpia la sesión.
   - Si no hay stock: se informa el error al usuario.
7. Escribe `pedidos` para consultar el estado de sus pedidos anteriores.

---

## ⌨️ Comandos disponibles

| Comando / texto | Acción |
|---|---|
| `/start`, `hola`, `menu`, `inicio` | Inicia o reinicia la conversación |
| `Bebidas` / `Comidas` / `Snacks` | Muestra el menú de esa categoría |
| *(número)* | Selecciona un producto o define una cantidad, según el contexto |
| `carrito` / `ver carrito` | Muestra el carrito actual |
| `confirmar` / `confirmo` | Confirma el pedido |
| `vaciar` / `vaciar carrito` | Vacía el carrito |
| `pedidos` / `mis pedidos` | Consulta el historial/estado de pedidos |
| `/estado <id_pedido> <nuevo_estado>` | *(admin)* Actualiza el estado de un pedido |
| Cualquier otro texto | Muestra el mensaje de ayuda |

---

## ✅ Requisitos previos

- Cuenta de **n8n** (self-hosted o n8n Cloud).
- Un **bot de Telegram** creado con [@BotFather](https://t.me/BotFather) y su token.
- Una cuenta de **Google** con acceso a Google Sheets y credenciales OAuth2 configuradas en n8n.
- Una hoja de cálculo de Google Sheets con las 4 pestañas descritas arriba (`usuarios`, `sesiones`, `menu`, `pedidos`), con sus respectivas cabeceras de columnas ya creadas.

---

## 🚀 Instalación

1. Clona o descarga este repositorio.
2. Abre tu instancia de n8n.
3. Ve a **Workflows → Import from File** e importa el archivo `chatorduz.json`.
4. Configura las credenciales:
   - **Telegram API**: agrega el token de tu bot en el nodo `Telegram Trigger` y en los nodos de tipo Telegram (envío de mensajes).
   - **Google Sheets OAuth2**: conecta tu cuenta de Google en todos los nodos de Google Sheets.
5. En cada nodo de Google Sheets, actualiza el `documentId` para apuntar a tu propia hoja de cálculo.
6. Carga tu menú inicial en la pestaña `menu` (productos, precios, categorías y stock).
7. Activa el workflow (`Active`).

---

## ⚙️ Configuración

- **ID del administrador**: revisa el nodo `Router` / `Procesar Cambio Estado` para configurar qué usuario(s) de Telegram tienen permisos de administrador (comando `/estado`) y a dónde se envían las notificaciones de nuevos pedidos y el reporte diario.
- **Categorías del menú**: definidas en el nodo `Router` (`Bebidas`, `Comidas`, `Snacks`). Pueden ampliarse editando el código del nodo.
- **Horario del reporte diario**: configurado en el nodo `Reporte Diario` (Schedule Trigger), por defecto a las **20:00 h**.

---

## 🛠️ Panel de administrador

El administrador interactúa con el bot mediante el comando:

```
/estado <id_pedido> <nuevo_estado>
```

El workflow:
1. Verifica que el pedido exista (`¿Pedido Existe?`).
2. Actualiza el estado en la hoja `pedidos`.
3. Notifica automáticamente al cliente sobre el cambio de estado de su pedido.
4. Confirma la operación al administrador (o informa el error si el pedido no existe).

---

## 📊 Reporte diario automático

Todos los días a las **20:00 h**, el workflow:
1. Lee todos los pedidos registrados en la hoja `pedidos`.
2. Genera un resumen (`Armar Reporte`) con el total de pedidos y ventas del día.
3. Envía el reporte por Telegram al administrador.

---

## 🧩 Notas técnicas

- La conversación se maneja completamente **sin base de datos externa**: el estado de cada usuario (pantalla actual + carrito) se guarda y lee de la hoja `sesiones` en cada mensaje.
- El carrito se almacena como una cadena JSON dentro de la celda `carrito temporal`.
- El control de stock se realiza en tiempo real al momento de confirmar el pedido, evitando sobreventa.
- Todo el enrutamiento de la conversación se resuelve con nodos **Code** (JavaScript) y un nodo **Switch**, sin depender de servicios externos de NLP.

---

## 📄 Licencia

Este proyecto puede adaptarse libremente para otras cafeterías o negocios de venta de productos por catálogo vía Telegram.
