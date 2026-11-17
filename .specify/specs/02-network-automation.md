# Especificación Técnica Funcional: 02 - Módulo de Automatización de Red y Motor de Diffs

* **ID de Especificación:** `SPEC-02-NET-AUTOMATION`
* **Estado:** Aprobado para Implementación
* **Documentos Rectores:** [`RULES.md`](../../RULES.md), [`ARCHITECTURE.md`](../../ARCHITECTURE.md), [`01-database-and-inventory.md`](./01-database-and-inventory.md)
* **Módulo Objetivo:** `net_automation.py`

---

## 1. Propósito y Alcance

Este documento define la especificación técnica y funcional del módulo **`net_automation.py`**, responsable de:
1. Establecer sesiones SSHv2 deterministas y seguras desde el **Host Windows** hacia el router **Cisco c7200 (`R1`)** ejecutándose en **GNS3 VM** a través de la interfaz virtual Host-Only (`VMnet` / Nodo Cloud).
2. Extraer la configuración en ejecución (`show running-config`) aplicando parámetros defensivos de tiempo de espera (`read_timeout=10`), desactivación de modo rápido (`fast_cli=False`) y control de buffers de terminal.
3. Manejar de forma explícita y diferenciada las excepciones de red, autenticación y sistema operativo (`NetmikoTimeoutException`, `NetmikoAuthenticationException`, `socket.error` / `OSError`).
4. Calcular en el backend las diferencias línea por línea entre dos versiones de configuración utilizando `difflib.unified_diff`.

---

## 2. Arquitectura del Módulo `net_automation.py`

### 2.1. Dependencias Autorizadas

* **Librería Estándar de Python:**
  * `difflib` (`unified_diff`)
  * `socket` (`error`, `timeout`)
  * `typing` (`Dict`, `Any`, `List`, `Optional`)
  * `re` (normalización opcional de encabezados volátiles de IOS)
* **Librerías Externas Gobernadas por `RULES.md`:**
  * `netmiko` (`ConnectHandler`)
  * `netmiko.exceptions` (`NetmikoTimeoutException`, `NetmikoAuthenticationException`, `NetmikoBaseException`)
* **Módulos Internos:**
  * `crypto` (`decrypt_password`) para descifrar en memoria volátil el campo `encrypted_password` cuando el diccionario provenga directamente de `database.get_device_by_id(..., include_encrypted_password=True)`.

---

### 2.2. Parámetros Defensivos de Conexión Netmiko (`cisco_ios`)

Los routers Cisco serie 7200 emulados en Dynamips/GNS3 presentan latencia variable en la generación de salida por consola/VTY (especialmente al serializar `show running-config`). Para evitar truncamiento de datos, corrupción de prompt o bloqueos indefinidos de hilos en el backend, toda instancia de `ConnectHandler` **DEBE** construirse con el siguiente perfil defensivo:

```python
netmiko_params = {
    "device_type": device_dict.get("device_type", "cisco_ios"),  # Estrictamente 'cisco_ios'
    "host": device_dict["ip_address"],
    "username": device_dict["username"],
    "password": plain_password,       # Descifrada en memoria volátil con decrypt_password()
    "port": int(device_dict.get("port", 22)),
    "fast_cli": False,                # OBLIGATORIO: Evita saturar el buffer VTY en Dynamips/GNS3
    "timeout": 10,                    # Timeout de socket TCP/SSH inicial (segundos)
    "conn_timeout": 10,               # Timeout de establecimiento de canal SSH
    "auth_timeout": 10,               # Timeout de negociación de credenciales AAA/local
    "banner_timeout": 15,             # Margen para banners MOTD/Login extensos en IOS
    "global_delay_factor": 1.0,       # Factor base de cadencia de lectura de buffer
}
```

