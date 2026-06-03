# Informe de Laboratorio 6.1
## Automatización de Tareas Administrativas con Bash

---

**Universidad Mayor Real y Pontificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnología**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiante:**                      **Carrera:**

- Juan Pablo Taboada Camacho          Ing. Sistemas

---

## 1. Introducción

El presente laboratorio tiene como objetivo desarrollar habilidades en la automatización de tareas administrativas mediante scripts Bash en un entorno virtualizado con VirtualBox. Se configuraron dos máquinas virtuales con Ubuntu Server 24.04 LTS: una VM de Administración (`Lab6.1-Admin`) que actúa como servidor principal de scripts y una VM Objetivo (`Lab6.1-Target`) sobre la que se aplican las tareas automatizadas de forma remota. A lo largo del laboratorio se implementaron scripts para gestión de usuarios, monitoreo de servicios, análisis de logs, despliegue de software y un menú interactivo de administración.

---

## 2. Objetivos del Laboratorio

- **Dominar los fundamentos de Bash Scripting** aplicando variables, argumentos posicionales, condicionales, bucles y pipes en scripts funcionales.

- **Automatizar tareas repetitivas** de administración como gestión masiva de usuarios, monitoreo de servicios y análisis de logs del sistema.

- **Desarrollar scripts reutilizables** para el despliegue desatendido de software y la verificación del estado del sistema.

- **Configurar acceso SSH sin contraseña** entre servidores para habilitar la ejecución remota de scripts de forma segura y automatizada.

- **Construir una interfaz CLI** mediante un menú interactivo que facilite la ejecución controlada de tareas administrativas.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
Internet (NAT)
      |
   [Lab6.1-Admin - Ubuntu Server 24.04]
   enp0s3 (NAT) | enp0s8 (Red Interna)
      |
   [Red Interna - VirtualBox]
      |
   [Lab6.1-Target - Ubuntu Server 24.04]
   enp0s8 (Red Interna)
```

### 3.2 Esquema de la Infraestructura

| Máquina Virtual | Hostname | Rol | Interfaces | IP |
|-----------------|----------|-----|------------|-----|
| `Lab6.1-Admin` | `admin` | Servidor de Administración (Scripts, Menú, Cron) | NAT + Red Interna | `192.168.40.2/29` |
| `Lab6.1-Target` | `target` | Servidor Objetivo (Pruebas de usuarios, servicios) | Red Interna | `192.168.40.3/29` |

### 3.3 Tabla de Direccionamiento

| Red | Segmento | Máscara | Gateway | Hosts |
|-----|----------|---------|---------|-------|
| `192.168.40.0/29` | Red Interna | `255.255.255.248` | `192.168.40.2` (Admin) | Admin (.2), Target (.3) |

---

## 4. Preparación del Entorno

### 4.1 Configuración de Red en VirtualBox

Se creó una red interna llamada `Red_Lab6_1` en VirtualBox. Se configuraron los adaptadores de cada VM de la siguiente manera:

- **VM Lab6.1-Admin:** Adaptador 1 en modo NAT (acceso a internet) y Adaptador 2 en Red Interna.
- **VM Lab6.1-Target:** Adaptador 1 en Red Interna únicamente.

Se configuró el reenvío de puertos en el Adaptador NAT de la VM Admin para permitir acceso SSH desde la máquina anfitriona:

| Nombre | Protocolo | Puerto Host | Puerto Invitado | Propósito |
|--------|-----------|-------------|-----------------|-----------|
| SSH | TCP | 2222 | 22 | Acceso Remoto |

### 4.2 Instalación de Paquetes

**En VM Admin:**

```bash
sudo apt update
sudo apt install -y openssh-server openssh-client iptables iptables-persistent curl netcat-openbsd nginx
```

- `openssh-server`: levanta el servicio SSH para recibir conexiones entrantes.
- `openssh-client`: permite usar los comandos `ssh`, `ssh-keygen` y `ssh-copy-id`.
- `iptables` + `iptables-persistent`: permiten definir y persistir reglas de NAT para que el Target salga a internet a través del Admin.
- `curl`: permite hacer peticiones HTTP para verificar que Nginx responde correctamente.
- `netcat-openbsd`: permite verificar si un puerto está abierto en un servidor remoto.
- `nginx`: servidor web necesario para los ejercicios de análisis de logs y health check.

**En VM Target:**

```bash
sudo apt update
sudo apt install -y openssh-server
```

- `openssh-server`: permite recibir conexiones SSH desde la VM Admin para ejecutar scripts de forma remota.

### 4.3 Configuración de IP Estática — VM Admin

Se editó el archivo de configuración de Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

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
        - 192.168.40.2/29
      nameservers:
        addresses:
          - 8.8.8.8
```
![alt text](<Screenshot 2026-06-03 001328.png>)

- `enp0s3`: interfaz NAT, obtiene IP automáticamente por DHCP para acceder a internet.
- `enp0s8`: interfaz de red interna, se le asigna la IP estática `192.168.40.2` con máscara `/29`.
- `optional: true`: indica que el arranque del sistema no debe esperar a que esta interfaz esté activa.
- `nameservers`: define `8.8.8.8` (Google DNS) para la resolución de nombres.

```bash
sudo netplan apply
```

- `netplan apply`: aplica la configuración de red sin necesidad de reiniciar la máquina. Lee el archivo YAML y reconfigura las interfaces en caliente.

Verificación de que la IP quedó asignada correctamente:

```bash
ip a show enp0s8
```

- `ip a`: muestra la información de todas las interfaces de red del sistema. Es equivalente al antiguo `ifconfig`.
- `show enp0s8`: filtra la salida para mostrar únicamente la interfaz `enp0s8`. Se debe ver la IP `192.168.40.2/29` asignada.

### 4.4 Configuración de IP Estática — VM Target

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.40.3/29
      nameservers:
        addresses:
          - 192.168.40.2
      routes:
        - to: default
          via: 192.168.40.2
