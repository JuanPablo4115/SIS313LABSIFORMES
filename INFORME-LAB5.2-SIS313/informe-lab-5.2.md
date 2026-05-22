# Informe de Laboratorio 5.2
## Detección de Intrusiones y Respuesta a Incidentes

---

**Universidad Mayor Real y Pontificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnología**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiante:** Juan Pablo Taboada Camacho — Ing. Sistemas

**Semestre:** 1/2026

---

## 1. Introducción

El presente laboratorio tiene como objetivo implementar un sistema de detección y prevención de intrusiones en un entorno virtualizado con VirtualBox, simulando un escenario real de ataque y defensa entre dos máquinas virtuales alojadas en la misma PC.

Se configuraron dos máquinas virtuales con Ubuntu Server 24.04 LTS: una actuando como **servidor objetivo** (con Nginx, SSH y Fail2ban) y otra como **máquina atacante** (con Hydra, Nmap y curl). El servidor actúa también como gateway para la VM atacante, proporcionando conectividad a internet mediante NAT y reenvío de paquetes.

Se simularon ataques de fuerza bruta SSH, escaneos de puertos y reconocimiento web, analizando los logs del sistema para identificar patrones de ataque. Finalmente se aplicó el ciclo completo de respuesta a incidentes: identificar, contener, erradicar, recuperar y documentar.

---

## 2. Objetivos del Laboratorio

- **Configurar Fail2ban** como sistema de prevención de intrusiones basado en host (HIPS) para proteger los servicios SSH y web.

- **Simular ataques de fuerza bruta** y escaneos de puertos desde una VM atacante utilizando Hydra y Nmap.

- **Analizar logs del sistema** (`auth.log`, `access.log`) para identificar patrones de ataque y actividad sospechosa.

- **Aplicar el ciclo de respuesta a incidentes:** identificar, contener, erradicar, recuperar y documentar.

- **Comprender la importancia de la auditoría** y el monitoreo de logs en la ciberseguridad.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
Internet
     |
  [NAT VirtualBox]
     |
  [VM Lab5.2-Server — enp0s3 (NAT) — 10.0.2.15]
     |
  [Red Interna — 192.168.30.0/29]
     |
  ┌──┴──────────────────────────────────┐
  |                                     |
[VM Lab5.2-Server]           [VM Lab5.2-Attacker]
 192.168.30.2                 192.168.30.3
 Nginx + SSH + Fail2ban       Hydra + Nmap + curl
 (Gateway para Attacker)
