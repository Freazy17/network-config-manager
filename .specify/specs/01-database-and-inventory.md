# Especificación Técnica Funcional: 01 - Módulo de Base de Datos e Inventario

* **ID de Especificación:** `SPEC-01-DB-INVENTORY`
* **Estado:** Aprobado para Implementación
* **Documentos Rectores:** [`RULES.md`](../../RULES.md), [`ARCHITECTURE.md`](../../ARCHITECTURE.md)
* **Módulos Objetivo:** `crypto.py`, `database.py`

---

## 1. Propósito y Alcance

Este documento especifica el diseño técnico y funcional de la capa de persistencia en **SQLite puro** y el subsistema criptográfico basado en **`cryptography.fernet.Fernet`** para **Network Config Manager**.

El módulo garantiza:
1. Almacenamiento relacional íntegro de dispositivos de red (`devices`) e instantáneas de configuración (`backups`).
2. Cumplimiento estricto de la política **Zero Plaintext Password**: ninguna contraseña se persiste en claro en disco ni se expone en listados de API.
3. Inmunidad frente a ataques de **SQL Injection** mediante el uso exclusivo de consultas parametrizadas con marcadores posicionales (`?`).

---

## 2. Esquema SQLite (DDL, Llaves e Índices)

### 2.1. Directivas de Conexión Obligatorias

En SQLite, las restricciones de clave foránea están desactivadas por defecto en cada nueva conexión. Por tanto, toda conexión abierta por el módulo de acceso a datos **DEBE** ejecutar inmediatamente al conectarse:

```sql
PRAGMA foreign_keys = ON;
```

### 2.2. Sentencias DDL Exactas

El procedimiento de inicialización (`init_db()`) debe ejecutar de forma transaccional las siguientes sentencias DDL:

```sql
-- Habilitar integridad referencial en la sesión actual
PRAGMA foreign_keys = ON;

-- 1. Tabla de Inventario de Dispositivos
CREATE TABLE IF NOT EXISTS devices (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    hostname TEXT NOT NULL,
    ip_address TEXT NOT NULL UNIQUE,
    device_type TEXT NOT NULL DEFAULT 'cisco_ios',
    username TEXT NOT NULL,
    encrypted_password TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 2. Tabla de Historial de Respaldos de Configuración
CREATE TABLE IF NOT EXISTS backups (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id INTEGER NOT NULL,
    raw_config TEXT NOT NULL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (device_id) REFERENCES devices(id) ON DELETE CASCADE
);

-- 3. Índices de Rendimiento y Búsqueda
CREATE UNIQUE INDEX IF NOT EXISTS idx_devices_ip_address 
    ON devices(ip_address);

CREATE INDEX IF NOT EXISTS idx_backups_device_id 
    ON backups(device_id);

CREATE INDEX IF NOT EXISTS idx_backups_device_timestamp 
    ON backups(device_id, timestamp DESC);
```

### 2.3. Especificación de Restricciones e Índices

| Objeto | Tipo | Columna(s) | Propósito Arquitectónico |
| :--- | :--- | :--- | :--- |
| `PK_devices` | `PRIMARY KEY` | `devices(id)` | Identificador único autoincremental de cada nodo administrado. |
| `UQ_devices_ip` | `UNIQUE` + `INDEX` | `devices(ip_address)` | Evita duplicidad de direcciones IPv4 de gestión en el inventario y acelera validaciones de alta. |
| `PK_backups` | `PRIMARY KEY` | `backups(id)` | Identificador único autoincremental de cada versión de respaldo (`v1`, `v2`). |
| `FK_backups_device` | `FOREIGN KEY` | `backups(device_id)` $\rightarrow$ `devices(id)` | Garantiza que todo backup pertenezca a un dispositivo válido. Con `ON DELETE CASCADE`, al eliminar un dispositivo se purgan automáticamente sus respaldos huérfanos. |
| `idx_backups_device_id` | `INDEX` | `backups(device_id)` | Optimiza los `JOIN` y el borrado en cascada (`ON DELETE CASCADE`) desde la tabla padre `devices`. |
| `idx_backups_device_timestamp` | `INDEX` | `backups(device_id, timestamp DESC)` | Optimiza la recuperación ordenada cronológicamente de versiones de configuración por dispositivo para poblar la UI. |