```
![alt text](<Screenshot 2026-06-03 001809.png>)

- `addresses`: asigna la IP `192.168.40.3` al Target.
- `nameservers`: usa al Admin (`192.168.40.2`) como servidor DNS, quien a su vez reenvía las consultas a `8.8.8.8`.
- `routes`: define la ruta por defecto hacia el Admin (`192.168.40.2`), quien actúa como gateway para que el Target pueda salir a internet.

```bash
sudo netplan apply
```

- Aplica la configuración de red en la VM Target sin reiniciar.

Verificación de la IP asignada:

```bash
ip a show enp0s8
```

- Muestra la configuración de la interfaz `enp0s8` del Target. Se debe ver `192.168.40.3/29` asignada.

### 4.5 Habilitar Reenvío de Paquetes — VM Admin

```bash
sudo nano /etc/sysctl.conf
# Descomentar: net.ipv4.ip_forward=1
sudo sysctl -p
```

- `net.ipv4.ip_forward=1`: habilita en el kernel Linux la capacidad de reenviar paquetes de una interfaz a otra, convirtiendo al Admin en un router.
- `sysctl -p`: recarga la configuración del kernel desde `/etc/sysctl.conf` sin reiniciar.

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

![alt text](<Screenshot 2026-06-03 001304.png>)
![alt text](<Screenshot 2026-06-03 001455.png>)
![alt text](<Screenshot 2026-06-03 001744.png>)

- `-t nat`: selecciona la tabla NAT de iptables.
- `-A POSTROUTING`: agrega una regla en la cadena POSTROUTING, que se ejecuta justo antes de que un paquete salga por la interfaz de red.
- `-o enp0s3`: aplica la regla solo a los paquetes que salen por la interfaz NAT.
- `-j MASQUERADE`: reemplaza la IP de origen del paquete con la IP del Admin, permitiendo que el Target navegue usando la IP del Admin.
- `iptables-persistent` + `netfilter-persistent save`: guardan las reglas de iptables en disco para que persistan después de un reinicio.

Verificación de conectividad desde VM Target:

```bash
ping -c 4 8.8.8.8
```

- `ping`: envía paquetes ICMP echo-request para verificar conectividad de red.
- `-c 4`: limita el número de paquetes enviados a 4. Sin este flag, ping corre indefinidamente.
- `8.8.8.8`: servidor DNS público de Google, usado como destino para comprobar que el Target tiene acceso a internet a través del Admin.

### 4.6 Acceso SSH con Clave — VM Admin hacia VM Target

```bash
ssh-keygen -t ed25519 -C "admin@lab61" -f ~/.ssh/id_lab61
```

- `ssh-keygen`: genera un par de claves criptográficas (pública y privada).
- `-t ed25519`: usa el algoritmo Ed25519, más seguro y eficiente que RSA.
- `-C "admin@lab61"`: agrega un comentario identificador a la clave.
- `-f ~/.ssh/id_lab61`: define el nombre y ruta del archivo donde se guarda el par de claves.

```bash
ssh-copy-id -i ~/.ssh/id_lab61.pub jp@192.168.40.3
```

- `ssh-copy-id`: copia la clave pública al archivo `~/.ssh/authorized_keys` del servidor remoto.
- `-i ~/.ssh/id_lab61.pub`: especifica qué clave pública copiar.
- La clave queda almacenada permanentemente en el disco del Target, por lo que persiste entre reinicios.

```bash
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3
```
![alt text](<Screenshot 2026-06-03 002411.png>)
- `-i ~/.ssh/id_lab61`: indica al cliente SSH que use la clave privada correspondiente para autenticarse sin contraseña.
- Al conectarse exitosamente sin pedir contraseña, se confirma que la clave fue copiada correctamente.

Para salir de la sesión SSH del Target:

```bash
exit
```

- `exit`: cierra la sesión SSH activa y regresa a la terminal de la VM Admin.

### 4.7 Configurar sudo sin Contraseña — VM Target

Para permitir la ejecución remota de comandos con `sudo` sin necesidad de una terminal interactiva:

```bash
sudo visudo
# Agregar:
jp      ALL=(ALL) NOPASSWD: ALL
```

- `visudo`: editor seguro para el archivo `/etc/sudoers`, valida la sintaxis antes de guardar para evitar bloqueos del sistema.
- `NOPASSWD: ALL`: permite al usuario `jp` ejecutar cualquier comando con `sudo` sin que se solicite contraseña, necesario para automatización remota via SSH.

Verificación de que sudo funciona sin contraseña desde Admin:

```bash
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3 "sudo whoami"
```

- Ejecuta el comando `whoami` con `sudo` en el Target de forma remota. Si responde `root` sin pedir contraseña, la configuración es correcta.
- `whoami`: imprime el nombre del usuario efectivo actual. Al usar `sudo`, debe devolver `root`.

### 4.8 Preparar Directorios de Trabajo — VM Admin

```bash
sudo mkdir -p /opt/admin_scripts
sudo mkdir -p /var/backups/data_center
sudo chmod 755 /opt/admin_scripts
```

- `mkdir -p`: crea el directorio y todos los directorios padre necesarios sin generar error si ya existen.
- `chmod 755`: asigna permisos de lectura, escritura y ejecución al propietario, y solo lectura y ejecución a grupo y otros usuarios.

---

## 5. Práctica Guiada — Ejercicios Individuales

### Ejercicio 1: Configuración de Red Estática

La configuración de red estática se realizó como parte de la preparación del entorno (secciones 4.3, 4.4 y 4.5). Se asignó la IP `192.168.40.2/29` al Admin y `192.168.40.3/29` al Target, habilitando el reenvío de paquetes y la regla NAT para conectividad a internet desde el Target.

---

### Ejercicio 2: Script de Bienvenida y Log

**Archivo:** `/opt/admin_scripts/01_intro.sh`

```bash
#!/bin/bash
# Script de bienvenida y registro de acceso

LOG_FILE="/tmp/admin_access.log"
NOMBRE=$1
ROL=$2

echo "========================================="
echo "¡Bienvenido, $NOMBRE! Su rol es $ROL."
echo "========================================="

echo "$(date '+%Y-%m-%d %H:%M:%S') - Usuario del sistema: $USER. Nombre: $NOMBRE, Rol: $ROL." >> $LOG_FILE

echo "Último registro añadido a $LOG_FILE:"
tail -n 1 $LOG_FILE
```
![alt text](<Screenshot 2026-06-03 002452.png>)
**Explicación del script:**

- `#!/bin/bash`: shebang, indica al sistema operativo que este script debe ejecutarse con el intérprete Bash.
- `$1` y `$2`: argumentos posicionales recibidos al invocar el script. `$1` corresponde al primer argumento (nombre) y `$2` al segundo (rol).
- `$USER`: variable de entorno del sistema que contiene el nombre del usuario que ejecuta el script.
- `$(date '+%Y-%m-%d %H:%M:%S')`: sustitución de comando, ejecuta `date` y embebe su salida en la cadena de texto con formato año-mes-día hora:minuto:segundo.
- `>>`: operador de redireccionamiento que añade (append) la salida al archivo sin sobrescribir el contenido existente.
- `tail -n 1`: muestra solo la última línea del archivo de log para confirmar el registro.

**Ejecución:**

```bash
sudo chmod +x /opt/admin_scripts/01_intro.sh
/opt/admin_scripts/01_intro.sh "Juan Perez" "Administrador"
/opt/admin_scripts/01_intro.sh "Ana Lopez" "Soporte"
cat /tmp/admin_access.log
```

![alt text](<Screenshot 2026-06-03 002542.png>)
- `chmod +x`: otorga permiso de ejecución al script. Sin este permiso, el sistema rechaza ejecutarlo directamente.

---