```

### 3.2 Tabla de Direccionamiento

| Máquina Virtual | Hostname | Rol | Interfaces | IP Interna |
|---|---|---|---|---|
| `Lab5.2-Server` | `SERVIDOR` | Servidor Objetivo (Nginx, SSH, Fail2ban) | NAT + Red Interna | `192.168.30.2/29` |
| `Lab5.2-Attacker` | `ATACANTE` | Máquina Atacante (Hydra, Nmap, curl) | Red Interna | `192.168.30.3/29` |

| Parámetro | Valor |
|---|---|
| Red interna | `192.168.30.0/29` |
| Gateway (Server → Attacker) | `192.168.30.2` |
| Interfaz NAT del Server | `enp0s3` — `10.0.2.15` |
| Interfaz interna del Server | `enp0s8` — `192.168.30.2` |
| Puerto SSH (host anfitrión) | `2222 → 22` |
| Puerto HTTP (host anfitrión) | `8080 → 80` |

### 3.3 Esquema de Red Completo

```
┌───────────────────────────────────────────────────────────────────┐
│                        PC ANFITRIONA                              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              VirtualBox — Red Interna (192.168.30.0/29)     │  │
│  │                                                             │  │
│  │  ┌──────────────────────────┐   ┌────────────────────────┐  │  │
│  │  │   VM Lab5.2-Server       │   │  VM Lab5.2-Attacker    │  │  │
│  │  │   Hostname: SERVIDOR     │   │  Hostname: ATACANTE    │  │  │
│  │  │                          │   │                        │  │  │
│  │  │  enp0s3: NAT (internet)  │   │  enp0s8: Red Interna   │  │  │
│  │  │  enp0s8: 192.168.30.2   ◄───►  192.168.30.3          │  │  │
│  │  │                          │   │                        │  │  │
│  │  │  Servicios:              │   │  Herramientas:         │  │  │
│  │  │  • SSH (puerto 22)       │   │  • Hydra (fuerza bruta)│  │  │
│  │  │  • Nginx (puerto 80)     │   │  • Nmap (escaneo)      │  │  │
│  │  │  • Fail2ban (HIPS)       │   │  • curl (web recon)    │  │  │
│  │  │  • UFW (firewall)        │   │                        │  │  │
│  │  └──────────┬───────────────┘   └────────────────────────┘  │  │
│  │             │                                                 │  │
│  │    [NAT VirtualBox]                                          │  │
│  └─────────────┼─────────────────────────────────────────────┘  │  │
│                │                                                   │
│          [Internet]                                               │
└───────────────────────────────────────────────────────────────────┘
```

### 3.4 Configuración de Adaptadores en VirtualBox

#### VM Lab5.2-Server

```
┌─────────────────────────────────────────────────────┐
│         VirtualBox — VM Lab5.2-Server               │
├─────────────────────────────────────────────────────┤
│  Adaptador 1 (enp0s3)                               │
│  ┌───────────────────────────────────────────────┐  │
│  │ Conectado a: NAT                              │  │
│  │ Port Forwarding:                              │  │
│  │   SSH  → 127.0.0.1:2222 → 10.0.2.15:22       │  │
│  │   HTTP → 127.0.0.1:8080 → 10.0.2.15:80       │  │
│  └───────────────────────────────────────────────┘  │
│  Adaptador 2 (enp0s8)                               │
│  ┌───────────────────────────────────────────────┐  │
│  │ Conectado a: Red Interna (Red_Lab5_2)         │  │
│  │ IP: 192.168.30.2/29 (estática)               │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

#### VM Lab5.2-Attacker

```
┌─────────────────────────────────────────────────────┐
│         VirtualBox — VM Lab5.2-Attacker             │
├─────────────────────────────────────────────────────┤
│  Adaptador 1 (enp0s3)                               │
│  ┌───────────────────────────────────────────────┐  │
│  │ Conectado a: Red Interna (Red_Lab5_2)         │  │
│  │ IP: 192.168.30.3/29 (estática)               │  │
│  │ Gateway: 192.168.30.2 (VM Server)             │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 4. Preparación del Entorno

### 4.1 VM Lab5.2-Server — Configuración de Red Estática

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

El archivo quedó configurado así:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.30.2/29
      nameservers:
        addresses:
          - 8.8.8.8
```
![alt text](<Screenshot 2026-05-21 191919.png>)
```bash
sudo netplan apply
```

### 4.2 Habilitar Reenvío de Paquetes (IP Forwarding)

Para que la VM Atacante pueda salir a internet a través del Server se necesita habilitar el reenvío de paquetes IPv4:

```bash
sudo nano /etc/sysctl.conf
```
![alt text](<Screenshot 2026-05-21 192030.png>)
- `/etc/sysctl.conf`: archivo de parámetros del kernel de Linux. Aquí se activan o desactivan funciones del sistema operativo de forma persistente (sobreviven reinicios).
- Se buscó y descomentó la línea `net.ipv4.ip_forward=1`, que le indica al kernel que puede reenviar paquetes entre interfaces de red (comportamiento de router).

Cargamos los parametros definidos en el archivo:

```bash
sudo sysctl -p
```

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

- `iptables`: herramienta para gestionar las reglas del firewall y NAT del kernel de Linux.
- `-t nat`: selecciona la tabla NAT (Network Address Translation).
- `-A POSTROUTING`: agrega una regla en la cadena POSTROUTING, que se ejecuta justo antes de que un paquete salga por una interfaz.
- `-o enp0s3`: la regla aplica solo a paquetes que salen por la interfaz `enp0s3` (la NAT de VirtualBox).
- `-j MASQUERADE`: enmascara la IP de origen del paquete con la IP de la interfaz de salida. Permite que el tráfico de la VM Atacante (`192.168.30.3`) salga a internet como si viniera del Server.

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