---

## 3. Seguridad Criptográfica Fernet (`crypto.py`)

### 3.1. Gestión de la Clave Maestra (`ENCRYPTION_KEY`)

* La clave maestra simétrica reside exclusivamente en el archivo `.env` ubicado en la raíz del proyecto bajo la variable `ENCRYPTION_KEY`.
* Formato requerido: Clave de 32 bytes codificada en Base64 URL-safe (generada mediante `Fernet.generate_key().decode('utf-8')`).
* Ejemplo en `.env` (archivo **prohibido** en Git):
  ```ini
  ENCRYPTION_KEY=8yX2v9K1pL4mN7qR0tW3zA6cF9hJ2kM5nP8sV1xY4aE=
  ```

### 3.2. Especificación Funcional del Módulo `crypto.py`

El módulo `crypto.py` encapsula toda operación criptográfica y **nunca** almacena contraseñas en caché.

#### Dependencias Permitidas
* `os` (librería estándar)
* `dotenv.load_dotenv` (`python-dotenv`)
* `cryptography.fernet.Fernet`, `cryptography.fernet.InvalidToken`

#### Firmas de Función y Contratos

```python
from cryptography.fernet import Fernet, InvalidToken

def get_fernet() -> Fernet:
    """
    Carga las variables de entorno desde .env, recupera ENCRYPTION_KEY
    y retorna una instancia inicializada de Fernet.

    Raises:
        RuntimeError: Si la variable de entorno ENCRYPTION_KEY no está definida o está vacía.
        ValueError: Si ENCRYPTION_KEY no es una clave Fernet válida (32 bytes URL-safe base64).
    """

def encrypt_password(plain_password: str) -> str:
    """
    Cifra una contraseña en texto plano utilizando Fernet (AES-128-CBC + HMAC-SHA256).

    Args:
        plain_password (str): Contraseña en texto claro (no vacía).

    Returns:
        str: Token Fernet codificado en UTF-8 (ej. 'gAAAAABn...'), listo para persistir en SQLite.

    Raises:
        ValueError: Si plain_password es None o cadena vacía.
        RuntimeError: Si ENCRYPTION_KEY no está configurada válidamente.
    """

def decrypt_password(encrypted_password: str) -> str:
    """
    Descifra en memoria volátil un token Fernet almacenado en la base de datos.

    Args:
        encrypted_password (str): Token Fernet recuperado de la columna devices.encrypted_password.

    Returns:
        str: Contraseña original en texto plano para uso inmediato en ConnectHandler de Netmiko.

    Raises:
        ValueError: Si encrypted_password está vacío.
        InvalidToken: Si el token fue alterado, está corrupto o la clave ENCRYPTION_KEY no coincide.
    """
```

#### Flujo Interno de Cifrado y Descifrado
1. **En `encrypt_password(plain_password)`:**
   * Verifica que `isinstance(plain_password, str)` y `len(plain_password) > 0`.
   * Convierte la cadena a bytes: `plain_bytes = plain_password.encode("utf-8")`.
   * Ejecuta `token_bytes = get_fernet().encrypt(plain_bytes)`.
   * Retorna `token_bytes.decode("utf-8")`.
2. **En `decrypt_password(encrypted_password)`:**
   * Verifica que `isinstance(encrypted_password, str)` y `len(encrypted_password) > 0`.
   * Convierte el token a bytes: `token_bytes = encrypted_password.encode("utf-8")`.
   * Ejecuta `plain_bytes = get_fernet().decrypt(token_bytes)`.
   * Retorna `plain_bytes.decode("utf-8")`.