### Ejercicio 3: Script de Verificación de Archivos y Directorios

**Archivo:** `/opt/admin_scripts/02_check.sh`

```bash
#!/bin/bash
# Verificación rápida de archivos y directorios críticos

LOG_FILE="/tmp/admin_access.log"
DIR_WEB="/var/www/html"

if [ -f "$LOG_FILE" ]; then
    echo "[OK] El archivo de log $LOG_FILE existe."
else
    echo "[ALERTA] El archivo de log NO fue encontrado."
fi

if [ -d "$DIR_WEB" ]; then
    echo "[OK] El directorio web $DIR_WEB existe."
else
    echo "[ERROR] El directorio web $DIR_WEB no existe. Creándolo..."
    sudo mkdir -p "$DIR_WEB"
    echo "[OK] Directorio creado."
fi

USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//g')
if [ "$USAGE" -gt 85 ]; then
    echo "[CRITICO] Uso de disco: $USAGE%. Limpieza requerida."
else
    echo "[OK] Uso de disco: $USAGE%."
fi
```
![alt text](<Screenshot 2026-06-03 002657.png>)

**Explicación del script:**

- `[ -f "$LOG_FILE" ]`: operador de prueba que verifica si la ruta corresponde a un archivo regular existente.
- `[ -d "$DIR_WEB" ]`: operador de prueba que verifica si la ruta corresponde a un directorio existente.
- `df -h /`: muestra el uso del disco de la partición raíz en formato legible por humanos (human-readable).
- `awk 'NR==2 {print $5}'`: procesa la segunda línea (NR==2) de la salida de `df` y extrae la quinta columna, que contiene el porcentaje de uso.
- `sed 's/%//g'`: elimina el símbolo `%` del resultado para que pueda usarse como número entero en la comparación.
- `-gt 85`: operador de comparación numérica "mayor que" (greater than). Se usa `-gt` en lugar de `>` porque dentro de `[ ]` el símbolo `>` es un operador de redirección.
![alt text](<Screenshot 2026-06-03 002710.png>)

---

### Ejercicio 4: Procesamiento de Puertos con Pipes y Filtros

**Archivo:** `/opt/admin_scripts/03_pipes.sh`

```bash
#!/bin/bash
# Top 5 puertos TCP en escucha

echo "Top 5 puertos TCP más utilizados:"
sudo ss -tuln | grep 'tcp ' | awk '{print $5}' | cut -d':' -f2 | sort | uniq -c | sort -nr | head -n 5
```
![alt text](<Screenshot 2026-06-03 002818.png>)

**Explicación del pipeline:**

- `ss -tuln`: muestra los sockets activos. `-t` filtra TCP, `-u` UDP, `-l` solo los que están en escucha (listening), `-n` muestra números de puerto en lugar de nombres de servicio.
- `grep 'tcp '`: filtra solo las líneas que contienen conexiones TCP.
- `awk '{print $5}'`: extrae la quinta columna de cada línea, que contiene la dirección y puerto en formato `IP:puerto`.
- `cut -d':' -f2`: divide la cadena usando `:` como delimitador y toma el segundo campo, que es el número de puerto.
- `sort`: ordena alfabéticamente los puertos para que `uniq` pueda contar las repeticiones.
- `uniq -c`: cuenta las ocurrencias consecutivas de cada valor único, anteponiéndolas al resultado.
- `sort -nr`: reordena numéricamente (`-n`) de mayor a menor (`-r`) según el conteo.
- `head -n 5`: muestra solo las primeras 5 líneas, es decir, los 5 puertos más frecuentes.

![alt text](<Screenshot 2026-06-03 002743.png>)
---

### Ejercicio 5: Iteración y Análisis de Logs con Bucles

**Archivo:** `/opt/admin_scripts/04_summarize_logs.sh`

```bash
#!/bin/bash
# Análisis de logs de Nginx con bucles

LOG_DIR="/var/log/nginx/"
ACCESS_LOG="/var/log/nginx/access.log"
STATUS_COUNT=0

echo "--- 1. Conteo de archivos log (bucle FOR) ---"

for LOG_FILE in $LOG_DIR*.log; do
    if [ -f "$LOG_FILE" ]; then
        LINE_COUNT=$(wc -l < "$LOG_FILE")
        echo "  [FOR] $(basename $LOG_FILE): $LINE_COUNT líneas"
    fi
done

echo -e "\n--- 2. Análisis de peticiones 200 (bucle WHILE) ---"

if [ -f "$ACCESS_LOG" ]; then
    while read LINE; do
        if echo "$LINE" | grep -q " 200 "; then
            STATUS_COUNT=$((STATUS_COUNT + 1))
        fi
    done < "$ACCESS_LOG"
    echo "  [WHILE] Total peticiones HTTP 200: $STATUS_COUNT"
else
    echo "  [ERROR] $ACCESS_LOG no encontrado"
fi
```
![alt text](<Screenshot 2026-06-03 003002.png>)
**Explicación del script:**

- `for LOG_FILE in $LOG_DIR*.log`: itera sobre todos los archivos con extensión `.log` dentro del directorio de logs de Nginx. El `*` es un glob que expande todos los archivos que coincidan.
- `wc -l < "$LOG_FILE"`: cuenta las líneas del archivo. El `<` redirige el contenido del archivo como entrada estándar de `wc`, evitando que el nombre del archivo aparezca en la salida.
- `basename $LOG_FILE`: extrae solo el nombre del archivo sin la ruta completa, para mostrarlo de forma más limpia.
- `while read LINE`: lee el archivo de log línea por línea. Es más eficiente que cargar todo el archivo en memoria, especialmente útil para archivos grandes.
- `< "$ACCESS_LOG"`: redirige el archivo como entrada al bucle `while`, procesando cada línea secuencialmente.
- `grep -q " 200 "`: busca silenciosamente (`-q`) el código HTTP 200 en la línea. No imprime nada, solo devuelve 0 (encontrado) o 1 (no encontrado).
- `$((STATUS_COUNT + 1))`: aritmética entera en Bash usando la sintaxis `$(( ))`.
- `echo -e "\n"`: el flag `-e` habilita la interpretación de secuencias de escape como `\n` (nueva línea).
![alt text](<Screenshot 2026-06-03 003035.png>)
> **Nota:** Para este ejercicio fue necesario iniciar el servicio Nginx en la VM Admin, ya que el directorio `/var/log/nginx/` no existe hasta que el servicio se inicia al menos una vez:
>
> ```bash
> sudo systemctl enable --now nginx
> sudo systemctl status nginx
> ```
>
> - `systemctl enable --now nginx`: habilita el servicio para que inicie automáticamente con el sistema (`enable`) y lo inicia de inmediato (`--now`).
> - `systemctl status nginx`: muestra el estado actual del servicio. Se debe ver `active (running)` para confirmar que está corriendo.

---