- `iptables-persistent`: paquete que guarda las reglas de iptables para que sobrevivan reinicios del sistema.
- `netfilter-persistent save`: guarda las reglas actuales en disco.

![alt text](<Screenshot 2026-05-21 192315.png>)

![alt text](<Screenshot 2026-05-21 192338.png>)

### 4.3 VM Lab5.2-Attacker — Configuración de Red Estática

El archivo netplan de la VM Atacante quedó así:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.30.3/29
      nameservers:
        addresses:
          - 192.168.30.2
      routes:
        - to: default
          via: 192.168.30.2
```
![alt text](<Screenshot 2026-05-21 191957.png>)
```bash
sudo netplan apply
```

---

## 5. Configuración del Servidor Objetivo (VM Lab5.2-Server)

### 5.1 Instalación de Nginx y SSH

```bash
sudo apt update
```

```bash
sudo apt install nginx openssh-server -y
```
- `nginx`: servidor web de alto rendimiento y bajo consumo de memoria, ampliamente usado en producción.
- `openssh-server`: servidor SSH que permite conexiones remotas seguras al sistema.
![alt text](<Screenshot 2026-05-21 192814.png>)
```bash
sudo systemctl enable --now nginx
sudo systemctl enable --now sshd
```

![alt text](<Screenshot 2026-05-21 192913.png>)
- `enable`: configura el servicio para que se inicie automáticamente cada vez que el sistema arranca, creando enlaces simbólicos en los directorios de systemd.
- `--now`: además de habilitarlo para el arranque, lo inicia inmediatamente. Es equivalente a ejecutar `enable` y `start` juntos.
- `nginx` y `sshd`: nombres de los servicios de Nginx y SSH respectivamente.

Creamos un sitio web de prueba

```bash
sudo bash -c 'echo "<h1>Servidor de Prueba - Lab 5.2</h1>" > /var/www/html/index.html'
sudo bash -c 'echo "<p>Objetivo de simulación de intrusiones</p>" >> /var/www/html/index.html'
```
![alt text](<Screenshot 2026-05-21 192951.png>)
- `bash -c`: ejecuta el texto entre comillas simples en un nuevo subshell con privilegios de root. Necesario para que la redirección `>` también se ejecute como root.
- `echo`: imprime el texto especificado en la salida estándar.
- `>`: redirige la salida al archivo, creándolo o sobreescribiéndolo si existe.
- `>>`: redirige la salida al archivo agregando al final sin sobreescribir.
- `/var/www/html/index.html`: ruta del archivo HTML raíz de Nginx. Es la página que se muestra al acceder a `http://192.168.30.2/`.

Verifiamos el estado de los servicios

```bash
sudo systemctl status nginx
sudo systemctl status sshd
```
![alt text](<Screenshot 2026-05-21 192958.png>)
![alt text](<Screenshot 2026-05-21 193016.png>)
```bash
sudo ss -tulnp | grep -E "nginx|sshd"
```
![alt text](<Screenshot 2026-05-21 193040.png>)
- `ss`: herramienta moderna para inspeccionar sockets de red (reemplaza al antiguo `netstat`).
- `-t`: muestra sockets TCP.
- `-u`: muestra sockets UDP.
- `-l`: muestra solo los sockets en estado LISTEN (esperando conexiones entrantes).
- `-n`: muestra números de puerto en vez de nombres de servicio (más rápido y evita resolución DNS).
- `-p`: muestra el proceso (PID y nombre) que tiene abierto cada socket.
- `|`: pipe, redirige la salida del comando anterior como entrada del siguiente.
- `grep -E "nginx|sshd"`: filtra las líneas que contengan "nginx" o "sshd". `-E` habilita expresiones regulares extendidas que permiten el operador `|` (OR).



### 5.2 Instalación y Configuración de Fail2ban

```bash
sudo apt install fail2ban -y
```
![alt text](<Screenshot 2026-05-21 194840.png>)
- `fail2ban`: sistema de prevención de intrusiones basado en host (HIPS). Monitorea archivos de log en tiempo real y bloquea automáticamente IPs que muestran comportamiento malicioso (múltiples intentos fallidos de autenticación).