#### Control de Buffers y Ejecución de Comandos
1. **Desactivación de Paginación (`terminal length 0`):** Con `fast_cli=False`, Netmiko ejecuta automáticamente `terminal length 0` y `terminal width 511` al iniciar la sesión (`session_preparation`).
2. **Limpieza Explícita de Buffer (`clear_buffer()`):** Antes de enviar `show running-config`, el módulo debe invocar `connection.clear_buffer()` para descartar mensajes residuales de syslog en la VTY (ej. `%SYS-5-CONFIG_I`) que pudieran contaminar la captura.
3. **Timeout de Lectura Estricto (`read_timeout=10`):** La llamada a `send_command` debe fijar explícitamente `read_timeout=10`, además de `strip_prompt=True` y `strip_command=True`:
   ```python
   raw_output = connection.send_command(
       command_string="show running-config",
       read_timeout=10,
       strip_prompt=True,
       strip_command=True
   )
   ```
4. **Cierre Garantizado del Canal SSH:** El uso de *context manager* (`with ConnectHandler(**netmiko_params) as connection:`) o un bloque `try...finally` con `connection.disconnect()` es obligatorio para liberar inmediatamente la línea `vty` en el router `R1`.

---

### 2.3. Manejo Explícito de Excepciones

Queda prohibido capturar `Exception` de forma genérica sin antes interceptar y clasificar explícitamente las excepciones de red y autenticación.

| Excepción Capturada | Causa Técnica en la Topología | Comportamiento en `test_connection()` | Comportamiento en `fetch_running_config()` |
| :--- | :--- | :--- | :--- |
| `NetmikoAuthenticationException` | Usuario o contraseña rechazados por la configuración `login local` / AAA del router `R1`. | Captura explícita, registra diagnóstico y retorna `False`. | Captura explícita y relanza `NetmikoAuthenticationException` con mensaje descriptivo para que la capa API retorne `HTTP 401 Unauthorized`. |
| `NetmikoTimeoutException` | IP inalcanzable en la red Host-Only (`VMnet`), router `R1` apagado en GNS3 o comando superó `read_timeout=10`. | Captura explícita, registra diagnóstico y retorna `False`. | Captura explícita y relanza `NetmikoTimeoutException` con mensaje descriptivo para que la capa API retorne `HTTP 504 Gateway Timeout`. |
| `(socket.error, OSError, EOFError)` | Puerto TCP/22 cerrado (`ConnectionRefusedError`), caída abrupta de la interfaz virtual o corte de transporte SSH. | Captura explícita, registra diagnóstico y retorna `False`. | Captura explícita y lanza `ConnectionError` (o `RuntimeError` de transporte) detallando el fallo de socket/OS para que la capa API retorne `HTTP 502 Bad Gateway`. |

---

## 3. Especificación de Funciones Requeridas

### 3.1. Resolución Segura de Credenciales en Memoria

Tanto `test_connection` como `fetch_running_config` reciben `device_dict: Dict[str, Any]`. Para soportar tanto diccionarios provenientes de la base de datos (`encrypted_password`) como diccionarios de prueba directa (`password` / `plain_password`), se define una función auxiliar interna `_resolve_credentials(device_dict: Dict[str, Any]) -> Dict[str, Any]`:
* Si `device_dict` contiene `"encrypted_password"`, invoca `decrypt_password(device_dict["encrypted_password"])` en memoria volátil.
* Si `device_dict` contiene `"plain_password"` o `"password"`, utiliza dicho valor en memoria.
* Nunca modifica ni muta el `device_dict` original con la contraseña en claro.

---

### 3.2. `test_connection(device_dict: Dict[str, Any]) -> bool`

#### Firma y Contrato
```python
def test_connection(device_dict: Dict[str, Any]) -> bool:
    """
    Verifica la alcanzabilidad SSH y validez de credenciales contra un dispositivo Cisco IOS.

    Args:
        device_dict (Dict[str, Any]): Diccionario con las claves:
            - 'ip_address' (str): Dirección IPv4 del router (ej. '192.168.56.101').
            - 'username' (str): Usuario SSH.
            - 'encrypted_password' (str) o 'password' (str): Credencial del dispositivo.
            - 'device_type' (str, opcional): Por defecto 'cisco_ios'.

    Returns:
        bool: True si la sesión SSH se establece y el router devuelve un prompt IOS válido;
              False si ocurre NetmikoTimeoutException, NetmikoAuthenticationException o error de socket/OS.
    """
```