### Ejercicio 6: Gestión Masiva de Usuarios desde CSV

**Archivo CSV:** `/opt/admin_scripts/usuarios.csv`

```bash
sudo bash -c 'echo "ana_sistemas,sistemas" > /opt/admin_scripts/usuarios.csv'
sudo bash -c 'echo "luis_soporte,soporte" >> /opt/admin_scripts/usuarios.csv'
sudo bash -c 'echo "eva_sistemas,sistemas" >> /opt/admin_scripts/usuarios.csv'
sudo bash -c 'echo "carlos_redes,redes" >> /opt/admin_scripts/usuarios.csv'
```
![alt text](<Screenshot 2026-06-03 003312.png>)

- `sudo bash -c '...'`: ejecuta el comando entre comillas como superusuario. Se necesita porque la redirección `>` y `>>` se ejecuta como `root` para escribir en `/opt/admin_scripts/`.

**Archivo:** `/opt/admin_scripts/05_user_manager.sh`

```bash
#!/bin/bash
# Gestión masiva de usuarios y grupos desde CSV

CSV_FILE="/opt/admin_scripts/usuarios.csv"

if [ ! -f "$CSV_FILE" ]; then
    echo "[ERROR] Archivo $CSV_FILE no encontrado."
    exit 1
fi

cat "$CSV_FILE" | while IFS=',' read -r USERNAME GROUPNAME; do
    GROUPNAME=$(echo "$GROUPNAME" | tr -d '[:space:]')
    USERNAME=$(echo "$USERNAME" | tr -d '[:space:]')

    if ! grep -q "^$GROUPNAME:" /etc/group; then
        sudo groupadd "$GROUPNAME"
        echo "[OK] Grupo '$GROUPNAME' creado."
    fi

    if ! id "$USERNAME" &>/dev/null; then
        sudo useradd -m -g "$GROUPNAME" -s /bin/bash "$USERNAME"
        echo "[OK] Usuario '$USERNAME' creado (grupo: $GROUPNAME)."
    else
        echo "[INFO] Usuario '$USERNAME' ya existe."
    fi
done
```
![alt text](<Screenshot 2026-06-03 003337.png>)

**Explicación del script:**

- `[ ! -f "$CSV_FILE" ]`: el `!` niega la condición. Verifica si el archivo NO existe y termina el script con `exit 1` si es así.
- `IFS=','`: define la coma como separador de campos (Internal Field Separator). Permite que `read` divida automáticamente cada línea del CSV en sus columnas.
- `read -r USERNAME GROUPNAME`: lee dos variables por línea. `-r` evita que las barras invertidas sean interpretadas como caracteres de escape.
- `tr -d '[:space:]'`: elimina todos los espacios en blanco de la variable, limpiando posibles espacios al inicio o final de cada campo del CSV.
- `grep -q "^$GROUPNAME:" /etc/group`: busca si el grupo ya existe en `/etc/group`. El `^` ancla la búsqueda al inicio de línea para evitar coincidencias parciales.
- `id "$USERNAME" &>/dev/null`: verifica si el usuario existe. `&>/dev/null` redirige tanto la salida estándar como los errores a `/dev/null` para que no se muestre nada en pantalla.
- `useradd -m -g "$GROUPNAME" -s /bin/bash`: crea el usuario con su directorio home (`-m`), asignado al grupo especificado (`-g`) y con shell Bash (`-s /bin/bash`).
![alt text](<Screenshot 2026-06-03 003416.png>)
Verificación de usuarios y grupos creados en Admin:

```bash
grep -E "sistemas|soporte|redes" /etc/group
id ana_sistemas
id luis_soporte
```
![alt text](<Screenshot 2026-06-03 003506.png>)
- `grep -E`: búsqueda con expresiones regulares extendidas. Filtra todas las líneas de `/etc/group` que contengan alguno de los grupos creados.
- `id ana_sistemas`: muestra el UID, GID y grupos del usuario `ana_sistemas`. Confirma que el usuario fue creado y asignado correctamente al grupo.

**Replicación en VM Target:**

```bash
scp -i ~/.ssh/id_lab61 /opt/admin_scripts/usuarios.csv /opt/admin_scripts/05_user_manager.sh jp@192.168.40.3:/tmp/
```
![alt text](<Screenshot 2026-06-03 003540.png>)
- `scp`: copia archivos de forma segura entre hosts usando el protocolo SSH. Los dos archivos se copian al directorio `/tmp/` del Target en una sola instrucción.
- La salida muestra el progreso de cada archivo con porcentaje, velocidad y tiempo estimado (ej. `100%  763  94.1KB/s  00:00`).

Como el script busca el CSV en `/opt/admin_scripts/` fue necesario crearlo también en el Target:

```bash
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3
sudo mkdir -p /opt/admin_scripts
sudo cp /tmp/usuarios.csv /opt/admin_scripts/
sudo bash /tmp/05_user_manager.sh
exit
```
![alt text](<Screenshot 2026-06-03 003613.png>)
![alt text](<Screenshot 2026-06-03 003742.png>)

- `mkdir -p /opt/admin_scripts`: crea el directorio en el Target igual que en el Admin para que el script encuentre el CSV en la ruta esperada.
- `cp /tmp/usuarios.csv /opt/admin_scripts/`: copia el CSV desde `/tmp/` (donde llegó por `scp`) a la ruta que el script espera.

- `scp`: copia archivos de forma segura entre hosts usando el protocolo SSH.
- Los dos archivos se copian al directorio `/tmp/` del Target en una sola instrucción.
- El segundo comando ejecuta el script remotamente en el Target vía SSH, sin necesidad de iniciar sesión interactiva.

---

### Ejercicio 7: Despliegue Desatendido de Nginx

**Archivo:** `/opt/admin_scripts/deploy_nginx.sh`

```bash
#!/bin/bash
# Despliegue desatendido de Nginx y configuración base

echo "[INFO] Iniciando despliegue de Nginx..."

sudo apt update
sudo apt install nginx -y

sudo systemctl enable --now nginx

if systemctl is-active --quiet nginx; then
    echo "[OK] Nginx instalado y activo."
else
    echo "[ERROR] Nginx no pudo iniciarse."
    exit 1
fi

sudo bash -c 'echo "<h1>Servidor desplegado automaticamente</h1>" > /var/www/html/index.html'
sudo bash -c 'echo "<p>Fecha de despliegue: '"$(date)"'</p>" >> /var/www/html/index.html'

echo "[OK] Despliegue completado."
```
![alt text](<Screenshot 2026-06-03 004205.png>)

**Explicación del script:**