```bash
sudo systemctl enable --now fail2ban
```
![alt text](<Screenshot 2026-05-21 195142.png>)
- Habilitamos e iniciamos el servicio Fail2ban de inmediato, igual que se hizo con nginx y sshd.

```bash
sudo nano /etc/fail2ban/jail.local
```

- `/etc/fail2ban/jail.local`: archivo de configuración local de Fail2ban. Se usa este archivo en lugar de editar `jail.conf` (el archivo por defecto) porque `jail.conf` puede ser sobreescrito al actualizar el paquete. El archivo `.local` siempre tiene prioridad y no es tocado por las actualizaciones.

Contenido configurado:

```ini
[DEFAULT]
bantime = 600
findtime = 600
maxretry = 3
backend = systemd

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 3
```
![alt text](<Screenshot 2026-05-21 195226.png>)
- `[DEFAULT]`: sección con valores por defecto que aplican a todos los jails si no se especifica lo contrario.
- `bantime = 600`: duración del bloqueo de la IP en segundos (10 minutos). Pasado ese tiempo, la IP es desbaneada automáticamente.
- `findtime = 600`: ventana de tiempo en segundos dentro de la cual se cuentan los intentos fallidos. Si en 10 minutos se superan los `maxretry`, se aplica el baneo.
- `maxretry = 3`: número máximo de intentos fallidos permitidos dentro de la ventana `findtime` antes de banear la IP.
- `backend = systemd`: le indica a Fail2ban que lea los logs del sistema desde el journal de systemd (en vez de archivos de texto plano), que es el sistema de logging de Ubuntu 24.04.
- `[sshd]`: jail (celda) para proteger el servicio SSH.
  - `enabled = true`: activa este jail.
  - `port = ssh`: monitorea el puerto SSH (22 por defecto).
  - `filter = sshd`: usa el filtro predefinido `/etc/fail2ban/filter.d/sshd.conf` que sabe cómo detectar intentos fallidos en los logs de SSH.
  - `logpath = /var/log/auth.log`: archivo de log donde SSH registra los intentos de autenticación.
- `[nginx-http-auth]`: jail para proteger la autenticación HTTP de Nginx.
  - `filter = nginx-http-auth`: filtro que detecta intentos fallidos de autenticación HTTP básica en Nginx.
  - `logpath = /var/log/nginx/error.log`: archivo de log de errores de Nginx.

Reiniciamos fail2ban:

```bash
sudo systemctl restart fail2ban
```
Verificamos el servicio:

```bash
sudo fail2ban-client status
```

- `fail2ban-client`: herramienta de línea de comandos para comunicarse con el daemon de Fail2ban (fail2ban-server) a través de un socket Unix.
- `status`: muestra el estado general de Fail2ban: número de jails activos y sus nombres.


```bash
sudo fail2ban-client status sshd
```

- `status sshd`: muestra el estado detallado del jail `sshd`: intentos fallidos actuales y totales, IPs actualmente baneadas y lista completa de IPs baneadas.
![alt text](<Screenshot 2026-05-21 195322.png>)
---

## 6. Simulación de Ataques (VM Lab5.2-Attacker)

### 6.1 Ataque de Fuerza Bruta a SSH con Hydra

```bash
sudo apt install hydra -y
```
![alt text](<Screenshot 2026-05-21 195407.png>)
- `hydra`: herramienta de fuerza bruta en red (también llamada THC-Hydra). Soporta decenas de protocolos (SSH, FTP, HTTP, RDP, etc.) y puede probar miles de combinaciones usuario/contraseña automáticamente en paralelo.

```bash
nano passwords.txt
```

Se creó un diccionario de contraseñas comunes:
```
123456
password
admin
ubuntu
root
12345678
qwerty
letmein
welcome
princess
```
![alt text](<Screenshot 2026-05-21 195433.png>)
- Este tipo de archivo se denomina **wordlist** o diccionario. En ataques reales se usan diccionarios de millones de entradas (como `rockyou.txt`). Aquí se usó uno pequeño con 10 contraseñas comunes para la simulación.