#### Algoritmo Paso a Paso
1. Extraer y validar la presencia de `ip_address`, `username` y la credencial (`encrypted_password` o `password`).
2. Construir los parámetros defensivos de Netmiko (`device_type="cisco_ios"`, `fast_cli=False`, `timeout=10`, `conn_timeout=10`, `auth_timeout=10`).
3. Abrir bloque `try`:
   * Instanciar `with ConnectHandler(**params) as conn:`.
   * Limpiar el buffer de entrada (`conn.clear_buffer()`).
   * Solicitar el prompt activo mediante `prompt = conn.find_prompt()`.
   * Retornar `bool(prompt and len(prompt.strip()) > 0)`.
4. Bloques `except` explícitos:
   * `except NetmikoAuthenticationException:` $\rightarrow$ retornar `False`.
   * `except NetmikoTimeoutException:` $\rightarrow$ retornar `False`.
   * `except (socket.error, OSError, EOFError, NetmikoBaseException):` $\rightarrow$ retornar `False`.
5. Bloque `finally`:
   * Eliminar la referencia local a la contraseña en texto plano (`del plain_password`).

---

### 3.3. `fetch_running_config(device_dict: Dict[str, Any]) -> str`

#### Firma y Contrato
```python
def fetch_running_config(device_dict: Dict[str, Any]) -> str:
    """
    Conecta vía SSH al dispositivo Cisco IOS, ejecuta 'show running-config' con
    read_timeout=10 y retorna la configuración limpia.

    Args:
        device_dict (Dict[str, Any]): Diccionario con 'ip_address', 'username',
            'encrypted_password' (o 'password') y 'device_type' ('cisco_ios').

    Returns:
        str: Configuración en ejecución normalizada (sin encabezados transitorios de volcado).

    Raises:
        NetmikoAuthenticationException: Si las credenciales SSH son rechazadas por el router.
        NetmikoTimeoutException: Si el router no responde dentro de read_timeout=10.
        ConnectionError: Si ocurre un fallo de socket/OS subyacente.
        ValueError: Si la salida devuelta está vacía o contiene un error de sintaxis IOS ('% Invalid input').
    """
```

#### Algoritmo Paso a Paso
1. Resolver la contraseña en memoria volátil mediante `_resolve_credentials(device_dict)`.
2. Abrir sesión con `with ConnectHandler(**params) as conn:` (`fast_cli=False`).
3. Ejecutar `conn.clear_buffer()` para vaciar cualquier mensaje asíncrono previo en la terminal.
4. Ejecutar el comando con timeout determinista:
   ```python
   raw_config = conn.send_command(
       "show running-config",
       read_timeout=10,
       strip_prompt=True,
       strip_command=True,
   )
   ```
5. **Sanitización y Control de Calidad del Buffer:**
   * Verificar que `raw_config` no esté vacío (`if not raw_config or not raw_config.strip(): raise ValueError(...)`).
   * Verificar que no contenga errores de privilegio o comando inválido de IOS (`% Invalid input detected`, `% Incomplete command`, `Command authorization failed`).
   * Normalizar saltos de línea (`\r\n` $\rightarrow$ `\n`) y eliminar las líneas de estado transitorias del comando `show running-config` (`Building configuration...` y `Current configuration : <N> bytes`) para almacenar únicamente la configuración declarativa pura del router.
6. Retornar la cadena `cleaned_config`.
7. Capturar explícitamente:
   * `except NetmikoAuthenticationException as auth_err:` $\rightarrow$ relanzar `NetmikoAuthenticationException` con contexto del host.
   * `except NetmikoTimeoutException as timeout_err:` $\rightarrow$ relanzar `NetmikoTimeoutException` indicando expiración de `read_timeout=10`.
   * `except (socket.error, OSError, EOFError) as os_err:` $\rightarrow$ lanzar `ConnectionError(f"Error de socket/OS al conectar con {host}: {os_err}") from os_err`.

---

### 3.4. `calculate_diff(old_config: str, new_config: str) -> List[str]`