- `apt install nginx -y`: instala Nginx sin requerir confirmación del usuario. El flag `-y` responde "sí" automáticamente a todas las preguntas del instalador, haciendo el proceso completamente desatendido.
- `systemctl enable --now nginx`: combina en un solo comando la habilitación del servicio al arranque (`enable`) y su inicio inmediato (`--now`), equivalente a `systemctl enable nginx && systemctl start nginx`.
- `systemctl is-active --quiet nginx`: verifica si el servicio está activo. `--quiet` suprime la salida de texto; el comando solo devuelve un código de retorno (0 si activo, distinto de 0 si no).
- La página HTML de prueba se genera dinámicamente con `$(date)`, que inserta la fecha y hora exacta del despliegue en el archivo `index.html`.

**Ejecución remota en VM Target:**

```bash
scp -i ~/.ssh/id_lab61 /opt/admin_scripts/deploy_nginx.sh jp@192.168.40.3:/tmp/
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3 "sudo bash /tmp/deploy_nginx.sh"
curl http://192.168.40.3
```
![alt text](<Screenshot 2026-06-03 004304.png>)
![alt text](<Screenshot 2026-06-03 004500.png>)

- `curl http://192.168.40.3`: realiza una petición HTTP GET al Target para verificar que Nginx está respondiendo y muestra el contenido HTML recibido en la terminal.

---

### Ejercicio 8: Limpieza Automatizada de Logs

**Archivo:** `/opt/admin_scripts/log_cleanup.sh`

```bash
#!/bin/bash
# Limpieza automatizada de logs antiguos del sistema

DIAS=30
LOG_DIRS="/var/log /var/log/nginx /var/log/apache2"
REPORTE="/tmp/cleanup_report.log"

echo "[INFO] Iniciando limpieza de logs mayores a $DIAS dias..." | sudo tee "$REPORTE"

for DIR in $LOG_DIRS; do
    if [ -d "$DIR" ]; then
        COUNT=$(find "$DIR" -type f -name "*.log*" -mtime +$DIAS | wc -l)
        if [ "$COUNT" -gt 0 ]; then
            find "$DIR" -type f -name "*.log*" -mtime +$DIAS -delete
            echo "[OK] $DIR: $COUNT logs eliminados." | sudo tee -a "$REPORTE"
        else
            echo "[INFO] $DIR: No hay logs antiguos para eliminar." | sudo tee -a "$REPORTE"
        fi
    fi
done

echo "[INFO] Limpieza finalizada. Reporte: $REPORTE" | sudo tee -a "$REPORTE"
```
![alt text](<Screenshot 2026-06-03 004543.png>)
**Explicación del script:**

- `find "$DIR" -type f -name "*.log*" -mtime +$DIAS`: busca archivos (`-type f`) cuyo nombre contenga `.log` y cuya fecha de modificación sea mayor a 30 días (`-mtime +30`). El `+` indica "más de N días".
- `| wc -l`: cuenta cuántos archivos encontró el comando `find` antes de eliminarlos, para reportarlo.
- `-delete`: flag de `find` que elimina directamente los archivos encontrados sin moverlos a la papelera.
- `tee "$REPORTE"`: escribe la salida tanto en la terminal como en el archivo de reporte simultáneamente. Es equivalente a imprimir y guardar al mismo tiempo.
- `tee -a`: el flag `-a` (append) hace que `tee` agregue al archivo en lugar de sobrescribirlo.

**Ejecución:**

```bash
sudo chmod +x /opt/admin_scripts/log_cleanup.sh
sudo /opt/admin_scripts/log_cleanup.sh
```

- `sudo /opt/admin_scripts/log_cleanup.sh`: se ejecuta con `sudo` porque el script necesita permisos de superusuario para eliminar archivos en `/var/log/`.

Verificación del reporte generado:

```bash
cat /tmp/cleanup_report.log
```
![alt text](<Screenshot 2026-06-03 004615.png>)
- Muestra el contenido del reporte de limpieza, listando qué directorios fueron procesados y cuántos logs fueron eliminados en cada uno.

---

### Ejercicio 9: Rollback de Usuarios Creados

**Archivo:** `/opt/admin_scripts/user_cleanup.sh`

```bash
#!/bin/bash
# Rollback: eliminar usuarios y grupos creados desde el CSV

CSV_FILE="/opt/admin_scripts/usuarios.csv"

if [ ! -f "$CSV_FILE" ]; then
    echo "[ERROR] Archivo $CSV_FILE no encontrado."
    exit 1
fi

echo "⚠️  Este script eliminará los usuarios y grupos definidos en $CSV_FILE"
read -p "¿Continuar? (si/no): " CONFIRM

if [ "$CONFIRM" != "si" ]; then
    echo "Cancelado."
    exit 0
fi

cat "$CSV_FILE" | while IFS=',' read -r USERNAME GROUPNAME; do
    USERNAME=$(echo "$USERNAME" | tr -d '[:space:]')
    GROUPNAME=$(echo "$GROUPNAME" | tr -d '[:space:]')

    if id "$USERNAME" &>/dev/null; then
        sudo userdel -r "$USERNAME"
        echo "[OK] Usuario '$USERNAME' eliminado."
    else
        echo "[INFO] Usuario '$USERNAME' no existe."
    fi

    if grep -q "^$GROUPNAME:" /etc/group; then
        MEMBERS=$(grep "^$GROUPNAME:" /etc/group | cut -d':' -f4)
        if [ -z "$MEMBERS" ]; then
            sudo groupdel "$GROUPNAME"
            echo "[OK] Grupo '$GROUPNAME' eliminado."
        else
            echo "[INFO] Grupo '$GROUPNAME' tiene miembros, no se elimina."
        fi
    fi
done
```

![alt text](<Screenshot 2026-06-03 004646.png>)

**Explicación del script:**

- `read -p "¿Continuar? (si/no): " CONFIRM`: solicita una confirmación interactiva al usuario antes de proceder. Esto evita eliminaciones accidentales al ejecutar el script por error.
- `[ "$CONFIRM" != "si" ]`: si la respuesta no es exactamente "si", el script termina con `exit 0` (salida limpia sin error).
- `userdel -r "$USERNAME"`: elimina el usuario y su directorio home (`-r`). Sin `-r`, el directorio quedaría huérfano en el sistema.
- `cut -d':' -f4`: divide la línea de `/etc/group` por `:` y extrae el cuarto campo, que contiene la lista de miembros del grupo.
- `[ -z "$MEMBERS" ]`: verifica si la variable está vacía (`-z` = zero length). Solo elimina el grupo si no tiene miembros para evitar dejar usuarios sin grupo primario.

> **Nota:** Los mensajes `mail spool not found` son advertencias normales de `userdel`, indicando que los usuarios no tenían buzón de correo configurado. No afectan la eliminación del usuario.

**Ejecución:**

```bash
sudo chmod +x /opt/admin_scripts/user_cleanup.sh
sudo /opt/admin_scripts/user_cleanup.sh
```

Al ejecutarse, el script solicita confirmación antes de proceder:

```
⚠️  Este script eliminará los usuarios y grupos definidos en /opt/admin_scripts/usuarios.csv
¿Continuar? (si/no): si
```