```bash
hydra -l jp -P passwords.txt ssh://192.168.30.2
```
![alt text](<Screenshot 2026-05-21 195507.png>)
- `hydra`: lanza la herramienta de fuerza bruta.
- `-l jp`: especifica un único usuario a probar (en minúscula `-l` = un solo usuario). Se usó `jp` que es el nombre de usuario del servidor.
- `-P passwords.txt`: especifica el archivo de contraseñas a probar (en mayúscula `-P` = archivo de contraseñas). Hydra probará cada línea del archivo como contraseña.
- `ssh://192.168.30.2`: protocolo y dirección del objetivo. Hydra intentará autenticarse por SSH contra la IP `192.168.30.2` en el puerto 22 por defecto.


Hydra probó las 10 contraseñas sin encontrar ninguna válida, pero generó múltiples intentos fallidos que Fail2ban detectó.


![alt text](<Screenshot 2026-05-21 200244.png>)
### 6.2 Escaneo de Puertos con Nmap

```bash
sudo apt install nmap -y
```
![alt text](<Screenshot 2026-05-21 200346.png>)
- `nmap`: Network Mapper, la herramienta de escaneo de redes más utilizada en el mundo. Permite descubrir hosts, servicios, versiones de software y sistemas operativos en una red.

```bash
nmap -sV 192.168.30.2
```
![alt text](<Screenshot 2026-05-21 200422.png>)
- `nmap`: lanza el escáner de red.
- `-sV`: activa la detección de versiones de servicios. Nmap intenta determinar qué software y qué versión está corriendo en cada puerto abierto, enviando sondas específicas y analizando las respuestas (banner grabbing).
- `192.168.30.2`: dirección IP del objetivo a escanear.
Nmap identificó los puertos abiertos, los servicios que corren en ellos y sus versiones exactas, información que un atacante real usaría para buscar vulnerabilidades conocidas (CVEs).

### 6.3 Reconocimiento Web con curl

```bash
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}" http://192.168.30.2/admin$i
  echo " -> /admin$i"
done
```
![alt text](<Screenshot 2026-05-21 200517.png>)
- `for i in $(seq 1 50)`: bucle en bash que itera con la variable `i` tomando valores del 1 al 50. `seq 1 50` genera la secuencia de números; `$(...)` ejecuta el comando y usa su salida.
- `curl`: herramienta de transferencia de datos por URL. Soporta HTTP, HTTPS, FTP y muchos otros protocolos.
- `-s`: modo silencioso (silent). Suprime la barra de progreso y mensajes de error para que la salida sea limpia.
- `-o /dev/null`: descarta el cuerpo de la respuesta HTTP enviándolo al dispositivo nulo (`/dev/null`). Solo nos interesa el código de estado, no el contenido HTML.
- `-w "%{http_code}"`: imprime el código de estado HTTP de la respuesta (ej: `404`, `200`, `403`). La opción `-w` acepta variables de formato de curl.
- `http://192.168.30.2/admin$i`: URL del objetivo. La variable `$i` se reemplaza por el número actual del bucle, generando `/admin1`, `/admin2`, ..., `/admin50`.
- `echo " -> /admin$i"`: imprime la ruta que se está probando para hacer la salida legible.

---

## 7. Análisis de Logs (VM Lab5.2-Server)

### 7.1 Análisis de auth.log — Ataques SSH

```bash
sudo grep "Failed password" /var/log/auth.log
```

- `grep`: herramienta de búsqueda de patrones en texto. Busca líneas que contengan el patrón especificado.
- `"Failed password"`: patrón a buscar. El servidor SSH registra esta cadena exacta en cada intento fallido de autenticación.
- `/var/log/auth.log`: archivo de log de autenticación del sistema. Registra todos los intentos de login (exitosos y fallidos) para SSH, sudo, PAM y otros mecanismos de autenticación.


```bash
sudo grep "Failed password" /var/log/auth.log | wc -l
```

- `wc`: herramienta para contar palabras, líneas o caracteres (Word Count).
- `-l`: cuenta solo el número de líneas de la entrada. Como cada línea de grep corresponde a un intento fallido, el resultado es el total de intentos.