---

## 4. Funciones de Acceso a Datos (DAL - `database.py`)

### 4.1. Reglas de Implementación de la Capa DAL
* **Prohibición de Interpolación:** Ninguna consulta SQL puede utilizar `f-strings`, `.format()` ni concatenación `+`. Todos los valores dinámicos deben pasarse como tupla en el segundo argumento de `cursor.execute(sql, params)` usando el marcador `?`.
* **Factoría de Filas:** Configurar `conn.row_factory = sqlite3.Row` para transformar resultados en diccionarios serializables a JSON.
* **Cifrado en Frontera:** `add_device()` recibe `plain_password` desde la capa de negocio/API e invoca internamente `crypto.encrypt_password(plain_password)` antes de ejecutar el `INSERT`.

---

### 4.2. Especificación Detallada de Funciones

#### 4.2.1. `init_db()`
* **Firma:**
  ```python
  def init_db(db_path: str = "network_manager.db") -> None:
  ```
* **Responsabilidad:** Crea el archivo SQLite si no existe, activa `PRAGMA foreign_keys = ON;` y ejecuta las sentencias DDL de las tablas `devices`, `backups` y sus índices asociados.
* **Queries SQL Ejecutados:**
  ```sql
  PRAGMA foreign_keys = ON;

  CREATE TABLE IF NOT EXISTS devices (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      hostname TEXT NOT NULL,
      ip_address TEXT NOT NULL UNIQUE,
      device_type TEXT NOT NULL DEFAULT 'cisco_ios',
      username TEXT NOT NULL,
      encrypted_password TEXT NOT NULL,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  CREATE TABLE IF NOT EXISTS backups (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      device_id INTEGER NOT NULL,
      raw_config TEXT NOT NULL,
      timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (device_id) REFERENCES devices(id) ON DELETE CASCADE
  );

  CREATE UNIQUE INDEX IF NOT EXISTS idx_devices_ip_address ON devices(ip_address);
  CREATE INDEX IF NOT EXISTS idx_backups_device_id ON backups(device_id);
  CREATE INDEX IF NOT EXISTS idx_backups_device_timestamp ON backups(device_id, timestamp DESC);
  ```

---

#### 4.2.2. `add_device()`
* **Firma:**
  ```python
  def add_device(
      hostname: str,
      ip_address: str,
      username: str,
      plain_password: str,
      device_type: str = "cisco_ios",
      db_path: str = "network_manager.db"
  ) -> dict:
  ```
* **Responsabilidad:**
  1. Valida que `hostname`, `ip_address`, `username` y `plain_password` no estén vacíos.
  2. Cifra `plain_password` invocando `encrypted_pw = encrypt_password(plain_password)`.
  3. Inserta el registro mediante consulta parametrizada `?`.
  4. Recupera y retorna un diccionario con los metadatos del dispositivo recién creado (**excluyendo** `encrypted_password`).
* **Query SQL Parametrizado:**
  ```sql
  INSERT INTO devices (hostname, ip_address, device_type, username, encrypted_password)
  VALUES (?, ?, ?, ?, ?);
  ```
  * **Tupla de Parámetros:** `(hostname.strip(), ip_address.strip(), device_type.strip(), username.strip(), encrypted_pw)`
* **Query de Confirmación Post-Inserción:**
  ```sql
  SELECT id, hostname, ip_address, device_type, username, created_at
  FROM devices
  WHERE id = ?;
  ```
  * **Tupla de Parámetros:** `(cursor.lastrowid,)`
* **Excepciones:**
  * `sqlite3.IntegrityError`: Si `ip_address` ya existe en la base de datos (la capa API lo mapeará a `HTTP 409 Conflict`).

---

#### 4.2.3. `get_devices()`
* **Firma:**
  ```python
  def get_devices(db_path: str = "network_manager.db") -> list[dict]:
  ```