- Se debe escribir exactamente `si` (sin tilde) para confirmar, ya que el script compara la respuesta con la cadena literal `"si"`.
![alt text](<Screenshot 2026-06-03 004728.png>)
Verificación de que los usuarios fueron eliminados:

```bash
id ana_sistemas
grep "sistemas" /etc/group
```

- `id ana_sistemas`: debe devolver `id: 'ana_sistemas': no such user`, confirmando que el usuario fue eliminado.
- `grep "sistemas" /etc/group`: no debe devolver ninguna línea, confirmando que el grupo también fue eliminado.

---

### Ejercicio 10: Health Check Automático de Servicios y Disco

**Archivo:** `/opt/admin_scripts/06_check_system.sh`

```bash
#!/bin/bash
# Health check de servicios críticos y recursos

LOGFILE="/var/log/system_check.log"
sudo touch $LOGFILE

if ! systemctl is-active --quiet nginx; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - ALERTA: Nginx inactivo. Reiniciando..." | sudo tee -a $LOGFILE > /dev/null
    sudo systemctl restart nginx
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') - OK: Nginx activo." | sudo tee -a $LOGFILE > /dev/null
fi

USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//g')
THRESHOLD=85

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - CRITICO: Disco al $USAGE%." | sudo tee -a $LOGFILE > /dev/null
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') - OK: Disco al $USAGE%." | sudo tee -a $LOGFILE > /dev/null
fi

MEM_AVAILABLE=$(free | grep Mem | awk '{print $7}')
echo "$(date '+%Y-%m-%d %H:%M:%S') - INFO: Memoria disponible: ${MEM_AVAILABLE}KB." | sudo tee -a $LOGFILE > /dev/null

echo "----------------------------------------"
echo "Últimas entradas del log:"
sudo tail -n 5 $LOGFILE
```
![alt text](<Screenshot 2026-06-03 004831.png>)

**Explicación del script:**

- `sudo touch $LOGFILE`: crea el archivo de log si no existe, sin modificarlo si ya existe.
- `! systemctl is-active --quiet nginx`: el `!` invierte el código de retorno. El bloque `then` se ejecuta solo cuando Nginx está **inactivo**, reiniciándolo automáticamente.
- `sudo systemctl restart nginx`: reinicia el servicio si se detecta que está caído, implementando una auto-recuperación básica.
- `free | grep Mem | awk '{print $7}'`: obtiene la memoria disponible del sistema. `free` muestra el uso de memoria, `grep Mem` filtra la línea de memoria RAM, y `awk '{print $7}'` extrae la séptima columna (memoria disponible en KB).
- `| sudo tee -a $LOGFILE > /dev/null`: escribe en el log pero redirige la salida de `tee` a `/dev/null` para que no se muestre en pantalla (solo se guarda en el archivo).
- `sudo tail -n 5 $LOGFILE`: muestra las últimas 5 entradas del log al finalizar para confirmar que los registros se guardaron correctamente.

**Ejecución:**

```bash
sudo chmod +x /opt/admin_scripts/06_check_system.sh
sudo /opt/admin_scripts/06_check_system.sh
```
![alt text](<Screenshot 2026-06-03 004858.png>)
- Se ejecuta con `sudo` porque el script necesita escribir en `/var/log/system_check.log` y reiniciar servicios si fuera necesario.
- La salida muestra el estado de Nginx, porcentaje de uso de disco y memoria disponible en KB.

---

### Ejercicio 11: Menú Interactivo de Administración

**Creación del usuario `menu`:**

```bash
sudo useradd -m -s /bin/bash menu
sudo passwd menu
sudo visudo
# Agregar: menu    ALL=(ALL) NOPASSWD: ALL
```

- `useradd -m -s /bin/bash menu`: crea el usuario `menu` con directorio home (`-m`) y shell Bash (`-s /bin/bash`).
- Se agrega al sudoers con `NOPASSWD` para que el menú pueda ejecutar scripts administrativos sin interrumpir al usuario con una solicitud de contraseña.

**Archivo:** `/opt/admin_scripts/07_admin_menu.sh`

```bash
#!/bin/bash
# Menú interactivo de administración

CHECK_SCRIPT="/opt/admin_scripts/06_check_system.sh"
USER_SCRIPT="/opt/admin_scripts/05_user_manager.sh"

while true; do
    clear
    echo "=========================================="
    echo "  M E N Ú   D E   A D M I N I S T R A C I Ó N"
    echo "=========================================="
    echo "1) Health Check (Servicios y Disco)"
    echo "2) Gestión Masiva de Usuarios (CSV)"
    echo "3) Ver Logs de Acceso"
    echo "4) Salir"
    echo "=========================================="
    read -p "Ingrese una opción [1-4]: " opcion

    case $opcion in
        1)
            echo "Ejecutando health check..."
            sudo bash "$CHECK_SCRIPT"
            read -p "Presione Enter para continuar..."
            ;;
        2)
            echo "Ejecutando gestión de usuarios..."
            sudo bash "$USER_SCRIPT"
            read -p "Presione Enter para continuar..."
            ;;
        3)
            echo "Mostrando últimos registros..."
            if [ -f /tmp/admin_access.log ]; then
                tail -n 10 /tmp/admin_access.log
            else
                echo "No hay registros aún."
            fi
            read -p "Presione Enter para continuar..."
            ;;
        4)
            echo "Saliendo del menú. ¡Hasta pronto!"
            break
            ;;
        *)
            echo "Opción inválida. Intente de nuevo."
            read -p "Presione Enter para continuar..."
            ;;
    esac
done
```
![alt text](<Screenshot 2026-06-03 004958.png>)

**Explicación del script:**

- `while true`: crea un bucle infinito que mantiene el menú activo hasta que el usuario seleccione la opción de salir. Sin este bucle, el menú se cerraría después de ejecutar una sola opción.
- `clear`: limpia la pantalla antes de mostrar el menú en cada iteración, mejorando la experiencia visual del usuario.
- `read -p "Ingrese una opción [1-4]: " opcion`: muestra un mensaje y espera que el usuario ingrese un valor, guardándolo en la variable `opcion`.
- `case $opcion in`: estructura de selección múltiple que evalúa el valor de `opcion` y ejecuta el bloque correspondiente. Es más legible que múltiples `if-elif`.
- `;;`: marca el fin de cada caso en la estructura `case`, equivalente a un `break` en otros lenguajes.
- `*)`: patrón comodín que captura cualquier valor no contemplado en los casos anteriores, mostrando un mensaje de opción inválida.
- `break`: termina el bucle `while true` cuando el usuario elige la opción 4.

**Dar permisos de ejecución al script:**

```bash
sudo chmod +x /opt/admin_scripts/07_admin_menu.sh
```


**Configuración de auto-ejecución al iniciar sesión:**