```bash
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

- `awk`: lenguaje de procesamiento de texto orientado a columnas. Procesa la entrada línea por línea.
- `'{print $(NF-3)}'`: imprime el campo en la posición `NF-3`, donde `NF` es el número total de campos de la línea. En una línea de `auth.log` como `... Failed password for jp from 192.168.30.3 port 42064 ssh2`, `NF-3` apunta a la columna que contiene la dirección IP.
- `sort`: ordena las líneas alfabéticamente. Necesario antes de `uniq` para que las líneas iguales queden juntas.
- `uniq -c`: elimina líneas duplicadas consecutivas y antepone el número de repeticiones de cada una. Permite contar cuántas veces aparece cada IP.
- `sort -nr`: reordena el resultado de forma numérica (`-n`) y en orden descendente (`-r`), poniendo primero las IPs con más intentos.
![alt text](<Screenshot 2026-05-21 195558.png>)

### 7.2 Verificación del Baneo por Fail2ban

```bash
sudo fail2ban-client status sshd
```
![alt text](<Screenshot 2026-05-21 195610.png>)

Fail2ban detectó los 10 intentos desde `192.168.30.3` y la baneó automáticamente.

```bash
sudo iptables -L -n | grep fail2ban
```
![alt text](<Screenshot 2026-05-21 200021.png>)
![alt text](<Screenshot 2026-05-21 200151.png>)
- `-L`: lista todas las reglas de las cadenas de iptables.
- `-n`: muestra direcciones IP y puertos en formato numérico (sin resolver nombres DNS, más rápido).
- `grep fail2ban`: filtra solo las líneas relacionadas con cadenas creadas por Fail2ban.

En Ubuntu 24.04, Fail2ban usa nftables internamente en vez de iptables, por lo que este comando no muestra las reglas de baneo. El baneo funciona correctamente y se verifica con `fail2ban-client status sshd`.

### 7.3 Análisis de access.log — Reconocimiento Web

```bash
sudo grep " 404 " /var/log/nginx/access.log | head -20
```
![alt text](<Screenshot 2026-05-21 203302.png>)
- `/var/log/nginx/access.log`: archivo donde Nginx registra cada petición HTTP recibida, incluyendo IP del cliente, fecha, método HTTP, ruta solicitada, código de respuesta y User-Agent del navegador.
- `grep " 404 "`: busca líneas con código de respuesta 404 (Not Found). Los espacios alrededor del número evitan coincidencias falsas con otros números.
- `head -20`: muestra solo las primeras 20 líneas del resultado para no saturar la pantalla.


El log delató el uso de Nmap por su User-Agent `Nmap Scripting Engine` y el reconocimiento web por `curl/8.5.0`.

```bash
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr
```

- `awk '{print $1}'`: imprime solo el primer campo de cada línea, que en el formato de log de Nginx es la dirección IP del cliente.
- El resto del pipeline (`sort | uniq -c | sort -nr`) cuenta y ordena igual que antes.


```bash
sudo awk '{print $1, $4, $6, $7, $9}' /var/log/nginx/access.log | grep " 404 " | head -30
```
![alt text](<Screenshot 2026-05-21 203357.png>)
- `awk '{print $1, $4, $6, $7, $9}'`: imprime los campos 1 (IP), 4 (fecha), 6 (método HTTP), 7 (ruta) y 9 (código de respuesta) para tener una vista resumida y legible de cada petición.

---

## 8. Ciclo de Respuesta a Incidentes

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  1. IDENTI- │────►│  2. CONTE-  │────►│  3. ERRADI- │────►│  4. RECU-   │────►│  5. DOCU-   │
│    FICAR    │     │    NER      │     │    CAR      │     │    PERAR    │     │    MENTAR   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### 8.1 Identificar

**¿Qué servicio está siendo atacado?**
Dos servicios fueron atacados simultáneamente:
- **SSH (puerto 22)**: ataque de fuerza bruta con Hydra probando 10 contraseñas.
- **HTTP/Nginx (puerto 80)**: escaneo de puertos con Nmap y reconocimiento web con 54 peticiones a rutas inexistentes.

**¿Desde qué IP(s) proviene el ataque?**
- **IP atacante principal:** `192.168.30.3` (VM Atacante).
- La IP `10.0.2.2` corresponde a accesos legítimos del anfitrión vía NAT.

**¿Es un ataque de fuerza bruta, escaneo o ambos?**
**Ambos tipos combinados:** fuerza bruta SSH + escaneo de puertos + reconocimiento web automatizado.

```bash
sudo lastb
```
![alt text](<Screenshot 2026-05-21 203525.png>)
- `lastb`: muestra el historial de intentos de login **fallidos** del sistema, leyendo el archivo binario `/var/run/btmp`. Muestra usuario, terminal, IP de origen, fecha y hora de cada intento fallido.


```bash
sudo grep "Failed password" /var/log/auth.log | tail -20
```
![alt text](<Screenshot 2026-05-21 203608.png>)
- `tail -20`: muestra las últimas 20 líneas del resultado. Útil para ver los intentos más recientes primero.

```bash
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -10
```
![alt text](<Screenshot 2026-05-21 203630.png>)
- `head -10`: limita la salida a las 10 primeras líneas, mostrando solo las IPs con más peticiones 404.

### 8.2 Contener

Fail2ban aplicó contención automática. Se añadió bloqueo manual con UFW:

```bash
sudo ufw deny from 192.168.30.3
```
![alt text](<Screenshot 2026-05-21 203855.png>)
- `ufw`: Uncomplicated Firewall, interfaz simplificada para gestionar el firewall de Ubuntu (que internamente usa iptables/nftables).
- `deny`: crea una regla que rechaza (deniega) todo el tráfico.
- `from 192.168.30.3`: aplica la regla solo al tráfico originado en esa IP específica. Bloquea todos los puertos y protocolos provenientes de esa dirección.

Resultado: `Rules updated` — la regla fue agregada exitosamente.

### 8.3 Erradicar

```bash
sudo grep "Accepted" /var/log/auth.log
```
![alt text](<Screenshot 2026-05-21 203855-1.png>)
- `"Accepted"`: patrón que SSH registra cuando una autenticación es **exitosa**. Si aparece la IP atacante (`192.168.30.3`) en estos resultados, significaría que el atacante logró entrar y habría que tomar medidas urgentes (cambiar contraseñas, revocar claves, etc.).


Solo aparece `10.0.2.2` (acceso legítimo del administrador vía NAT). **Ningún acceso exitoso desde `192.168.30.3`.**

Si se hubiera detectado acceso exitoso del atacante, el siguiente paso sería cambiar credenciales inmediatamente:

```bash
sudo passwd usuario
```

- `passwd`: herramienta para cambiar contraseñas de usuarios del sistema. Solicita la nueva contraseña dos veces para confirmarla.

### 8.4 Recuperar

```bash
sudo systemctl status sshd
sudo systemctl status nginx
sudo systemctl status fail2ban
```
![alt text](<Screenshot 2026-05-21 203928.png>)
![alt text](<Screenshot 2026-05-21 203944.png>)
![alt text](<Screenshot 2026-05-21 204007.png>)
verificamos que cada servicio reporta `active (running)`, confirmando que el ataque no afectó la disponibilidad de los servicios legítimos.

```bash
sudo fail2ban-client set sshd unbanip 192.168.30.3
```
![alt text](<Screenshot 2026-05-21 200258.png>)
- `fail2ban-client set sshd`: envía un comando al jail `sshd` de Fail2ban.
- `unbanip 192.168.30.3`: desbanea manualmente la IP especificada antes de que expire el `bantime`. Útil en el laboratorio para continuar haciendo pruebas, o en producción cuando se confirma que el bloqueo fue un falso positivo.

### 8.5 Documentar

Se creó el archivo de registro del incidente:

```bash
nano incidente_001.md
```
---

## 9. Documentación del Incidente

```
## Incidente #001 - Lab 5.2