#### Firma y Contrato
```python
def calculate_diff(
    old_config: str,
    new_config: str,
    fromfile: str = "old_config",
    tofile: str = "new_config",
) -> List[str]:
    """
    Calcula las diferencias unificadas entre dos configuraciones de texto usando difflib.unified_diff.

    Args:
        old_config (str): Configuración base (v1).
        new_config (str): Configuración nueva a comparar (v2).
        fromfile (str, opcional): Etiqueta de la versión origen. Por defecto 'old_config'.
        tofile (str, opcional): Etiqueta de la versión destino. Por defecto 'new_config'.

    Returns:
        List[str]: Lista de líneas del diff en formato unificado estricto (sin saltos de línea finales
                   en cada elemento), incluyendo encabezados ('---', '+++'), bloques ('@@')
                   y marcadores ('+', '-', ' '). Si no hay cambios, retorna [].
    """
```

#### Algoritmo Paso a Paso
1. Validar que `old_config` y `new_config` sean cadenas (`str`); de lo contrario lanzar `TypeError` o `ValueError`.
2. Normalizar los finales de línea (`\r\n` $\rightarrow$ `\n`) y dividir ambas configuraciones en listas de líneas sin caracteres de salto al final:
   ```python
   old_lines = old_config.replace("\r\n", "\n").splitlines(keepends=False)
   new_lines = new_config.replace("\r\n", "\n").splitlines(keepends=False)
   ```
3. Ejecutar `difflib.unified_diff`:
   ```python
   diff_generator = difflib.unified_diff(
       old_lines,
       new_lines,
       fromfile=fromfile,
       tofile=tofile,
       lineterm="",
   )
   return list(diff_generator)
   ```
4. **Regla de Formato Estricto:**
   * Si `old_lines == new_lines`, `list(diff_generator)` retorna exactamente `[]` (lista vacía).
   * Si existen diferencias, la lista retornada contiene:
     * Índice `0`: `"--- <fromfile>"`
     * Índice `1`: `"+++ <tofile>"`
     * Encabezados de bloque: `"@@ -l,s +l,s @@"`
     * Líneas suprimidas respecto a `old_config`: prefijo `"-"` (ej. `"-hostname R1"`).
     * Líneas añadidas en `new_config`: prefijo `"+"` (ej. `"+hostname R1-CORE"`).
     * Líneas de contexto sin cambio: prefijo `" "` (un espacio en blanco).

---

## 4. Criterios de Aceptación y Plan de Pruebas

La validación del módulo `net_automation.py` comprende dos niveles obligatorios: **Pruebas con Simulación (Mocking)** para ejecución determinista sin hardware y **Pruebas en Vivo contra `R1` (Cisco c7200 en GNS3)** cuando el laboratorio está activo.

---

### 4.1. Pruebas con Simulación (`unittest.mock`)

#### Caso M1: `test_connection()` Exitoso con Mock
* **Configuración:** Parchear `net_automation.ConnectHandler` con `unittest.mock.patch` para que el *context manager* devuelva un objeto mock cuyo método `find_prompt()` retorne `"R1#"`.
* **Acción:** Invocar `result = test_connection(device_dict)`.
* **Verificación:**
  * `result is True`.
  * `ConnectHandler` fue invocado con `device_type="cisco_ios"`, `fast_cli=False` y la contraseña descifrada en claro.
  * `clear_buffer()` y `find_prompt()` fueron invocados una vez.

#### Caso M2: Manejo Explícito de `NetmikoTimeoutException`
* **Configuración:** Configurar el mock de `ConnectHandler` (o `send_command`) para lanzar `NetmikoTimeoutException("Timed-out reading channel, data not available.")`.
* **Acción y Verificación:**
  1. `test_connection(device_dict)` captura la excepción y retorna `False` sin colapsar el proceso.
  2. `fetch_running_config(device_dict)` captura y propaga explícitamente `NetmikoTimeoutException`.

#### Caso M3: Manejo Explícito de `NetmikoAuthenticationException`
* **Configuración:** Configurar el mock de `ConnectHandler` para lanzar `NetmikoAuthenticationException("Authentication to device failed.")`.
* **Acción y Verificación:**
  1. `test_connection(device_dict)` retorna `False`.
  2. `fetch_running_config(device_dict)` propaga explícitamente `NetmikoAuthenticationException`.