```bash
sudo nano /home/menu/.bashrc
# Agregar al final:
if [ -f "/opt/admin_scripts/07_admin_menu.sh" ]; then
    bash /opt/admin_scripts/07_admin_menu.sh
fi
```
![alt text](<Screenshot 2026-06-03 005044.png>)

- `.bashrc`: archivo de configuración de Bash que se ejecuta automáticamente cada vez que el usuario inicia una sesión interactiva. Al agregar el script aquí, el menú aparece automáticamente al hacer `su - menu`.
- `[ -f "..." ]`: verifica que el script exista antes de intentar ejecutarlo, evitando errores si el archivo fuera eliminado.

**Prueba del menú:**

```bash
su - menu
```

- `su - menu`: cambia al usuario `menu` iniciando una sesión de login completa (el `-` carga el entorno del usuario, incluyendo `.bashrc`). Al cargar `.bashrc`, se ejecuta automáticamente el script del menú.
- Al seleccionar la opción `1`, se ejecuta el health check mostrando el estado de Nginx, disco y memoria.
- Al seleccionar la opción `3`, se muestran los últimos 10 registros del log de acceso `/tmp/admin_access.log`.
- Al seleccionar la opción `4`, el bucle `while true` se interrumpe con `break` y regresa al prompt del sistema.

---

![alt text](<Screenshot 2026-06-03 005425.png>)

## 6. Práctica Grupal

### 6.1 Script Personalizado de Deploy

Se desarrolló el script `grupoX_deploy.sh` que acepta el nombre del grupo e integrantes como argumentos, crea un directorio web y genera una página HTML personalizada.

**Archivo:** `/opt/admin_scripts/grupoX_deploy.sh`

```bash
#!/bin/bash
GRUPO=$1
INTEGRANTE1=$2
INTEGRANTE2=$3
DIR="/var/www/$GRUPO"

if [ -z "$GRUPO" ]; then
    echo "Uso: $0 <nombre_grupo> <integrante1> <integrante2>"
    exit 1
fi

sudo mkdir -p "$DIR"
sudo bash -c "echo '<h1>Grupo $GRUPO</h1><p>$INTEGRANTE1 y $INTEGRANTE2</p>' > $DIR/index.html"
echo "$(date): Deploy de $GRUPO completado." | sudo tee -a /tmp/deploy.log
echo "[OK] Directorio $DIR creado con index.html"
```
![alt text](<Screenshot 2026-06-03 005558.png>)
**Explicación del script:**

- `[ -z "$GRUPO" ]`: verifica si la variable `$GRUPO` está vacía (`-z` = zero length). Si no se pasó el nombre del grupo como argumento, muestra el uso correcto y termina.
- `$0`: variable especial que contiene el nombre del script mismo, útil para mostrar mensajes de uso correctos.
- La página `index.html` se genera dinámicamente con los datos recibidos como argumentos, permitiendo personalizar el deploy para cualquier grupo sin modificar el script.

**Ejecución en Admin y replicación en Target:**

```bash
sudo chmod +x /opt/admin_scripts/grupoX_deploy.sh
/opt/admin_scripts/grupoX_deploy.sh "grupo1" "Juan Pablo" "Compañero"
```
![alt text](<Screenshot 2026-06-03 005627.png>)
- Primero se da permiso de ejecución al script, luego se ejecuta en la VM Admin con los datos del grupo.

Verificación en Admin:

```bash
cat /var/www/grupo1/index.html
cat /tmp/deploy.log
```
![alt text](<Screenshot 2026-06-03 005653.png>)
- `cat /var/www/grupo1/index.html`: muestra el HTML generado con el nombre del grupo e integrantes.
- `cat /tmp/deploy.log`: muestra el registro de la operación con fecha y hora.

Copia y ejecución en Target:

```bash
scp -i ~/.ssh/id_lab61 /opt/admin_scripts/grupoX_deploy.sh jp@192.168.40.3:/tmp/
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3 "sudo bash /tmp/grupoX_deploy.sh 'grupo1' 'Juan Pablo' "Compañero""
```

Verificación en Target:

```bash
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3 "cat /var/www/grupo1/index.html"
```
![alt text](<Screenshot 2026-06-03 005744.png>)
- Ejecuta `cat` remotamente en el Target para verificar que el archivo HTML fue creado correctamente con el mismo contenido que en el Admin.
scp -i ~/.ssh/id_lab61 /opt/admin_scripts/grupoX_deploy.sh jp@192.168.40.3:/tmp/
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3 "sudo bash /tmp/grupoX_deploy.sh 'grupo1' 'Juan Pablo' 'Compañero'"
```

### 6.2 Health Check Cruzado

Se desarrolló un script para verificar desde la VM Target que el servidor web de la VM Admin responde correctamente, simulando una verificación cruzada entre integrantes del grupo.

**Archivo:** `/opt/admin_scripts/health_check_cruzado.sh`

```bash
#!/bin/bash
IP_COMPANERO="192.168.40.2"

echo "=== Health Check Cruzado ==="
echo "Verificando servidor: $IP_COMPANERO"

if curl -s -o /dev/null -w "%{http_code}" http://$IP_COMPANERO | grep -q "200"; then
    echo "[OK] Servidor web de $IP_COMPANERO responde correctamente."
else
    echo "[ALERTA] Servidor web de $IP_COMPANERO NO responde."
fi
```
![alt text](<Screenshot 2026-06-03 005832.png>)
**Explicación del script:**

- `curl -s`: ejecuta curl en modo silencioso (`-s`) para suprimir la barra de progreso y mensajes de error.
- `-o /dev/null`: descarta el cuerpo de la respuesta HTTP, enviándolo a `/dev/null` ya que solo nos interesa el código de estado.
- `-w "%{http_code}"`: indica a curl que imprima solo el código de estado HTTP recibido (ej. 200, 404, 500).
- `grep -q "200"`: verifica si el código recibido es 200 (OK), que indica que el servidor web está funcionando correctamente.

**Ejecución desde Target (via SSH desde Admin):**

```bash
sudo chmod +x /opt/admin_scripts/health_check_cruzado.sh
scp -i ~/.ssh/id_lab61 /opt/admin_scripts/health_check_cruzado.sh jp@192.168.40.3:/tmp/
ssh -i ~/.ssh/id_lab61 jp@192.168.40.3 "bash /tmp/health_check_cruzado.sh"
```
![alt text](<Screenshot 2026-06-03 010052.png>)
- Se copia el script al Target con `scp` y se ejecuta remotamente con `ssh`. El Target hace la petición HTTP hacia el Admin (`192.168.40.2`) y reporta si responde con código 200.

> **Nota:** El nombre de la variable en el script usa `IP_COMPANERO` sin tilde, ya que la Ñ puede causar errores de encoding al copiar el script entre máquinas vía SSH.

### 6.3 Script de Inventario de Servidores

Se desarrolló el script `inventory.sh` que verifica el estado de todos los servidores listados en `servers.txt`, comprobando conectividad de red, disponibilidad del servicio SSH y estado del servidor web Nginx.

**Archivo:** `/opt/admin_scripts/servers.txt`

```
192.168.40.2
192.168.40.3
```

**Creación del archivo `servers.txt`:**

```bash
sudo bash -c 'echo "192.168.40.2" > /opt/admin_scripts/servers.txt'
sudo bash -c 'echo "192.168.40.3" >> /opt/admin_scripts/servers.txt'
cat /opt/admin_scripts/servers.txt
```
![alt text](<Screenshot 2026-06-03 010151.png>)
- El primer comando crea el archivo con la IP del Admin (`>`), el segundo agrega la IP del Target (`>>`).
- `cat /opt/admin_scripts/servers.txt`: muestra el contenido del archivo para verificar que ambas IPs quedaron registradas correctamente.

**Archivo:** `/opt/admin_scripts/inventory.sh`

```bash
#!/bin/bash
SERVERS_FILE="/opt/admin_scripts/servers.txt"
REPORTE="/tmp/inventory_report.txt"