* **Responsabilidad:** Retorna la lista completa de dispositivos registrados ordenados por `id ASC`, incluyendo la lista resumida de metadatos de sus backups (`id`, `timestamp`) y **omitiendo estrictamente** la columna `encrypted_password`.
* **Queries SQL Parametrizados:**
  ```sql
  -- Consulta principal de inventario (sin encrypted_password)
  SELECT id, hostname, ip_address, device_type, username, created_at
  FROM devices
  ORDER BY id ASC;

  -- Consulta parametrizada de metadatos de backups por dispositivo
  SELECT id, timestamp
  FROM backups
  WHERE device_id = ?
  ORDER BY timestamp DESC, id DESC;
  ```
  * **Tupla de Parámetros (segunda consulta):** `(device_id,)`

---

#### 4.2.4. `get_device_by_id()`
* **Firma:**
  ```python
  def get_device_by_id(
      device_id: int,
      include_encrypted_password: bool = False,
      db_path: str = "network_manager.db"
  ) -> dict | None:
  ```
* **Responsabilidad:**
  * Busca un dispositivo específico por su clave primaria `id`.
  * Por defecto (`include_encrypted_password=False`), retorna únicamente las columnas públicas (`id`, `hostname`, `ip_address`, `device_type`, `username`, `created_at`).
  * Cuando es invocado por el motor de automatización SSH (`include_encrypted_password=True`), incluye el campo `encrypted_password` para que el servicio de backup pueda descifrarlo en memoria volátil con `decrypt_password()`.
  * Retorna `None` si el `device_id` no existe.
* **Queries SQL Parametrizados:**
  * Cuando `include_encrypted_password == False`:
    ```sql
    SELECT id, hostname, ip_address, device_type, username, created_at
    FROM devices
    WHERE id = ?;
    ```
  * Cuando `include_encrypted_password == True`:
    ```sql
    SELECT id, hostname, ip_address, device_type, username, encrypted_password, created_at
    FROM devices
    WHERE id = ?;
    ```
  * **Tupla de Parámetros:** `(device_id,)`

---

#### 4.2.5. `delete_device()`
* **Firma:**
  ```python
  def delete_device(
      device_id: int,
      db_path: str = "network_manager.db"
  ) -> bool:
  ```
* **Responsabilidad:**
  * Activa `PRAGMA foreign_keys = ON;` en la conexión para garantizar que SQLite ejecute la cláusula `ON DELETE CASCADE` sobre la tabla `backups`.
  * Elimina el dispositivo cuyo `id` coincida con `device_id`.
  * Retorna `True` si se eliminó el registro (`cursor.rowcount > 0`) o `False` si el `device_id` no existía.
* **Query SQL Parametrizado:**
  ```sql
  DELETE FROM devices
  WHERE id = ?;
  ```
  * **Tupla de Parámetros:** `(device_id,)`

---

## 5. Criterios de Aceptación y Casos de Prueba

La implementación de este módulo se considerará aceptada únicamente cuando supere la totalidad de los siguientes casos de prueba verificables.

### Caso de Prueba 1: Almacenamiento Cifrado en Disco (Zero Plaintext Verification)
* **Objetivo:** Confirmar que la contraseña en texto plano jamás se escribe como texto legible en el archivo SQLite en disco.
* **Precondición:** Variable `ENCRYPTION_KEY` válida cargada en entorno y base de datos inicializada con `init_db()`.
* **Entrada:**
  ```python
  plain_secret = "Cisco7200_Sup3rS3cr3t!"
  device = add_device(
      hostname="R1",
      ip_address="192.168.56.101",
      username="admin",
      plain_password=plain_secret
  )
  ```
