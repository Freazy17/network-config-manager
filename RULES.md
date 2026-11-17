# Reglas de Gobernanza y Estándares Técnicos (`RULES.md`)

**Proyecto:** Network Config Manager  
**Rol Rector:** Lead System Architect  
**Estado:** Obligatorio (Enforcement Estricto)

Este documento define las reglas arquitectónicas, restricciones tecnológicas y estándares de seguridad que gobiernan el desarrollo y mantenimiento de **Network Config Manager**. Cualquier contribución o módulo que incumpla estos lineamientos será rechazado.

---

## 1. Frontend: Stack Nativo Zero-Build

El cliente web debe mantenerse ligero, auditable y libre de cadenas de compilación complejas.

* **HTML5 Semántico:** Uso estricto de etiquetas semánticas (`<header>`, `<main>`, `<section>`, `<article>`, `<nav>`, `<dialog>`, `<table>`, `<form>`) para estructurar la interfaz.
* **Estilos con Tailwind CSS vía CDN:** El diseño visual se implementará exclusivamente cargando Tailwind CSS desde su CDN oficial (`https://cdn.tailwindcss.com`). No se permite compilación local de CSS mediante PostCSS o CLI de Node.
* **JavaScript Nativo (Vanilla ES6+):**
  * Comunicación asíncrona con el backend exclusivamente mediante la API estándar `fetch` (`async`/`await`).
  * Actualización de estado en vista mediante manipulación directa del DOM (`document.getElementById`, `document.querySelector`, `document.createElement`, `Element.innerHTML` / `textContent`).
* **Tecnologías Prohibidas:**
  * **PROHIBIDO** el uso de Node.js, npm, yarn, pnpm o cualquier gestor de paquetes de JavaScript.
  * **PROHIBIDO** el uso de empaquetadores o *bundlers* como Webpack, Vite, Rollup, Parcel o esbuild.
  * **PROHIBIDO** el uso de frameworks o librerías de UI como React, Vue, Angular, Svelte, Alpine.js o jQuery.

---

## 2. Seguridad y Gestión Criptográfica de Credenciales

Las credenciales de administración de los equipos de red son activos críticos y deben protegerse en reposo y en tránsito interno.

* **Política de Cero Texto Plano (Zero Plaintext Policy):**
  * Queda estrictamente prohibido almacenar contraseñas en texto plano en la base de datos, escribirlas en archivos de log, imprimirlas en consola o exponerlas en las respuestas JSON de la API.
* **Cifrado Simétrico Obligatorio:**
  * Toda contraseña de dispositivo debe cifrarse antes de persistirse utilizando `cryptography.fernet.Fernet` (AES-128 en modo CBC con firma de integridad HMAC-SHA256).
  * El descifrado (`fernet.decrypt()`) solo puede ocurrir en memoria volátil inmediatamente antes de instanciar la conexión SSH hacia el dispositivo y la variable en texto claro debe descartarse al finalizar el bloque de ejecución.
* **Custodia de la Clave Maestra (`ENCRYPTION_KEY`):**
  * La clave simétrica debe residir exclusivamente en una variable de entorno definida en el archivo `.env` en la raíz local del servidor.
  * **PROHIBIDO** almacenar la clave en la base de datos SQLite.
  * **PROHIBIDO** codificar la clave directamente en el código fuente (*hardcoding*).
  * **PROHIBIDO** versionar el archivo `.env` en el repositorio Git (debe estar declarado obligatoriamente en `.gitignore`).

---

## 3. Automatización de Red (Netmiko)

La interacción con el plano de control/administración de los equipos de red debe ser determinista y tolerante a fallos de conectividad.

* **Driver y Perfil Estándar:**
  * Uso exclusivo de la librería `netmiko` (`ConnectHandler`) utilizando el identificador de plataforma `cisco_ios` para los routers administrados.