echo "========================================" | tee "$REPORTE"
echo "  INVENTARIO DE SERVIDORES" | tee -a "$REPORTE"
echo "  $(date '+%Y-%m-%d %H:%M:%S')" | tee -a "$REPORTE"
echo "========================================" | tee -a "$REPORTE"

while read IP; do
    echo "" | tee -a "$REPORTE"
    echo "Servidor: $IP" | tee -a "$REPORTE"

    ping -c 1 -W 2 "$IP" > /dev/null && \
        echo "  [OK] Ping" | tee -a "$REPORTE" || \
        echo "  [FAIL] Ping" | tee -a "$REPORTE"

    nc -z -w 2 "$IP" 22 && \
        echo "  [OK] SSH (puerto 22)" | tee -a "$REPORTE" || \
        echo "  [FAIL] SSH (puerto 22)" | tee -a "$REPORTE"

    curl -s -o /dev/null -w "%{http_code}" http://$IP | grep -q "200" && \
        echo "  [OK] Nginx (HTTP 200)" | tee -a "$REPORTE" || \
        echo "  [FAIL] Nginx no responde" | tee -a "$REPORTE"

done < "$SERVERS_FILE"

echo "" | tee -a "$REPORTE"
echo "========================================" | tee -a "$REPORTE"
echo "Reporte guardado en: $REPORTE"
```
![alt text](<Screenshot 2026-06-03 010213.png>)

**Explicación del script:**

- `while read IP; do ... done < "$SERVERS_FILE"`: lee el archivo `servers.txt` línea por línea, asignando cada IP a la variable `IP` en cada iteración del bucle.
- `ping -c 1 -W 2 "$IP"`: envía un solo paquete ICMP (`-c 1`) con un tiempo de espera máximo de 2 segundos (`-W 2`). Si el host responde, el comando retorna código 0 (éxito).
- `> /dev/null`: descarta la salida del ping ya que solo nos interesa si tuvo éxito o no, no el contenido.
- `&&`: operador lógico AND que ejecuta el comando siguiente solo si el anterior tuvo éxito (código de retorno 0).
- `||`: operador lógico OR que ejecuta el comando siguiente solo si el anterior falló (código de retorno distinto de 0).
- `nc -z -w 2 "$IP" 22`: usa netcat para verificar si el puerto 22 (SSH) está abierto. `-z` hace un escaneo sin enviar datos, `-w 2` define el timeout de 2 segundos.
- El reporte se genera simultáneamente en pantalla y en `/tmp/inventory_report.txt` mediante `tee`, dejando evidencia para auditoría.

**Ejecución:**

```bash
sudo chmod +x /opt/admin_scripts/inventory.sh
/opt/admin_scripts/inventory.sh
```
![alt text](<Screenshot 2026-06-03 010248.png>)

- `chmod +x`: otorga permiso de ejecución al script. Es obligatorio antes de poder ejecutarlo directamente con `./script.sh` o con su ruta completa.
- Al ejecutarlo se verifica ping, SSH y Nginx en cada servidor listado en `servers.txt`, mostrando `[OK]` o `[FAIL]` por cada verificación y guardando el resultado en `/tmp/inventory_report.txt`.

---

## 7. Tabla de Scripts Desarrollados

| Script | Función | VM |
|--------|---------|-----|
| `01_intro.sh` | Bienvenida y registro en log | Admin |
| `02_check.sh` | Verificación de archivos y uso de disco | Admin |
| `03_pipes.sh` | Análisis de puertos TCP activos | Admin |
| `04_summarize_logs.sh` | Análisis de logs Nginx con bucles | Admin |
| `05_user_manager.sh` | Gestión masiva de usuarios desde CSV | Admin + Target |
| `deploy_nginx.sh` | Despliegue desatendido de Nginx | Target |
| `log_cleanup.sh` | Limpieza de logs antiguos | Admin |
| `user_cleanup.sh` | Rollback de usuarios creados | Admin |
| `06_check_system.sh` | Health check de servicios y recursos | Admin |
| `07_admin_menu.sh` | Menú interactivo de administración | Admin |
| `grupoX_deploy.sh` | Deploy personalizado por grupo | Admin + Target |
| `health_check_cruzado.sh` | Verificación cruzada de servidor web | Target |
| `inventory.sh` | Inventario de servidores (ping/SSH/HTTP) | Admin |

---

## 8. Conclusiones

- La automatización con Bash permite ejecutar tareas administrativas complejas de forma repetible y confiable, reduciendo el margen de error humano en operaciones como creación masiva de usuarios o despliegue de servicios.

- La combinación de SSH sin contraseña con scripts Bash permite gestionar múltiples servidores desde un único punto de administración, lo que es la base de herramientas de orquestación modernas como Ansible.

- El uso de `IFS` y `read` para procesar archivos CSV demuestra cómo Bash puede manejar datos estructurados sin necesidad de herramientas externas, siendo útil para la automatización de tareas de aprovisionamiento.

- La estructura `case` en el menú interactivo ofrece una interfaz CLI clara y mantenible, permitiendo que usuarios con pocos conocimientos técnicos ejecuten tareas administrativas complejas de forma controlada.

- El script `inventory.sh` implementa un patrón de monitoreo básico que puede escalarse para verificar docenas de servidores, generando reportes de auditoría con fecha y hora para cada verificación.

- La política de `NOPASSWD` en sudoers, aunque conveniente para la automatización, debe aplicarse con cuidado en entornos de producción, limitando su alcance solo a los comandos estrictamente necesarios.

- Los operadores `&&` y `||` en Bash permiten construir lógica condicional en una sola línea, haciendo los scripts más concisos sin sacrificar legibilidad cuando se usan apropiadamente.

---