- **Fecha:** 22/05/2026
- **Hora de detección:** 23:54 UTC
- **IP Atacante:** 192.168.30.3 (VM Lab5.2-Attacker)
- **Servicios Afectados:** SSH (puerto 22) y HTTP Nginx (puerto 80)
- **Tipo de Ataque:** Fuerza bruta SSH + escaneo de puertos + reconocimiento web
- **Herramientas del atacante:** Hydra, Nmap, curl
- **Intentos Fallidos SSH:** 13 (10 detectados por Fail2ban)
- **Peticiones 404 HTTP:** 54
- **Accesos Exitosos del Atacante:** Ninguno detectado
- **Acciones Tomadas:**
  - Verificación de logs en /var/log/auth.log y /var/log/nginx/access.log
  - Confirmación de bloqueo automático por Fail2ban (192.168.30.3 baneada)
  - Bloqueo manual adicional con UFW (sudo ufw deny from 192.168.30.3)
  - Verificación de que no hubo accesos exitosos desde la IP atacante
  - Verificación de estado de servicios (sshd, nginx, fail2ban)
- **Estado:** Resuelto
```
![alt text](<Screenshot 2026-05-21 204107.png>)
---

## 10. Flujo del Ataque y la Defensa

```
VM Atacante (192.168.30.3)          VM Server (192.168.30.2)
         │                                    │
         │  1. nmap -sV 192.168.30.2          │
         │───────────────────────────────────►│
         │  ◄── Puerto 22 (SSH 9.6p1)         │
         │  ◄── Puerto 80 (Nginx 1.24.0)      │
         │                                    │
         │  2. hydra -l jp -P passwords.txt   │
         │───────────────────────────────────►│  ◄── auth.log registra intentos
         │                                    │  ◄── Fail2ban detecta 3+ intentos
         │                                    │  ◄── Fail2ban banea 192.168.30.3
         │                                    │
         │  3. curl http://.../admin{1..50}   │
         │───────────────────────────────────►│  ◄── access.log registra 54 x 404
         │  ◄── 404 Not Found (x50)           │
         │                                    │
         │  4. ssh jp@192.168.30.2            │
         │─────────── BLOQUEADO ─────────────►│  ◄── UFW/Fail2ban rechaza
         │  ◄── Connection refused            │
         │                                    │
                                     [Analista]
                                         │
                                    Identificar
                                    Contener (UFW)
                                    Erradicar (verificar accesos)
                                    Recuperar (servicios OK)
                                    Documentar (incidente_001.md)