#### Caso M4: Manejo Explícito de Errores de Socket/OS
* **Configuración:** Configurar el mock de `ConnectHandler` para lanzar `socket.error("Connection refused")` o `OSError(10061, "No connection could be made")`.
* **Acción y Verificación:**
  1. `test_connection(device_dict)` retorna `False`.
  2. `fetch_running_config(device_dict)` lanza `ConnectionError` encapsulando la causa original.

#### Caso M5: Extracción y Limpieza de `running-config` (`fetch_running_config`)
* **Configuración:** Configurar `conn_mock.send_command.return_value` con una salida realista de Cisco c7200 que incluya los encabezados `Building configuration...` y `Current configuration : 1420 bytes\n!\nversion 15.2\nhostname R1\n...\nend`.
* **Acción:** `config = fetch_running_config(device_dict)`.
* **Verificación:**
  * `conn_mock.send_command` fue llamado exactamente con `("show running-config", read_timeout=10, strip_prompt=True, strip_command=True)` (o argumentos equivalentes con `read_timeout=10`).
  * `"Building configuration..." not in config` y `"Current configuration :" not in config`.
  * `"hostname R1" in config` y `config.strip().endswith("end")`.

---

### 4.2. Criterios de Formateo Estricto del Diff (`calculate_diff`)

#### Caso D1: Detección Exacta de Líneas Añadidas (`+`) y Eliminadas (`-`)
* **Entrada:**
  ```python
  old_cfg = "!\nhostname R1\n!\ninterface FastEthernet0/0\n ip address 192.168.56.101 255.255.255.0\n no shutdown\n!\nend"
  new_cfg = "!\nhostname R1-GNS3\n!\ninterface FastEthernet0/0\n description WAN-LINK\n ip address 192.168.56.101 255.255.255.0\n no shutdown\n!\nend"
  ```
* **Acción:** `diff = calculate_diff(old_cfg, new_cfg)`
* **Verificación:**
  * `isinstance(diff, list)` y `len(diff) > 0`.
  * `diff[0].startswith("---")` y `diff[1].startswith("+++")`.
  * Contiene exactamente la línea eliminada `"-hostname R1"`.
  * Contiene exactamente las líneas añadidas `"+hostname R1-GNS3"` y `"+ description WAN-LINK"`.
  * Ningún elemento de `diff` termina en `\n` o `\r\n` (`lineterm=""`).

#### Caso D2: Configuraciones Idénticas Sin Falsos Positivos
* **Entrada:** `old_cfg` y `new_cfg` con el mismo contenido (incluso si una tiene finales de línea `\r\n` de Windows/Telnet y otra `\n` de Unix).
* **Acción:** `diff = calculate_diff(old_cfg, new_cfg)`
* **Verificación:** `diff == []`.

---

### 4.3. Pruebas en Vivo contra `R1` (Cisco c7200 en GNS3 VM)

* **Precondiciones del Entorno GNS3:**
  1. GNS3 VM en ejecución con el router Cisco c7200 (`R1`) encendido.
  2. Interfaz de gestión de `R1` conectada al nodo Cloud (puente Host-Only `VMnet`) con IP alcanzable desde el Host Windows (verificable con `ping`).
  3. SSHv2 operativo en `R1` (`ip domain-name`, `crypto key generate rsa`, usuario local y `transport input ssh` en `line vty 0 4`).
* **Secuencia de Verificación en Vivo:**
  1. Invocar `test_connection(r1_device)` $\rightarrow$ **Debe retornar `True`**.
  2. Invocar `cfg_1 = fetch_running_config(r1_device)` $\rightarrow$ **Debe retornar un `str` con longitud $> 100$ bytes que contenga `hostname` y `end`**.
  3. Realizar un cambio menor o simular una segunda versión `cfg_2` y pasar ambas a `calculate_diff(cfg_1, cfg_2)` $\rightarrow$ **Debe reflejar únicamente las líneas modificadas con prefijos `+` y `-`**.