* **Control Estricto de Timeouts:**
  * Toda ejecución de comando de extracción (ej. `send_command("show running-config")`) debe especificar explícitamente `read_timeout=30` (30 segundos, calibrado para Cisco IOSv/QEMU real; `timeout`, `conn_timeout`, `auth_timeout` y `banner_timeout` también ≥ 30 s, `global_delay_factor=2.0`) para evitar que sesiones colgadas bloqueen los hilos del servidor web.
* **Manejo Obligatorio y Explícito de Excepciones:**
  * Se prohíbe el uso de bloques `except Exception:` genéricos sin antes interceptar de forma explícita las excepciones nativas de Netmiko:
    1. `netmiko.exceptions.NetmikoTimeoutException`: Captura fallos de alcanzabilidad IP, caídas de la interfaz virtual Host-Only/GNS3 o bloqueos en el transporte SSH. Debe retornar un error controlado (HTTP `504 Gateway Timeout` o `502 Bad Gateway`).
    2. `netmiko.exceptions.NetmikoAuthenticationException`: Captura rechazos de usuario/contraseña en el proceso de autenticación SSH del router. Debe retornar un error controlado (HTTP `401 Unauthorized` o `422 Unprocessable Entity`).
  * Toda sesión SSH abierta debe cerrarse de forma garantizada en un bloque `finally` (`connection.disconnect()`) o mediante el uso de *context managers* (`with ConnectHandler(...) as conn:`).

---

## 4. Motor de Comparación de Configuraciones (Diffs)

El cálculo de diferencias entre respaldos es responsabilidad exclusiva del backend para garantizar consistencia y rendimiento.

* **Procesamiento Server-Side:**
  * Las comparaciones entre dos versiones de configuración (`v1` y `v2`) se procesarán en Python utilizando `difflib.unified_diff` de la librería estándar.
* **Normalización de Entrada y Salida:**
  * Las configuraciones crudas (`raw_config`) se dividirán en listas de líneas normalizadas (`splitlines(keepends=False)`) antes de invocarse `difflib.unified_diff(v1_lines, v2_lines, fromfile=f"backup_{v1}", tofile=f"backup_{v2}", lineterm="")`.
  * El frontend únicamente recibirá el resultado ya procesado en formato JSON y se limitará a representarlo visualmente con código de colores semántico (`+` adiciones en verde, `-` supresiones en rojo, `@@` metadatos de bloque).

---

## 5. Capa de Persistencia (SQLite)

El almacenamiento de inventario e historial de respaldos debe ser ligero, atómico e inmune a inyecciones SQL.

* **SQLite Puro:**
  * Uso directo del módulo estándar `sqlite3` de Python sobre un archivo local de base de datos (ej. `network_manager.db`).
  * Habilitación obligatoria de integridad referencial en cada conexión mediante `PRAGMA foreign_keys = ON;`.
* **Prevención Absoluta de SQL Injection:**
  * **PROHIBIDO** construir consultas SQL concatenando cadenas, usando `f-strings`, `.format()` o el operador `%`.
  * **OBLIGATORIO** el uso de consultas parametrizadas directas empleando marcadores posicionales `?` (ej. `cursor.execute("SELECT * FROM devices WHERE id = ?", (device_id,))`).

---

## Matriz de Cumplimiento Rápido

| Dominio | Permitido / Obligatorio | Prohibido |
| :--- | :--- | :--- |
| **Frontend** | HTML5 semántico, Tailwind CDN, JS nativo (`fetch`, DOM) | Node.js, npm, Vite/Webpack, React/Vue/Angular |
| **Seguridad** | `cryptography.fernet.Fernet`, clave en `.env` | Contraseñas en texto plano, clave en BD o en Git |
| **Red / SSH** | `netmiko` (`cisco_ios`), `read_timeout=30`, captura explícita de Timeout/Auth | Conexiones sin timeout, excepciones silenciadas |
| **Comparación** | `difflib.unified_diff` ejecutado en backend | Librerías JS de diff en cliente, comparación manual |
| **Base de Datos** | `sqlite3` con consultas parametrizadas (`?`) | Interpolación de strings en SQL (`f"SELECT ... {id}"`) |