* **Verificación Directa en SQLite:**
  1. Ejecutar consulta cruda `SELECT encrypted_password FROM devices WHERE id = ?` con `(device["id"],)`.
  2. **Aserción 1:** `row["encrypted_password"] != plain_secret`.
  3. **Aserción 2:** `plain_secret not in row["encrypted_password"]`.
  4. **Aserción 3:** `row["encrypted_password"].startswith("gAAAAA")` (encabezado estándar de versión `0x80` de tokens Fernet en Base64).
  5. **Aserción 4 (Inspección binaria en disco):** Leer los bytes crudos del archivo `.db` desde el sistema de archivos y comprobar que la secuencia de bytes `b"Cisco7200_Sup3rS3cr3t!"` **no** existe en ningún sector del archivo.

---

### Caso de Prueba 2: Recuperación y Descifrado Íntegro (Round-Trip Integrity)
* **Objetivo:** Confirmar que la contraseña cifrada en SQLite se recupera y descifra de forma exacta e intacta en memoria.
* **Pasos:**
  1. Invocar `record = get_device_by_id(device["id"], include_encrypted_password=True)`.
  2. Invocar `recovered_password = decrypt_password(record["encrypted_password"])`.
* **Resultado Esperado:**
  * `recovered_password == "Cisco7200_Sup3rS3cr3t!"` (igualdad exacta de caracteres, símbolos y longitud).

---

### Caso de Prueba 3: Ocultamiento de Credenciales en Consultas de Inventario
* **Objetivo:** Garantizar que `get_devices()` y `get_device_by_id(id, include_encrypted_password=False)` nunca filtren el campo `encrypted_password` ni la contraseña en claro hacia las respuestas de la API.
* **Pasos:**
  1. Invocar `devices_list = get_devices()`.
  2. Invocar `single_device = get_device_by_id(device["id"])`.
* **Resultado Esperado:**
  * `"encrypted_password" not in devices_list[0]` y `"password" not in devices_list[0]`.
  * `"encrypted_password" not in single_device` y `"password" not in single_device`.

---

### Caso de Prueba 4: Rechazo por Clave Inválida o Manipulación de Ciphertext (HMAC Verification)
* **Objetivo:** Confirmar que Fernet detecta cualquier alteración del texto cifrado o cambio no autorizado de `ENCRYPTION_KEY`.
* **Pasos:**
  1. Recuperar `encrypted_password` desde la base de datos.
  2. Modificar un carácter del token cifrado o intentar descifrarlo con una `ENCRYPTION_KEY` distinta.
* **Resultado Esperado:**
  * `decrypt_password()` lanza explícitamente `cryptography.fernet.InvalidToken` y no retorna datos corruptos.

---

### Caso de Prueba 5: Neutralización de Inyección SQL (Parameterized Queries)
* **Objetivo:** Validar que las consultas parametrizadas (`?`) impiden la ejecución de código SQL malicioso inyectado en campos de entrada.
* **Entrada Maliciosa:**
  ```python
  malicious_hostname = "R1'); DROP TABLE devices; --"
  add_device(
      hostname=malicious_hostname,
      ip_address="192.168.56.250",
      username="admin",
      plain_password="Password123!"
  )
  ```
* **Resultado Esperado:**
  * La tabla `devices` **no** es eliminada.
  * El registro se almacena tratando la cadena `"R1'); DROP TABLE devices; --"` como un literal de texto inofensivo en la columna `hostname`.

---

### Caso de Prueba 6: Integridad Referencial y Borrado en Cascada (`delete_device`)
* **Objetivo:** Confirmar que al eliminar un dispositivo con `delete_device(device_id)` se eliminan automáticamente todos sus respaldos asociados en `backups`.
* **Pasos:**
  1. Crear el dispositivo `R1` (`id = 1`) e insertar 2 registros asociados en la tabla `backups` con `device_id = 1`.
  2. Ejecutar `deleted = delete_device(1)`.
  3. Consultar `SELECT COUNT(*) FROM backups WHERE device_id = ?` con `(1,)`.
* **Resultado Esperado:**
  * `deleted is True`.
  * `get_device_by_id(1) is None`.
  * El conteo de backups asociados a `device_id = 1` es exactamente `0`.