```

---


## 11. Conclusiones

- **Fail2ban es una herramienta efectiva** de prevención de intrusiones basada en host. Con una configuración mínima (3 intentos fallidos en 10 minutos), bloqueó automáticamente la IP atacante antes de que pudiera comprometer el sistema.

- **Los logs son la primera línea de evidencia** en la respuesta a incidentes. El archivo `auth.log` registró cada intento fallido de SSH con timestamp, usuario e IP de origen; mientras que `access.log` de Nginx registró cada petición web incluyendo el User-Agent del cliente, lo que permitió identificar incluso el uso de Nmap por su firma característica en el campo User-Agent.

- **La combinación de ataques es más peligrosa que uno solo.** El atacante combinó escaneo de puertos (obtener información del objetivo), fuerza bruta SSH (intentar acceso al sistema) y reconocimiento web (buscar rutas o recursos vulnerables). Esta secuencia es típica de un pentest o ataque real organizado.

- **Nmap expone información sensible:** el escaneo con `-sV` reveló las versiones exactas de OpenSSH 9.6p1 y Nginx 1.24.0, así como el sistema operativo Ubuntu. En un entorno real, esto permitiría al atacante buscar CVEs (vulnerabilidades conocidas) específicos para esas versiones.

- **El ciclo de respuesta a incidentes estructura la reacción:** seguir los pasos de Identificar → Contener → Erradicar → Recuperar → Documentar evita que la respuesta sea caótica y garantiza que no se pase por alto ningún paso crítico, como verificar si el atacante ya había logrado acceso antes de ser baneado.

- **UFW complementa a Fail2ban:** mientras Fail2ban aplica bloqueos automáticos temporales basados en patrones de logs, UFW permite bloqueos manuales permanentes cuando el analista confirma que una IP es maliciosa. Usar ambas herramientas en conjunto fortalece la postura de seguridad del servidor.

- **La práctica en entorno virtualizado** con dos VMs en la misma máquina física replicó fielmente un escenario real de ataque/defensa, permitiendo observar en tiempo real cómo los ataques se registran en los logs y cómo los sistemas de defensa responden automáticamente.

---
