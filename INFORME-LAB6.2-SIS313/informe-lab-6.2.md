# Informe de Laboratorio 6.2
## Backups Automáticos, Rotación y Recuperación

---

**Universidad Mayor Real y Pontificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnología**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiante:**                      **Carrera:**

- Juan Pablo Taboada Camacho          Ing. Sistemas

---

## 1. Introducción

El presente laboratorio tiene como objetivo implementar un sistema de backups automáticos, rotación de logs y recuperación de datos en un entorno virtualizado con VirtualBox. Se configuraron dos máquinas con Ubuntu Server 24.04: una actúa como servidor de backups (ejecutando scripts, tareas cron y almacenamiento) y otra como servidor de base de datos y archivos web (MariaDB + Nginx). Se implementaron políticas de retención de datos, verificación de integridad de backups y se documentó el RTO (Recovery Time Objective) del servicio.

Un **backup** es una copia de seguridad de datos que permite restaurar un sistema ante pérdida accidental, corrupción o desastre. El **RTO (Recovery Time Objective)** es el tiempo máximo tolerable para recuperar un servicio tras un fallo. En este laboratorio se mide y documenta ese tiempo de forma práctica.

---

## 2. Objetivos del Laboratorio

- **Desarrollar scripts de backup** para archivos (`tar`) y bases de datos (`mysqldump`).
- **Implementar backups remotos seguros** mediante SSH con compresión (`gzip`).
- **Aplicar políticas de retención** de datos y rotación de logs automatizadas.
- **Planificar tareas periódicas** con `cron` para la ejecución desatendida de backups.
- **Verificar la recuperación** de datos y servicios, documentando el RTO.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
Internet (NAT)
      |
[Lab6.2-Backup - Ubuntu Server 24.04]
  enp0s3 (NAT - DHCP)  |  enp0s8 (Red Interna - 192.168.50.2/29)
                        |
         [Red Interna VirtualBox - 192.168.50.0/29]
         Segmento: 192.168.50.0 | Broadcast: 192.168.50.7
                        |
[Lab6.2-DB - Ubuntu Server 24.04]
  enp0s8 (Red Interna - 192.168.50.3/29)
  Servicios: MariaDB (puerto 3306) + Nginx (puerto 80)
```

### 3.2 Esquema de la Infraestructura

| Máquina Virtual | Hostname | Rol                                                 | Interfaces        | IP Interna (`/29`) |
|-----------------|----------|-----------------------------------------------------|-------------------|--------------------|
| `Lab6.2-Backup` | `backup` | Servidor de Backups (Scripts, Cron, Almacenamiento) | NAT + Red Interna | `192.168.50.2`     |
| `Lab6.2-DB`     | `db`     | Servidor de Base de Datos (MariaDB + Nginx)         | Red Interna       | `192.168.50.3`     |

### 3.3 Tabla de Direccionamiento

| VM              | Interfaz | IP             | Máscara           | Gateway        | Rol                    |
|-----------------|----------|----------------|-------------------|----------------|------------------------|
| `Lab6.2-Backup` | `enp0s3` | DHCP (NAT)     | -                 | -              | Salida a Internet      |
| `Lab6.2-Backup` | `enp0s8` | `192.168.50.2` | `255.255.255.248` | -              | Gateway de red interna |
| `Lab6.2-DB`     | `enp0s8` | `192.168.50.3` | `255.255.255.248` | `192.168.50.2` | Servidor de datos      |

### 3.4 Reenvío de Puertos (Port Forwarding)

| Nombre | Protocolo | IP Host   | Puerto Host | Puerto Invitado | Propósito                       |
|--------|-----------|-----------|-------------|-----------------|---------------------------------|
| SSH    | TCP       | 127.0.0.1 | 2222        | 22              | Acceso SSH desde el host físico |

---

## 4. Preparación del Entorno

### 4.1 Configuración de Red en VirtualBox

Se creó la red interna `Red_Lab6_2` en VirtualBox (**Herramientas → Redes → Crear**).

- **VM Lab6.2-Backup:** Adaptador 1 en modo NAT (acceso a internet) + Adaptador 2 en Red Interna.
- **VM Lab6.2-DB:** Adaptador 1 en Red Interna (sin acceso directo a internet; usa Backup como gateway).

### 4.2 Configuración de IP Estática en VM Backup

Netplan es el gestor de configuración de red en Ubuntu Server 24.04. El archivo `50-cloud-init.yaml` define las interfaces de red en formato YAML.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml

```
sudo          : Ejecuta el comando con privilegios de superusuario (root)
                Necesario porque los archivos en /etc/ son propiedad de root
                nano          : Editor de texto de terminal, más sencillo que vim
/etc/netplan/ : Directorio donde Netplan busca sus archivos de configuración
50-cloud-init.yaml : El prefijo numérico (50) indica el orden de carga;
                     archivos con número menor se procesan primero

Contenido del archivo:

```yaml
network:
  version: 2                  # Versión del esquema de Netplan (siempre 2 en Ubuntu 20+)
  ethernets:
    enp0s3:
      dhcp4: true             # Obtiene IP automática del servidor DHCP del adaptador NAT
    enp0s8:
      dhcp4: no               # No usar DHCP; la IP será asignada manualmente
      optional: true          # Si esta interfaz no está disponible al arrancar,
                              # el sistema no espera indefinidamente por ella
      addresses:
        - 192.168.50.2/29     # IP estática + prefijo de subred /29 (máscara 255.255.255.248)
                              # /29 permite 6 hosts: .1 a .6 (siendo .0 red y .7 broadcast)
      nameservers:
        addresses:
          - 8.8.8.8           # Servidor DNS primario de Google
                              # Permite resolver nombres de dominio (ej: google.com → IP)
```

![alt text](<Screenshot 2026-06-02 214116.png>)

```bash
sudo netplan apply
# netplan apply : Lee todos los archivos YAML en /etc/netplan/ y aplica
#                 la configuración de red sin necesidad de reiniciar la VM
#                 Si hay un error de sintaxis YAML, el comando falla y muestra la línea
#                 Si la configuración es correcta, las interfaces se reconfiguran al instante
```

### 4.3 Configuración de IP Estática en VM DB

La VM DB no tiene interfaz NAT, por lo que necesita definir una ruta por defecto apuntando al Backup como gateway para tener conectividad con el exterior.

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.50.3/29       # IP estática del servidor DB en la red interna
      nameservers:
        addresses:
          - 192.168.50.2        # Usa la VM Backup como servidor DNS/gateway
                                # Backup reenvía las consultas DNS a 8.8.8.8
      routes:
        - to: default           # Ruta por defecto (equivalente a 0.0.0.0/0)
                                # Aplica a todo el tráfico que no tenga ruta específica
          via: 192.168.50.2     # Dirección del siguiente salto (hop): la VM Backup
                                # Todo el tráfico externo de DB sale por aquí
```

![alt text](<Screenshot 2026-06-02 215106.png>)

```bash
sudo netplan apply
# Mismo comportamiento que en VM Backup: aplica la nueva configuración de red en caliente
```

### 4.4 Habilitar Reenvío de Paquetes en VM Backup

Para que la VM Backup funcione como router/gateway de la VM DB, el kernel debe estar configurado para reenviar paquetes entre interfaces de red.

```bash
sudo nano /etc/sysctl.conf
# sysctl.conf : Archivo de configuración de parámetros del kernel de Linux
#               Cada línea tiene el formato: nombre.del.parametro = valor
#               Buscar y descomentar (quitar el #) la línea:
#               net.ipv4.ip_forward=1
#               Esta línea habilita el enrutamiento IP entre interfaces
```
![alt text](<Screenshot 2026-06-02 214300.png>)


```bash
sudo sysctl -p
# sysctl    : Herramienta para leer y modificar parámetros del kernel en tiempo real
# -p        : Load — carga y aplica los parámetros del archivo /etc/sysctl.conf
#             Sin -p, los cambios en el archivo no tendrían efecto hasta reiniciar
# Salida esperada: net.ipv4.ip_forward = 1
#                  Confirma que el reenvío de paquetes está activado en el kernel
```
![alt text](<Screenshot 2026-06-02 214316.png>)


```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
# iptables       : Herramienta de administración del firewall/NAT del kernel Linux
# -t nat         : Tabla — opera en la tabla 'nat' (Network Address Translation)
#                  Existen otras tablas: filter (firewall), mangle (modificar paquetes)
# -A POSTROUTING : Append — agrega la regla a la cadena POSTROUTING
#                  POSTROUTING se aplica DESPUÉS de que el kernel decide la ruta del paquete,
#                  justo antes de que salga por la interfaz de red
# -o enp0s3      : Output interface — la regla solo aplica a paquetes que SALEN
#                  por la interfaz enp0s3 (la interfaz NAT con acceso a internet)
# -j MASQUERADE  : Jump MASQUERADE — acción a tomar: sustituir la IP origen del paquete
#                  por la IP asignada dinámicamente a enp0s3
#                  Diferencia con SNAT: MASQUERADE se adapta automáticamente si la IP cambia
#                  (ideal para IPs dinámicas por DHCP como en este caso)
# Resultado: cuando DB (192.168.50.3) envía un paquete a internet,
#            Backup reemplaza la IP origen por su propia IP pública antes de reenviarlo
```

```bash
sudo apt install iptables-persistent -y
# apt install         : Instala paquetes desde los repositorios de Ubuntu
# iptables-persistent : Paquete que instala un servicio systemd que guarda y restaura
#                       las reglas de iptables automáticamente al iniciar el sistema
# -y                  : Responde 'yes' automáticamente a todas las confirmaciones
#                       Sin -y, apt preguntaría "¿Desea continuar? [S/n]"
```

![alt text](<Screenshot 2026-06-02 214451.png>)

```bash
sudo netfilter-persistent save
# netfilter-persistent : Herramienta del paquete iptables-persistent
# save                 : Guarda las reglas actuales de iptables en archivos:
#                        /etc/iptables/rules.v4 → reglas IPv4
#                        /etc/iptables/rules.v6 → reglas IPv6
#                        Estos archivos se cargan automáticamente al arrancar el sistema
```

![alt text](<Screenshot 2026-06-02 214632.png>)

### 4.5 Preparar Directorios de Trabajo

**En VM Backup:**

```bash
sudo mkdir -p /opt/backup_scripts
# mkdir         : Make Directory — crea un directorio
# -p            : Parents — crea también todos los directorios padre necesarios
#                 Sin -p, fallaría si /opt no existiera
#                 Con -p, no da error si el directorio ya existe (idempotente)
# /opt/         : Directorio estándar FHS (Filesystem Hierarchy Standard) para
#                 software de terceros o scripts de administración adicionales
# backup_scripts: Nombre descriptivo del subdirectorio para nuestros scripts
```

```bash
sudo mkdir -p /var/backups/data_center
# /var/         : Directorio para datos variables (cambian durante la operación normal)
# /var/backups/ : Directorio estándar del sistema destinado a almacenar backups
# data_center   : Subdirectorio para backups de bases de datos
```

```bash
sudo mkdir -p /var/backups/files
# files : Subdirectorio separado para backups de archivos web (tar.gz)
#         Separar BD y archivos facilita la gestión y restauración independiente
```

```bash
sudo chmod 755 /opt/backup_scripts
# chmod         : Change Mode — cambia los permisos de un archivo o directorio
# 755           : Notación octal de permisos:
#                 7 (dueño)  = 4(leer) + 2(escribir) + 1(ejecutar) = rwx
#                 5 (grupo)  = 4(leer) + 0 + 1(ejecutar)           = r-x
#                 5 (otros)  = 4(leer) + 0 + 1(ejecutar)           = r-x
#                 El dueño puede todo; grupo y otros pueden leer y ejecutar (no escribir)
#                 Necesario para que los scripts dentro puedan ser ejecutados
```

![alt text](<Screenshot 2026-06-02 214049.png>)

**En VM DB:**

```bash
sudo mkdir -p /var/www/html
# /var/www/html : Directorio raíz estándar del servidor web Nginx
#                 Es donde Nginx busca los archivos para servir por defecto

sudo mkdir -p /var/log/nginx/archive
# /var/log/nginx/ : Directorio donde Nginx guarda sus logs de acceso y error
# archive         : Subdirectorio para almacenar los logs ya rotados y comprimidos

sudo mkdir -p /opt/backup_scripts
# Mismo propósito que en VM Backup: almacena el script log_rotate.sh
```

---

## 5. Sección Individual — Ejercicios

### Ejercicio 1: Configuración de SSH sin Contraseña

Para que los scripts de backup puedan conectarse automáticamente a la VM DB sin pedir contraseña cada vez, se configura autenticación por clave pública/privada SSH.

**En VM Backup — generar el par de claves:**

```bash
ssh-keygen -t ed25519 -C "backup@lab62" -f ~/.ssh/id_backup
# ssh-keygen  : Herramienta para generar pares de claves criptográficas SSH
# -t ed25519  : Type — algoritmo de clave a usar
#               Ed25519 es una curva elíptica moderna, más rápida y segura que RSA
#               Genera claves más cortas pero igualmente seguras
# -C "backup@lab62" : Comment — comentario descriptivo que se añade al final
#                     de la clave pública; ayuda a identificar para qué sirve la clave
# -f ~/.ssh/id_backup : File — ruta completa donde guardar la clave privada
#                       ~/.ssh/ es el directorio estándar de configuración SSH del usuario
#                       Se generan DOS archivos:
#                       ~/.ssh/id_backup     → clave PRIVADA (nunca compartir, proteger)
#                       ~/.ssh/id_backup.pub → clave PÚBLICA (se copia al servidor destino)
# Al ejecutar, pide una passphrase; se presiona Enter para dejarla vacía
# (necesario para automatización sin intervención humana)
```

![alt text](<Screenshot 2026-06-02 215714.png>)

```bash
ssh-copy-id -i ~/.ssh/id_backup.pub jp@192.168.50.3
# ssh-copy-id    : Herramienta que copia una clave pública al servidor destino
#                  de forma segura, añadiéndola al archivo authorized_keys
# -i             : Identity file — especifica el archivo de clave pública a copiar
# ~/.ssh/id_backup.pub : El archivo de clave pública generado en el paso anterior
# jp@192.168.50.3 : usuario@host — usuario con el que conectarse y host destino
#                  Esta vez sí pedirá la contraseña de jp en VM DB (última vez)
# Lo que hace internamente:
#   1. Se conecta a 192.168.50.3 con contraseña
#   2. Crea ~/.ssh/authorized_keys si no existe
#   3. Agrega el contenido de id_backup.pub al archivo
#   4. Configura permisos correctos (700 para .ssh/, 600 para authorized_keys)
# Después de esto, SSH usará la clave privada en lugar de contraseña
```

![alt text](<Screenshot 2026-06-02 215722.png>)

**Prueba de conectividad sin contraseña:**

```bash
ssh -i ~/.ssh/id_backup jp@192.168.50.3 "hostname"
# ssh           : Cliente SSH para conexiones remotas seguras
# -i            : Identity — especifica la clave privada a usar para autenticarse
#                 Sin -i, SSH intentaría con ~/.ssh/id_rsa o ~/.ssh/id_ed25519 por defecto
# jp@192.168.50.3 : usuario@IP_del_servidor_destino
# "hostname"    : Comando entre comillas que se ejecuta REMOTAMENTE en VM DB
#                 La salida del comando remoto se muestra en el terminal local
# Resultado esperado: db
# Si no pide contraseña = la clave SSH está configurada correctamente
```

![alt text](<Screenshot 2026-06-02 215731.png>)
> **¿Por qué es necesario SSH sin contraseña?** Los scripts de backup se ejecutan automáticamente por `cron` en segundo plano. No hay ningún usuario interactivo que pueda escribir una contraseña cuando cron lanza el script a las 00:00. La autenticación por clave permite que el proceso sea completamente desatendido.

---

### Ejercicio 2: Preparar el Servidor de Base de Datos

**Instalar MariaDB y Nginx en VM DB:**

```bash
sudo apt update
# apt           : Advanced Package Tool — gestor de paquetes de Ubuntu/Debian
# update        : Descarga la lista actualizada de paquetes disponibles desde
#                 los repositorios configurados en /etc/apt/sources.list
#                 NO instala ni actualiza ningún paquete; solo refresca el índice
#                 Es buena práctica ejecutarlo antes de cualquier instalación
#                 para asegurar que se instalen las versiones más recientes
```

```bash
sudo apt install mariadb-server nginx -y
# install          : Descarga e instala los paquetes especificados
# mariadb-server   : Sistema de gestión de bases de datos relacional
#                    Fork comunitario de MySQL, compatible en sintaxis y protocolo
#                    Incluye el servidor mysqld y el cliente mysql
# nginx            : Servidor web de alto rendimiento y bajo consumo de memoria
#                    Alternativa a Apache, excelente para servir archivos estáticos
# -y               : Yes — responde automáticamente "sí" a todas las confirmaciones
#                    Sin -y, apt preguntaría cuánto espacio se usará y si continuar
# Ambos paquetes se pueden instalar en el mismo comando separados por espacio
```

```bash
sudo systemctl enable --now mariadb
# systemctl     : Herramienta de administración del sistema systemd (init system de Ubuntu)
# enable        : Habilita el servicio para que arranque automáticamente
#                 al iniciar el sistema (crea un enlace simbólico en /etc/systemd/)
# --now         : Además de habilitar para el próximo arranque, inicia el servicio
#                 INMEDIATAMENTE (equivale a: systemctl enable + systemctl start)
# mariadb       : Nombre del servicio (unidad systemd) a gestionar
```

```bash
sudo systemctl enable --now nginx
# Mismo comportamiento que mariadb pero para el servidor web Nginx
# Nginx comenzará a escuchar en el puerto 80 (HTTP) inmediatamente
```

**Crear base de datos y tabla de prueba:**

```bash
sudo mysql -e "CREATE DATABASE IF NOT EXISTS lab62_db;"
# mysql         : Cliente de línea de comandos para MariaDB/MySQL
# -e            : Execute — ejecuta la sentencia SQL indicada entre comillas
#                 sin necesidad de entrar al cliente interactivo (mysql>)
#                 Muy útil para scripts y automatización
# CREATE DATABASE : Sentencia SQL que crea una nueva base de datos
# IF NOT EXISTS   : Cláusula de seguridad que evita error si la BD ya existe
#                   Sin ella, si lab62_db ya existiera daría error "Database exists"
# lab62_db        : Nombre de la base de datos a crear
```

```bash
sudo mysql -e "USE lab62_db; CREATE TABLE IF NOT EXISTS productos (
  id     INT           AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100),
  precio DECIMAL(10,2)
);"
# USE lab62_db       : Selecciona la base de datos activa para los comandos siguientes
# CREATE TABLE       : Crea una nueva tabla con las columnas especificadas
# IF NOT EXISTS      : Evita error si la tabla ya existe
# id INT AUTO_INCREMENT PRIMARY KEY :
#   INT            → tipo entero (4 bytes, hasta 2,147,483,647)
#   AUTO_INCREMENT → el valor se incrementa automáticamente en cada INSERT (1, 2, 3...)
#   PRIMARY KEY    → clave primaria: valor único, no nulo, con índice automático
# nombre VARCHAR(100) :
#   VARCHAR   → cadena de longitud variable (más eficiente que CHAR para textos cortos)
#   (100)     → longitud máxima de 100 caracteres
# precio DECIMAL(10,2) :
#   DECIMAL   → número de punto fijo (exacto, ideal para valores monetarios)
#   (10,2)    → máximo 10 dígitos en total, con exactamente 2 decimales
#               Ej: 99999999.99 es válido; 1200.00 ocupa los dígitos correctamente
```

```bash
sudo mysql -e "USE lab62_db; INSERT INTO productos (nombre, precio)
VALUES ('Laptop', 1200.00), ('Mouse', 25.50), ('Teclado', 45.00);"
# INSERT INTO productos (nombre, precio) : Especifica en qué columnas se insertan datos
#                                          No se incluye 'id' porque AUTO_INCREMENT lo genera
# VALUES ('Laptop', 1200.00), ...        : Múltiples filas en un solo INSERT
#                                          Más eficiente que 3 INSERT separados
#                                          (una sola transacción vs tres)
```

**Verificar datos insertados:**

```bash
sudo mysql -e "SELECT * FROM lab62_db.productos;"
# SELECT *                : Selecciona TODAS las columnas de la tabla
#                           El asterisco (*) es un comodín que significa "todas"
# FROM lab62_db.productos : Especifica la fuente: base_de_datos.tabla
#                           La notación con punto evita tener que hacer USE lab62_db antes
```

| id | nombre  | precio  |
|----|---------|---------|
| 1  | Laptop  | 1200.00 |
| 2  | Mouse   | 25.50   |
| 3  | Teclado | 45.00   |

![alt text](<Screenshot 2026-06-02 215642.png>)

**Crear archivo de credenciales seguro:**

```bash
sudo nano /etc/.my.cnf
# /etc/     : Directorio de archivos de configuración del sistema
# .my.cnf   : Archivo de opciones de MySQL/MariaDB
#             El punto al inicio lo hace "oculto" (no aparece con ls sin -a)
#             MariaDB/MySQL leen automáticamente este archivo si existe
```
![alt text](<Screenshot 2026-06-02 215526.png>)

Contenido del archivo:

```ini
[mysqldump]
# [mysqldump] : Sección que aplica solo al comando mysqldump
#               MariaDB tiene múltiples secciones: [mysql], [mysqld], [client], etc.
#               Cada programa lee solo su sección correspondiente

user = root
# user : Usuario con el que mysqldump se autenticará en MariaDB

password = ****
# password : Contraseña del usuario root de MariaDB
#            Al estar en este archivo con permisos 600, nunca aparece
#            en la línea de comandos ni en el historial de bash

single-transaction
# single-transaction : Inicia una transacción antes del dump (para tablas InnoDB)
#                      Garantiza una instantánea consistente de los datos:
#                      todos los datos que se exportan corresponden al mismo punto en el tiempo
#                      Sin esto, si hay escrituras durante el backup, los datos podrían ser inconsistentes
#                      Permite que la BD siga atendiendo lecturas/escrituras durante el backup
#                      (no bloquea las tablas)
```

```bash
sudo chmod 600 /etc/.my.cnf
# 600         : Permisos en octal:
#               6 (dueño)  = 4(leer) + 2(escribir) + 0 = rw-
#               0 (grupo)  = sin permisos              = ---
#               0 (otros)  = sin permisos              = ---
#               Solo el dueño (root) puede leer y modificar el archivo
#               Cualquier otro usuario del sistema obtendrá "Permission denied"
#               CRÍTICO: si este archivo tuviera permisos 644, cualquier usuario
#               podría leer la contraseña de root de MariaDB
```

```bash
sudo chown root:root /etc/.my.cnf
# chown         : Change Owner — cambia el dueño y grupo de un archivo
# root:root     : dueño:grupo → ambos se establecen a root
#                 El primer 'root' es el usuario dueño
#                 El segundo 'root' (después de :) es el grupo
#                 Combinado con chmod 600, solo el proceso root puede acceder al archivo
```

**Verificar permisos correctos:**

```bash
ls -la /etc/.my.cnf
# ls            : List — lista archivos y directorios
# -l            : Long format — muestra permisos, dueño, grupo, tamaño, fecha y nombre
# -a            : All — muestra también archivos ocultos (los que empiezan con punto)
#                 Sin -a, .my.cnf no aparecería porque empieza con punto
# Salida esperada: -rw------- 1 root root 78 jun 3 01:55 /etc/.my.cnf
#                  ↑          ↑ ↑    ↑    ↑
#                  permisos   │ dueño grupo tamaño
#                             └─ número de enlaces duros
```

![alt text](<Screenshot 2026-06-02 215648.png>)

**Probar que las credenciales funcionan localmente:**

```bash
sudo mysqldump --defaults-file=/etc/.my.cnf lab62_db | head -20
# mysqldump               : Herramienta que exporta bases de datos a formato SQL
# --defaults-file=/etc/.my.cnf : Lee usuario, contraseña y opciones del archivo indicado
#                                 IMPORTANTE: debe ser el PRIMER argumento de mysqldump
#                                 Si se pone después del nombre de la BD, es ignorado
# lab62_db                : Nombre de la base de datos a exportar
# | head -20              : Pipe a head: muestra solo las primeras 20 líneas del SQL
#                           Útil para verificar que el dump funciona sin esperar el archivo completo
# Salida esperada: encabezados de MariaDB dump, SET statements, CREATE TABLE productos
# Si sale "Access denied" → las credenciales en .my.cnf son incorrectas
# Si sale "errno 32 on write" al final → es normal cuando head corta el pipe antes de terminar
```

**Configurar sudo sin contraseña para mysqldump (necesario para backup remoto):**

```bash
sudo visudo
# visudo        : Editor seguro para el archivo /etc/sudoers
#                 A diferencia de nano /etc/sudoers, visudo valida la sintaxis ANTES
#                 de guardar, evitando dejar el sistema sin acceso sudo por un error
#                 Abre el archivo con el editor por defecto (nano en Ubuntu)
```
![alt text](<Screenshot 2026-06-02 220456.png>)
Agregar al final del archivo:

```
jp ALL=(ALL) NOPASSWD: /usr/bin/mysqldump
# jp            : El usuario al que aplica esta regla
# ALL           : Desde cualquier host (terminal, SSH, etc.)
# =(ALL)        : Puede ejecutar el comando como cualquier usuario (incluyendo root)
# NOPASSWD:     : No pide contraseña al usar sudo para este comando específico
# /usr/bin/mysqldump : Ruta COMPLETA y EXACTA del comando permitido
#                      Es buena práctica usar ruta absoluta para evitar PATH hijacking
# Propósito: cuando el script en VM Backup ejecuta via SSH:
#   "sudo mysqldump --defaults-file=..."
# El usuario jp en VM DB puede ejecutar mysqldump con sudo sin que el script
# tenga que proporcionar la contraseña (que no podría hacer al ser automatizado)
```

**Crear contenido web de prueba:**

```bash
sudo bash -c 'echo "<h1>Sitio de Prueba - Lab 6.2</h1>" > /var/www/html/index.html'
# sudo bash -c '...' : Ejecuta el comando entre comillas en un subshell con privilegios root
#                      Se usa esta forma porque la redirección (>) también necesita sudo:
#                      "sudo echo 'texto' > archivo" NO funciona porque el shell
#                      abre el archivo ANTES de aplicar sudo, sin permisos suficientes
# echo "..."         : Imprime el texto a stdout
# > /var/www/html/index.html : Redirige stdout al archivo (crea o sobreescribe)
```

```bash
sudo bash -c 'echo "<p>Datos importantes del laboratorio</p>" >> /var/www/html/index.html'
# >>  : Append — agrega al final del archivo sin borrar el contenido existente
#       A diferencia de >, no sobreescribe sino que acumula
```

```bash
sudo mkdir -p /var/www/html/docs
sudo bash -c 'echo "Documento confidencial" > /var/www/html/docs/secreto.txt'
# Crea un subdirectorio y un archivo de texto dentro del sitio web
# Sirve para demostrar que tar captura la estructura completa de directorios
```


![alt text](<Screenshot 2026-06-02 215931.png>)
---

### Ejercicio 3: Backup Local de Archivos con `tar`

**Script `/opt/backup_scripts/file_backup.sh` (VM Backup):**

```bash
#!/bin/bash
# !/bin/bash : Shebang — le indica al sistema operativo qué intérprete usar
#              Cuando el script se ejecuta (./script.sh), el SO lee esta línea
#              y lanza /bin/bash para interpretar el resto del archivo
#              Sin esta línea, el SO intentaría ejecutarlo con sh (shell básico)

DESTINO="/var/backups/files"
FECHA=$(date +%Y%m%d_%H%M)
# $( )        : Command substitution — ejecuta el comando dentro y sustituye
#               el resultado en la variable
# date        : Comando que muestra la fecha y hora del sistema
# +%Y%m%d_%H%M : Formato de salida:
#               %Y → año con 4 dígitos (ej: 2026)
#               %m → mes con 2 dígitos (ej: 06)
#               %d → día con 2 dígitos (ej: 03)
#               _  → separador literal
#               %H → hora en formato 24h con 2 dígitos (ej: 02)
#               %M → minuto con 2 dígitos (ej: 30)
#               Resultado: 20260603_0230
#               Usar la fecha en el nombre evita sobreescribir backups anteriores

DIR_FUENTE="/var/www/html"

sudo mkdir -p "$DESTINO"

if [ ! -d "$DIR_FUENTE" ]; then
    echo "[ERROR] Directorio fuente $DIR_FUENTE no existe."
    exit 1
    # [ ! -d "$DIR_FUENTE" ] : Condición bash:
    #   [  ]   → corchetes = test (evaluación de condición)
    #   !      → NOT (negación lógica)
    #   -d     → Directory — verdadero si el argumento existe y es un directorio
    #   Resultado: si el directorio NO existe, entrar al bloque if
    # exit 1   : Termina el script devolviendo código de salida 1
    #            Convención Unix: 0 = éxito, cualquier otro valor = error
    #            Permite que otros scripts o cron detecten si hubo fallo
fi

sudo tar -czf "$DESTINO/web-$FECHA.tar.gz" -C "$(dirname $DIR_FUENTE)" "$(basename $DIR_FUENTE)"
# tar          : Tape ARchiver — herramienta de archivado y compresión
# -c           : Create — crea un nuevo archivo tar (modo creación)
# -z           : gZip — comprime el archivo usando el algoritmo gzip
#                Reduce el tamaño típicamente entre 60-90% para texto/SQL/HTML
# -f archivo   : File — el SIGUIENTE argumento es el nombre del archivo de salida
#                IMPORTANTE: -f debe ir inmediatamente antes del nombre del archivo
# "$DESTINO/web-$FECHA.tar.gz" : Nombre del archivo de backup con fecha incluida
# -C directorio : Change directory — cambia al directorio indicado ANTES de archivar
#                 $(dirname $DIR_FUENTE) = dirname /var/www/html = /var/www
#                 Hace que tar trabaje desde /var/www
# "$(basename $DIR_FUENTE)" : Solo archiva este nombre relativo
#                 basename /var/www/html = html
#                 Resultado: el tar contiene 'html/index.html' en lugar de
#                 '/var/www/html/index.html' (sin rutas absolutas)
#                 Esto es importante para restaurar en otra ubicación si fuera necesario

if [ $? -eq 0 ]; then
    # $?      : Variable especial que contiene el código de salida del ÚLTIMO comando
    #           0 = el comando anterior (tar) terminó con éxito
    #           Cualquier otro valor = hubo un error
    # -eq     : Equal — operador de comparación numérica (igual a)
    echo "[OK] Backup creado: $DESTINO/web-$FECHA.tar.gz"
    ls -lh "$DESTINO/web-$FECHA.tar.gz"
    # ls -lh  : Lista el archivo en formato largo (-l) con tamaños legibles por humanos
    #           -h: human-readable → muestra 207 como "207" o "2.1K" en lugar de bytes crudos
else
    echo "[ERROR] Falló la creación del backup."
    exit 1
fi
```

![alt text](<Screenshot 2026-06-02 220206.png>)

**Ejecución:**

```bash
sudo chmod +x /opt/backup_scripts/file_backup.sh
# chmod         : Change Mode — modifica los permisos del archivo
# +x            : Agrega (+) el permiso de ejecución (x = execute)
#                 para el dueño, grupo y otros simultáneamente
#                 Sin este paso, intentar ejecutar el script daría:
#                 "bash: /opt/backup_scripts/file_backup.sh: Permission denied"
```

```bash
sudo /opt/backup_scripts/file_backup.sh
# Ejecuta el script como root (sudo)
# Se usa la ruta absoluta para evitar ambigüedades
# Resultado:
# [OK] Backup creado: /var/backups/files/web-20260603_0230.tar.gz
# -rw-r--r-- 1 root root 207 jun 3 02:30 /var/backups/files/web-20260603_0230.tar.gz
```
![alt text](<Screenshot 2026-06-02 220041.png>)
---

### Ejercicio 4: Backup Remoto de Base de Datos mediante SSH

**Script `/opt/backup_scripts/db_backup.sh` (VM Backup):**

```bash
#!/bin/bash
DB_HOST="192.168.50.3"           # IP de la VM DB
DB_NAME="lab62_db"               # Base de datos a respaldar
SSH_USER="jp"                    # Usuario SSH en la VM DB
BACKUP_DIR="/var/backups/data_center"
FILE_DATE=$(date +%Y%m%d_%H%M)
SSH_KEY="/home/jp/.ssh/id_backup" # Clave privada para autenticación SSH sin contraseña

sudo mkdir -p "$BACKUP_DIR"

echo "[INFO] Iniciando backup remoto de $DB_NAME desde $DB_HOST..."

ssh -i "$SSH_KEY" "$SSH_USER@$DB_HOST" \
    "sudo mysqldump --defaults-file=/etc/.my.cnf $DB_NAME" \
    | gzip | sudo tee "$BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz" >/dev/null

# ─── Desglose del pipeline completo ───────────────────────────────────────────
#
# ETAPA 1: ssh -i "$SSH_KEY" "$SSH_USER@$DB_HOST" "comando"
#   ssh           : Abre un canal cifrado hacia la VM DB (192.168.50.3)
#   -i "$SSH_KEY" : Usa la clave privada especificada para autenticarse
#                   (no pide contraseña gracias a la configuración del Ejercicio 1)
#   "sudo mysqldump --defaults-file=/etc/.my.cnf $DB_NAME" :
#       El comando entre comillas se ejecuta REMOTAMENTE en VM DB
#       su stdout viaja CIFRADO por el túnel SSH hasta VM Backup
#
# ETAPA 2: mysqldump --defaults-file=/etc/.my.cnf lab62_db
#   mysqldump        : Genera un volcado completo de la BD en formato SQL
#   --defaults-file  : Lee credenciales del archivo seguro (usuario, contraseña)
#                      Evita pasar la contraseña como argumento visible en ps aux
#   lab62_db         : Nombre de la BD a exportar
#   Salida: texto SQL con CREATE TABLE, INSERT INTO, etc. → va a stdout
#
# ETAPA 3: | gzip
#   |    : Pipe — conecta stdout del comando anterior con stdin del siguiente
#   gzip : Lee el SQL desde stdin y lo comprime al vuelo usando el algoritmo Deflate
#          No genera archivos temporales: comprime el stream en memoria en tiempo real
#          Reduce el tamaño típicamente de ~3KB a ~800 bytes (ahorro del 70-90%)
#
# ETAPA 4: | sudo tee "$BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz"
#   tee           : Lee stdin y escribe en DOS destinos simultáneamente:
#                   1. El archivo especificado
#                   2. stdout (que luego se redirige a /dev/null)
#   sudo tee      : Se usa sudo porque /var/backups/ requiere permisos de root para escribir
#                   "sudo gzip > archivo" no funcionaría porque > es del shell (sin sudo)
#   >/dev/null    : Descarta la salida de tee a stdout (no necesitamos verla en terminal)
#
# ──────────────────────────────────────────────────────────────────────────────

if [ $? -eq 0 ]; then
    echo "[OK] Backup completado: $BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz"
    ls -lh "$BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz"
else
    echo "[ERROR] Falló el backup remoto."
    exit 1
fi
```
![alt text](<Screenshot 2026-06-02 220447.png>)

**Ejecución:**

```bash
sudo chmod +x /opt/backup_scripts/db_backup.sh
sudo /opt/backup_scripts/db_backup.sh
# Resultado:
# [INFO] Iniciando backup remoto de lab62_db desde 192.168.50.3...
# [OK] Backup completado: /var/backups/data_center/lab62_db-20260603_0207.sql.gz
# -rw-r--r-- 1 root root 831 jun 3 02:07 /var/backups/data_center/lab62_db-20260603_0207.sql.gz
```

![alt text](<Screenshot 2026-06-02 220343.png>)

**Verificar el contenido del backup sin descomprimir:**

```bash
zcat /var/backups/data_center/lab62_db-*.sql.gz | head -20
# zcat         : Descomprime un archivo .gz y envía su contenido a stdout
#                Equivale a: gunzip -c archivo.gz
#                NO extrae el archivo al disco; trabaja en memoria (streaming)
# lab62_db-*.sql.gz : El asterisco (*) es un comodín que coincide con cualquier texto
#                     El shell expande esto al nombre del archivo más reciente
#                     que coincida con el patrón
# | head -20   : Muestra solo las primeras 20 líneas
#                Útil para verificar que el backup tiene contenido SQL válido
#                sin tener que descomprimir y leer el archivo completo
# Salida esperada:
# -- MariaDB dump 10.19 Distrib 10.11.14-MariaDB
# -- Host: localhost  Database: lab62_db
# -- Table structure for table `productos`
```
![alt text](<Screenshot 2026-06-02 220638.png>)
**Verificar lista de backups creados:**

```bash
ls -lt /var/backups/data_center/
# ls            : List
# -l            : Long format (permisos, dueño, tamaño, fecha, nombre)
# -t            : Time — ordena por fecha de modificación, más reciente primero
#                 Muy útil para ver rápidamente cuál es el backup más nuevo
# Permite detectar si hay backups fallidos (tamaño 0 o muy pequeño)
```

---

### Ejercicio 5: Verificación de Integridad de Backups

Antes de confiar en un backup, es fundamental verificar que no esté corrupto y que contenga los datos esperados.

**Script `/opt/backup_scripts/verify_backup.sh` (VM Backup):**

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/data_center"

LATEST=$(ls -t $BACKUP_DIR/lab62_db-*.sql.gz 2>/dev/null | head -n 1)
# ls -t         : Lista archivos ordenados por fecha, más reciente primero
# 2>/dev/null   : Redirige stderr (descriptor 2) a /dev/null
#                 Si no hay archivos .sql.gz, ls daría error en stderr;
#                 2>/dev/null lo silencia para que el script no muestre mensajes confusos
# | head -n 1   : Toma solo el primer resultado = el archivo más reciente
# El resultado se guarda en la variable $LATEST

if [ -z "$LATEST" ]; then
    echo "[ERROR] No se encontró ningún backup."
    exit 1
    # -z : Zero length — verdadero si la variable está VACÍA (longitud cero)
    #      Si ls no encontró archivos, $LATEST está vacío → entrar al if
fi

echo "[INFO] Verificando backup: $(basename $LATEST)"
# basename $LATEST : Extrae solo el nombre del archivo, sin la ruta completa
#                    Ejemplo: /var/backups/data_center/lab62_db-20260603_0207.sql.gz
#                           → lab62_db-20260603_0207.sql.gz

# ── VERIFICACIÓN 1: El archivo no debe estar vacío ────────────────────────────
SIZE=$(stat -c%s "$LATEST")
# stat          : Muestra información detallada de un archivo (metadatos)
# -c%s          : Custom format con el campo %s = tamaño en bytes
#                 Equivale a ver la columna de tamaño en ls -l pero solo ese dato
if [ "$SIZE" -eq 0 ]; then
    echo "[FALLO] El backup tiene tamaño 0 bytes."
    exit 1
    # -eq         : Equal — comparación numérica de igualdad
    #               Un backup de 0 bytes indica que algo falló durante la creación
fi
echo "[OK] Tamaño del backup: $SIZE bytes."

# ── VERIFICACIÓN 2: El archivo gzip debe ser válido (no corrupto) ─────────────
if gzip -t "$LATEST" 2>/dev/null; then
    echo "[OK] Archivo gzip válido."
    # gzip -t      : Test — verifica la integridad del archivo comprimido
    #                Lee el archivo completo y verifica su checksum CRC32
    #                Si el archivo está incompleto o dañado (bits alterados), falla
    #                No extrae nada al disco; solo comprueba la consistencia matemática
    # 2>/dev/null  : Suprime mensajes de error de gzip (ya capturamos el resultado con if)
    # Si gzip -t devuelve 0 (éxito) → el if es verdadero → "[OK]"
else
    echo "[FALLO] Archivo gzip corrupto."
    exit 1
fi

# ── VERIFICACIÓN 3: Confirmar que contiene estructura de tablas ───────────────
if zcat "$LATEST" | grep -q "CREATE TABLE"; then
    echo "[OK] Contiene estructura de tablas (CREATE TABLE)."
    # grep          : Global Regular Expression Print — busca patrones en texto
    # -q            : Quiet — no imprime las líneas coincidentes, solo devuelve
    #                 código 0 si encontró algo, o 1 si no encontró nada
    # "CREATE TABLE": Patrón a buscar; su presencia confirma que el backup
    #                 incluye la definición de la tabla (estructura)
    # Un backup sin CREATE TABLE podría ser un archivo SQL vacío o mal formado
fi

# ── VERIFICACIÓN 4: Confirmar que contiene datos ─────────────────────────────
COUNT=$(zcat "$LATEST" | grep -c "INSERT INTO")
# grep -c       : Count — en lugar de mostrar líneas, cuenta cuántas coinciden
#                 Devuelve el número de líneas que contienen "INSERT INTO"
# Un backup con estructura pero sin INSERT INTO = tabla creada pero vacía
if [ "$COUNT" -gt 0 ]; then
    echo "[OK] Contiene $COUNT sentencias INSERT INTO."
    # -gt         : Greater Than — mayor que (comparación numérica)
fi

echo "[INFO] Verificación completada exitosamente."
```
![alt text](<Screenshot 2026-06-02 220922.png>)

**Ejecución y resultado:**

```bash
sudo chmod +x /opt/backup_scripts/verify_backup.sh
sudo /opt/backup_scripts/verify_backup.sh
# [INFO] Verificando backup: lab62_db-20260603_0207.sql.gz
# [OK] Tamaño del backup: 831 bytes.
# [OK] Archivo gzip válido.
# [OK] Contiene estructura de tablas (CREATE TABLE).
# [OK] Contiene 1 sentencias INSERT INTO.
# [INFO] Verificación completada exitosamente.
```
![alt text](<Screenshot 2026-06-02 220958.png>)
---

### Ejercicio 6: Reporte de Estado de Backups

**Script `/opt/backup_scripts/backup_report.sh` (VM Backup):**

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/data_center"
FILES_DIR="/var/backups/files"

echo "========================================"
echo "  REPORTE DE BACKUPS - $(date)"
# $(date) : Inserta la fecha y hora actual en el mensaje
#            Sin argumentos, date muestra: "mié 03 jun 2026 02:11:07 UTC"
echo "========================================"

echo ""
echo "--- Backups de Base de Datos ---"
ls -lh "$BACKUP_DIR"/*.sql.gz 2>/dev/null \
| awk '{print $9, " | Tamaño:", $5, " | Fecha:", $6, $7, $8}'
# ls -lh *.sql.gz    : Lista solo los archivos .sql.gz en formato largo con tamaños legibles
# 2>/dev/null        : Suprime el error si no hay archivos .sql.gz
# | awk '{...}'      : AWK — procesador de texto por campos
#   awk divide cada línea en campos separados por espacios:
#   $1 = permisos (-rw-r--r--)
#   $2 = nlinks (1)
#   $3 = dueño (root)
#   $4 = grupo (root)
#   $5 = tamaño (831)
#   $6 = mes (jun)
#   $7 = día (3)
#   $8 = hora (02:07)
#   $9 = nombre completo del archivo
#   print $9, "| Tamaño:", $5, "| Fecha:", $6, $7, $8
#   → imprime solo los campos relevantes en formato legible

TOTAL_BD=$(ls -1 "$BACKUP_DIR"/*.sql.gz 2>/dev/null | wc -l)
# ls -1         : Lista un archivo por línea (formato mínimo)
# wc -l         : Word Count Lines — cuenta el número de líneas
#                 Como cada archivo ocupa una línea, wc -l = número de archivos

echo "Total: $TOTAL_BD backups"

echo ""
echo "--- Uso de Disco en /var/backups ---"
df -h /var/backups
# df            : Disk Free — muestra el espacio disponible en sistemas de archivos
# -h            : Human readable — usa unidades legibles (K, M, G) en lugar de bloques
# /var/backups  : Muestra solo la información del sistema de archivos
#                 que contiene ese directorio (no todos los sistemas de archivos)
```
![alt text](<Screenshot 2026-06-02 221041.png>)

**Resultado:**

```
========================================
  REPORTE DE BACKUPS - mié 03 jun 2026 02:11:07 UTC
========================================

--- Backups de Base de Datos ---
/var/backups/data_center/lab62_db-20260603_0207.sql.gz  | Tamaño: 831  | Fecha: jun 3 02:07
Total: 3 backups

--- Backups de Archivos ---
/var/backups/files/web-20260603_0230.tar.gz  | Tamaño: 207  | Fecha: jun 3 02:30
Total: 1 backups

--- Uso de Disco en /var/backups ---
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  8,1G  4,1G  3,6G  54% /
```
![alt text](<Screenshot 2026-06-02 221114.png>)
---

### Ejercicio 7: Rotación de Logs de Nginx

La rotación de logs es el proceso de archivar el log activo, comprimirlo y crear uno nuevo vacío para evitar que los archivos de log crezcan indefinidamente.

**En VM DB — generar tráfico de prueba:**

```bash
for i in {1..20}; do curl -s http://localhost/ > /dev/null; done
# for i in {1..20}  : Bucle for en bash
#   {1..20}         : Secuencia de números del 1 al 20 (expansión de llaves)
#   i               : Variable del bucle que toma los valores 1, 2, 3... 20
# do ... done       : Bloque que se ejecuta en cada iteración
# curl              : Client URL — herramienta para hacer peticiones HTTP/HTTPS
# -s                : Silent — no muestra barra de progreso ni mensajes de error
#                     Solo importa que la petición se realice, no ver la respuesta
# http://localhost/ : URL del servidor Nginx local (127.0.0.1 puerto 80)
# > /dev/null       : Descarta la respuesta HTML del servidor
#                     Solo nos interesa que cada petición quede registrada en el log
# Cada curl genera una línea en /var/log/nginx/access.log
```

```bash
ls -lh /var/log/nginx/access.log
# Verifica que el log existe y muestra su tamaño antes de rotar
# Resultado: -rw-r----- 1 www-data adm 1,6K jun 3 02:11 /var/log/nginx/access.log

wc -l /var/log/nginx/access.log
# wc -l         : Cuenta líneas del archivo
# Cada petición HTTP genera una línea en el log de acceso
# Resultado esperado: 20 /var/log/nginx/access.log
```
![alt text](<Screenshot 2026-06-02 221220.png>)

**Script `/opt/backup_scripts/log_rotate.sh` (VM DB):**

```bash
#!/bin/bash
LOG_NGINX="/var/log/nginx/access.log"
ARCHIVE="/var/log/nginx/archive"
FECHA=$(date +%Y%m%d)
# +%Y%m%d : Solo fecha sin hora (ej: 20260603)
#            Para logs, la granularidad de fecha es suficiente

sudo mkdir -p "$ARCHIVE"

if [ -f "$LOG_NGINX" ]; then
    # -f            : File — verdadero si el argumento existe y es un archivo regular
    #                 (no un directorio, ni un enlace simbólico, ni un dispositivo)

    # PASO 1: Renombrar el log activo agregando la fecha
    sudo mv "$LOG_NGINX" "$LOG_NGINX.$FECHA"
    # mv (move)     : Renombra/mueve el archivo
    #                 access.log → access.log.20260603
    # COMPORTAMIENTO IMPORTANTE: Nginx aún tiene el archivo antiguo abierto
    # (descriptor de archivo abierto). Seguirá escribiendo en él aunque
    # se haya renombrado, hasta que reciba señal de recargar.
    # Por eso el log renombrado todavía captura accesos durante el proceso.

    # PASO 2: Comprimir el log renombrado y guardarlo en el archivo
    sudo tar -czf "$ARCHIVE/access-$FECHA.tar.gz" -C /var/log/nginx "access.log.$FECHA"
    # tar -czf      : Create + gZip + File → crea archivo comprimido
    # -C /var/log/nginx : Trabaja desde este directorio
    # "access.log.$FECHA" : Archiva solo este archivo (nombre relativo)
    # El resultado es un .tar.gz que contiene el log completo del día

    # PASO 3: Eliminar el archivo original ya comprimido (evitar duplicado)
    sudo rm "$LOG_NGINX.$FECHA"
    # rm            : Remove — elimina el archivo permanentemente
    #                 El contenido ya está seguro en el .tar.gz

    # PASO 4: Crear nuevo archivo de log vacío con permisos correctos para Nginx
    sudo touch "$LOG_NGINX"
    # touch         : Si el archivo no existe, lo crea vacío
    #                 Si ya existe, solo actualiza su timestamp
    #                 Aquí lo usamos para crear un access.log nuevo y vacío

    sudo chown www-data:adm "$LOG_NGINX"
    # chown         : Asigna dueño y grupo al nuevo archivo
    # www-data      : Usuario del proceso Nginx (necesita ser dueño para escribir)
    # adm           : Grupo de administración del sistema (puede leer logs)
    # IMPORTANTE: sin chown, Nginx no podría escribir en el nuevo log
    #             porque root sería el dueño y Nginx corre como www-data

    sudo chmod 640 "$LOG_NGINX"
    # 640           : Permisos del log:
    #   6 (dueño www-data) = rw- : puede leer y escribir (necesario para nginx)
    #   4 (grupo adm)      = r-- : solo leer (administradores pueden revisar logs)
    #   0 (otros)          = --- : sin acceso (logs son información sensible)

    echo "[OK] Log rotado y archivado: $ARCHIVE/access-$FECHA.tar.gz"
fi
```
![alt text](<Screenshot 2026-06-02 221304.png>)

**Ejecución y verificación:**

```bash
sudo chmod +x /opt/backup_scripts/log_rotate.sh
sudo /opt/backup_scripts/log_rotate.sh
# [OK] Log rotado y archivado: /var/log/nginx/archive/access-20260603.tar.gz
```

![alt text](<Screenshot 2026-06-02 221529.png>)

```bash
ls -lh /var/log/nginx/archive/
# Debe mostrar el archivo comprimido:
# -rw-r--r-- 1 root root 224 jun 3 02:15 access-20260603.tar.gz

ls -lh /var/log/nginx/access.log
# Debe mostrar 0 bytes (archivo nuevo vacío):
# -rw-r----- 1 www-data adm 0 jun 3 02:15 /var/log/nginx/access.log
```
![alt text](<Screenshot 2026-06-02 221604.png>)
---

### Ejercicio 8: Retención de Datos y Limpieza Automática

**Script `/opt/backup_scripts/cleanup_old_backups.sh` (VM Backup):**

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/data_center"
FILES_DIR="/var/backups/files"
LOG_ARCHIVE="/var/log/nginx/archive"
DIAS_RETENCION=7   # Política de retención: conservar backups de los últimos 7 días

echo "[INFO] Aplicando política de retención ($DIAS_RETENCION días)..."

# ── Limpiar backups de BD ─────────────────────────────────────────────────────
if [ -d "$BACKUP_DIR" ]; then

    COUNT=$(find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$DIAS_RETENCION | wc -l)
    # find             : Herramienta de búsqueda de archivos y directorios
    # "$BACKUP_DIR"    : Directorio raíz donde buscar (busca también en subdirectorios)
    # -name "*.sql.gz" : Filtra por nombre de archivo usando comodín
    #                    * coincide con cualquier texto → busca archivos que terminen en .sql.gz
    # -mtime +7        : Modification Time — filtra por fecha de última modificación
    #                    +7 → hace MÁS de 7 días (más de 7*24 horas)
    #                    -7 → hace MENOS de 7 días
    #                     7 → exactamente 7 días
    #                    +0 → cualquier archivo (más de 0 días = más de 24 horas)
    # | wc -l          : Cuenta cuántos archivos encontró

    find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$DIAS_RETENCION -delete
    # -delete          : Elimina cada archivo encontrado que cumpla TODOS los criterios
    #                    anteriores (-name y -mtime)
    #                    IMPORTANTE: -delete debe ser el ÚLTIMO argumento de find
    #                    No hay confirmación ni papelera de reciclaje
    #                    En producción se recomienda hacer primero un dry-run sin -delete

    echo "[OK] $COUNT backups de BD eliminados."
fi

# Se repite la misma lógica para archivos .tar.gz y logs archivados
echo "[INFO] Limpieza completada."
```
![alt text](<Screenshot 2026-06-02 221653.png>)

**Ejecución:**

```bash
sudo chmod +x /opt/backup_scripts/cleanup_old_backups.sh
sudo /opt/backup_scripts/cleanup_old_backups.sh
# [INFO] Aplicando política de retención (7 días)...
# [OK] 0 backups de BD eliminados.
# [OK] 0 backups de archivos eliminados.
# [INFO] Limpieza completada.
# Resultado esperado: 0 porque los backups son del día actual (menos de 7 días de antigüedad)
```
![alt text](<Screenshot 2026-06-02 221720.png>)
---

### Ejercicio 9: Planificación con Cron

`cron` es el demonio (proceso en segundo plano) que ejecuta tareas programadas en Linux. Lee archivos llamados "crontabs" donde cada línea define una tarea y cuándo ejecutarla.

```bash
sudo crontab -e
# crontab       : Programa para gestionar las tablas de cron
# -e            : Edit — abre el crontab del usuario en el editor configurado
# sudo          : Edita el crontab del usuario ROOT (no del usuario jp)
#                 Los scripts de backup necesitan permisos de root para ejecutarse
# Al abrir por primera vez pide elegir editor:
#   1. /bin/nano  ← elegir esta opción (más sencillo)
#   2. /usr/bin/vim.basic
#   ...
```
![alt text](<Screenshot 2026-06-02 221754.png>)
Estructura de una línea cron:

```
# ┌────────── minuto       (0-59)
# │ ┌──────── hora         (0-23)
# │ │ ┌────── día del mes  (1-31)
# │ │ │ ┌──── mes          (1-12)
# │ │ │ │ ┌── día semana   (0=domingo ... 6=sábado, 7=domingo también)
# │ │ │ │ │
# * * * * *  /ruta/al/comando argumentos
#
# Comodines:
# *  = cualquier valor (todos)
# */ = cada N unidades (ej: */5 = cada 5 minutos)
# ,  = lista de valores (ej: 1,3,5 = lunes, miércoles, viernes)
# -  = rango (ej: 9-17 = de las 9 a las 17)
```



```bash
# Backup de BD diario a las 00:00 (medianoche)
0 0 * * * /opt/backup_scripts/db_backup.sh > /dev/null 2>&1
# 0           : minuto 0 (en punto, no a y cuarto ni a y media)
# 0           : hora 0 (medianoche)
# * * *       : cualquier día, mes y día de la semana → se ejecuta TODOS LOS DÍAS
# > /dev/null : redirige stdout a /dev/null (descarta salida normal)
# 2>&1        : redirige stderr (2) al mismo destino que stdout (1) = también a /dev/null
#               Sin esto, cron enviaría por correo local la salida del script a root
#               lo que generaría acumulación de correos en /var/mail/root

# Backup de archivos cada domingo a las 02:00
0 2 * * 0 /opt/backup_scripts/file_backup.sh > /dev/null 2>&1
# 0           : minuto 0
# 2           : hora 2 (2 AM)
# * *         : cualquier día del mes y mes
# 0           : domingo (día de la semana 0)
# Se ejecuta cada domingo a las 2 AM → backup semanal del sitio web

# Limpieza cada lunes a las 03:00
0 3 * * 1 /opt/backup_scripts/cleanup_old_backups.sh > /dev/null 2>&1
# 1           : lunes (día de la semana 1)
# Se ejecuta el lunes (día después del backup semanal del domingo)
# El orden lógico es: domingo se hace el backup → lunes se limpian los viejos

# Backup remoto grupal cada hora (al inicio de cada hora)
0 * * * * /opt/backup_scripts/db_backup_remoto.sh > /dev/null 2>&1
# 0           : minuto 0
# *           : cualquier hora → se repite cada hora (00:00, 01:00, 02:00, ...)
```
![alt text](<Screenshot 2026-06-02 221837.png>)

**Verificar que las tareas quedaron guardadas:**

```bash
sudo crontab -l
# -l (list)     : Muestra todas las tareas cron del usuario root sin abrir el editor
#                 Útil para confirmar que los cambios se guardaron correctamente
```
![alt text](<Screenshot 2026-06-02 221908.png>)
### Tabla de Planificación Cron

| Tarea                     | Expresión Cron | Frecuencia            | Propósito                           |
|---------------------------|----------------|-----------------------|-------------------------------------|
| Backup de BD              | `0 0 * * *`    | Diario a las 00:00    | Respalda la BD fuera de horario     |
| Backup de archivos        | `0 2 * * 0`    | Domingos a las 02:00  | Backup semanal del sitio web        |
| Limpieza backups antiguos | `0 3 * * 1`    | Lunes a las 03:00     | Aplica política de retención 7 días |
| Backup remoto grupal      | `0 * * * *`    | Cada hora             | Backup frecuente para demostración  |

---

### Ejercicio 10: Restauración de Archivos desde `tar`

```bash
# PASO 1: Inspeccionar contenido del backup sin extraer nada al disco
tar -tzf /var/backups/files/web-*.tar.gz
# -t            : lisT — lista el contenido del archivo tar sin extraer
# -z            : indica que el archivo está comprimido con gzip (descomprime en memoria)
# -f            : File — el siguiente argumento es el archivo a inspeccionar
# web-*.tar.gz  : el comodín * es expandido por el shell al nombre del archivo existente
# Salida:
# html/
# html/index.html
# Esta verificación previa es importante: confirma qué se va a restaurar
# y cuál será la estructura de directorios resultante
```

![alt text](<Screenshot 2026-06-02 222837.png>)

```bash
# PASO 2: Simular pérdida accidental de datos
sudo rm -rf /var/www/html/*
# rm            : Remove — elimina archivos y/o directorios
# -r            : Recursive — elimina directorios y todo su contenido (necesario para dirs)
# -f            : Force — no pide confirmación, ignora archivos inexistentes
# /var/www/html/* : El asterisco (*) expande a TODOS los archivos y subdirectorios
#                   dentro de html/ pero NO al directorio html en sí
#                   html/ queda vacío pero existente
# ADVERTENCIA: rm -rf es irreversible, no hay papelera de reciclaje en Linux

ls -la /var/www/html/
# -a            : All — muestra archivos ocultos (los que empiezan con punto)
#                 En un directorio vacío solo se ven . (directorio actual) y .. (padre)
# Confirma que html/ está vacío antes de restaurar
```

![alt text](<Screenshot 2026-06-02 223229.png>)

```bash
# PASO 3: Restaurar desde el backup
sudo tar -xzf /var/backups/files/web-*.tar.gz -C /var/www/
# -x            : eXtract — extrae el contenido del archivo (modo restauración)
# -z            : descomprime el gzip antes de extraer
# -f            : File — archivo a extraer
# -C /var/www/  : Extrae en este directorio destino
#                 El backup contiene 'html/index.html'
#                 Con -C /var/www/, extrae como /var/www/html/index.html
#                 Sin -C, extraería en el directorio actual
# Este comando es el INVERSO EXACTO de: tar -czf archivo.tar.gz -C /var/www html
```

```bash
# PASO 4: Verificar restauración exitosa
ls -la /var/www/html/
# Debe mostrar index.html con fecha del backup original (no la fecha actual)
# -rw-r--r-- 1 root root 60 jun 3 02:30 index.html

cat /var/www/html/index.html
# cat           : Concatenate — muestra el contenido del archivo en la terminal
# Salida esperada:
# <h1>Sitio de Prueba - Lab 6.2</h1>
# <p>Datos importantes</p>
```

---

### Ejercicio 11: Restauración de Base de Datos desde SQL

```bash
# PASO 1: Simular desastre — borrar la base de datos (en VM DB)
sudo mysql -e "DROP DATABASE lab62_db;"
# DROP DATABASE : Sentencia SQL que elimina la base de datos y TODOS sus objetos:
#                 tablas, vistas, procedimientos almacenados, etc.
#                 Es una operación IRREVERSIBLE (no hay ROLLBACK posible)
#                 En producción causaría pérdida total de datos → simula un desastre real

sudo mysql -e "SHOW DATABASES;"
# SHOW DATABASES : Lista todas las bases de datos disponibles en el servidor
# Confirma que lab62_db ya NO aparece → simula que el desastre ocurrió
# Solo deben quedar: information_schema, mysql, performance_schema, sys
```
![alt text](<Screenshot 2026-06-02 223311.png>)

```bash
# PASO 2: Desde VM Backup, transferir el archivo de backup a VM DB
scp /var/backups/data_center/lab62_db-20260603_0207.sql.gz jp@192.168.50.3:/tmp/
# scp           : Secure Copy Protocol — copia archivos entre hosts de forma cifrada
#                 Usa SSH como transporte, por lo tanto el tráfico va cifrado
# /var/backups/data_center/lab62_db-20260603_0207.sql.gz :
#                 Ruta completa del archivo a copiar en el host origen (VM Backup)
# jp@192.168.50.3 : usuario@host_destino → usuario jp en la IP de VM DB
# :/tmp/        : Ruta destino en el host remoto (VM DB)
#                 /tmp/ es el directorio temporal del sistema
#                 Todos los usuarios tienen permisos de escritura ahí
#                 El : antes de la ruta es la separación entre host y ruta
# Nota: esta transferencia SÍ pide contraseña porque scp usa autenticación
#       normal (no la clave id_backup configurada para los scripts automatizados)
# Salida: lab62_db-20260603_0207.sql.gz  100%  831   62.7KB/s   00:00
#   100%      = transferencia completada al 100%
#   831       = tamaño en bytes
#   62.7KB/s  = velocidad de transferencia
#   00:00     = tiempo transcurrido (menos de 1 segundo)
```
![alt text](<Screenshot 2026-06-02 223350.png>)
```bash
# PASO 3: En VM DB, recrear la BD vacía
sudo mysql -e "CREATE DATABASE lab62_db;"
# Es necesario crear primero la BD porque el backup generado por mysqldump
# sin la opción --databases no incluye la sentencia CREATE DATABASE ni USE
# Solo contiene el DDL (CREATE TABLE) y DML (INSERT INTO) de los objetos internos
```

```bash
# PASO 4: Restaurar y medir el RTO
time zcat /tmp/lab62_db-*.sql.gz | sudo mysql lab62_db
# time          : Comando de bash que mide la duración del comando que le sigue
#                 Muestra tres valores al terminar:
#                 real = tiempo real total transcurrido (lo que importa para el RTO)
#                 user = tiempo de CPU en modo usuario (proceso mysqldump/zcat)
#                 sys  = tiempo de CPU en modo kernel (operaciones de I/O, red, etc.)
# zcat          : Descomprime el .gz y envía el SQL a stdout sin crear archivo temporal
# | sudo mysql lab62_db :
#   mysql       : Cliente que recibe el SQL por stdin y lo ejecuta contra la BD lab62_db
#   lab62_db    : Nombre de la BD destino donde se ejecutarán los CREATE TABLE e INSERT
# El pipeline completo: descomprime → ejecuta SQL → restaura la BD en un solo paso
```

**Tiempo de restauración (RTO) medido:**

```
real    0m0,852s    ← Tiempo real: lo que percibe el usuario que espera
user    0m0,094s    ← CPU en espacio de usuario
sys     0m0,070s    ← CPU en kernel
```

```bash
# PASO 5: Verificar integridad post-restauración
sudo mysql -e "SHOW DATABASES;"
# lab62_db debe aparecer nuevamente en la lista de bases de datos

sudo mysql -e "SELECT * FROM lab62_db.productos;"
# Verifica que los 3 registros (Laptop, Mouse, Teclado) fueron restaurados correctamente
```

| id | nombre  | precio  |
|----|---------|---------|
| 1  | Laptop  | 1200.00 |
| 2  | Mouse   | 25.50   |
| 3  | Teclado | 45.00   |

> **RTO = 0.852 segundos.** Una base de datos de producción con millones de registros tendría un RTO mayor, pero el procedimiento de restauración es idéntico.

![alt text](<Screenshot 2026-06-02 223454.png>)
---

## 6. Sección Grupal — Recuperación Cruzada

### 6.1 Roles Asignados

| Rol                   | VM                   | Tarea Principal                               |
|-----------------------|----------------------|-----------------------------------------------|
| Integrante 1 (Backup) | `Lab6.2-Backup` (.2) | Ejecutar backup remoto, almacenar, transferir |
| Integrante 2 (DB)     | `Lab6.2-DB` (.3)     | Proveer BD, simular desastre, restaurar       |

### 6.2 Script de Backup Remoto Grupal

**Script `/opt/backup_scripts/db_backup_remoto.sh` (VM Backup):**

```bash
#!/bin/bash
DB_HOST="192.168.50.3"
DB_NAME="lab62_db"
SSH_USER="jp"
BACKUP_DIR="/var/backups/compañero"   # Directorio dedicado para backups grupales
FILE_DATE=$(date +%Y%m%d_%H%M)
SSH_KEY="/home/jp/.ssh/id_backup"

mkdir -p "$BACKUP_DIR"
# Sin sudo porque el directorio lo crea el usuario jp (no en /var/)
# /var/backups/compañero no existe por defecto → mkdir -p lo crea

echo "[INFO] Iniciando backup remoto hacia $BACKUP_DIR..."

ssh -i "$SSH_KEY" "$SSH_USER@$DB_HOST" \
    "sudo mysqldump --defaults-file=/etc/.my.cnf $DB_NAME" \
    | gzip > "$BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz"
# Igual que db_backup.sh pero sin sudo tee porque el directorio compañero
# es accesible por el usuario jp directamente (sin necesidad de root)
# > redirige la salida comprimida directamente al archivo de destino

if [ $? -eq 0 ]; then
    echo "[OK] Backup completado: $BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz"
    ls -lh "$BACKUP_DIR/$DB_NAME-$FILE_DATE.sql.gz"
else
    echo "[ERROR] Falló el backup remoto."
    exit 1
fi
```
![alt text](<Screenshot 2026-06-02 223839.png>)
**Ejecución:**

```bash
sudo chmod +x /opt/backup_scripts/db_backup_remoto.sh
sudo /opt/backup_scripts/db_backup_remoto.sh
# [INFO] Iniciando backup remoto hacia /var/backups/compañero...
# [OK] Backup completado: /var/backups/compañero/lab62_db-20260603_0239.sql.gz
# -rw-r--r-- 1 root root 830 jun 3 02:39 /var/backups/compañero/lab62_db-20260603_0239.sql.gz
```
![alt text](<Screenshot 2026-06-02 223920.png>)

### 6.3 Configuración de UFW en VM DB

UFW (Uncomplicated Firewall) es la interfaz simplificada de `iptables` en Ubuntu.

```bash
sudo apt install ufw -y
# Instala UFW si no está instalado (en Ubuntu Server suele estar preinstalado)
# -y : responde sí automáticamente a la confirmación de instalación
```

```bash
sudo ufw allow from 192.168.50.2 to any port 22
# ufw allow         : Agrega una regla que PERMITE el tráfico especificado
# from 192.168.50.2 : Solo acepta conexiones originadas en esa IP específica (VM Backup)
#                     Cualquier otra IP que intente conectarse al puerto 22 será bloqueada
# to any            : A cualquier interfaz de red de esta VM (enp0s8 en este caso)
# port 22           : Puerto destino 22 = servicio SSH
#                     SSH (Secure Shell) es el protocolo de administración remota
```

```bash
sudo ufw allow from 192.168.50.2 to any port 3306
# port 3306         : Puerto estándar de MySQL/MariaDB
#                     Permite que VM Backup pueda conectarse directamente a la BD
#                     si fuera necesario (para backups directos o consultas)
```

```bash
sudo ufw enable
# Activa el firewall y lo configura para arrancar automáticamente con el sistema
# ADVERTENCIA IMPORTANTE: Si no se configuró la regla de SSH (port 22) ANTES
# de habilitar UFW, se perdería el acceso remoto por SSH inmediatamente
# UFW pregunta confirmación: "Command may disrupt existing ssh connections. Proceed? (y|n)"
# Responder 'y' para confirmar
```

```bash
sudo ufw status
# Muestra el estado actual del firewall y todas las reglas activas
# Status: active → el firewall está habilitado y funcionando
# Resultado esperado:
# To          Action    From
# --          ------    ----
# 22          ALLOW     192.168.50.2
# 3306        ALLOW     192.168.50.2
```

![alt text](<Screenshot 2026-06-02 224051.png>)

**Reglas aplicadas:**

| Puerto | Acción | Origen       | Servicio |
|--------|--------|--------------|----------|
| 22     | ALLOW  | 192.168.50.2 | SSH      |
| 3306   | ALLOW  | 192.168.50.2 | MariaDB  |

### 6.4 Simulación de Desastre

```bash
sudo mysql -e "DROP DATABASE lab62_db;"
# Simula pérdida total: accidente, ransomware, fallo humano o corrupción de datos

sudo mysql -e "SHOW DATABASES;"
# Confirma que lab62_db ya no existe en el servidor
# Solo quedan las BDs del sistema: information_schema, mysql, performance_schema, sys
```

### 6.5 Prueba de Recuperación Cruzada

**Paso 1 — Integrante 1 transfiere el backup a VM DB:**

```bash
scp /var/backups/compañero/lab62_db-20260603_0239.sql.gz jp@192.168.50.3:/tmp/
# Copia segura del backup desde VM Backup hacia VM DB
# Esta transferencia usa SSH estándar (con contraseña), diferente a la clave
# id_backup usada en los scripts automatizados
# Resultado:
# lab62_db-20260603_0239.sql.gz  100%  830   57.5KB/s   00:00
```
![alt text](<Screenshot 2026-06-02 224416.png>)

**Paso 2 — Integrante 2 restaura la BD y mide el RTO:**

```bash
sudo mysql -e "CREATE DATABASE lab62_db;"
# Crea la BD vacía antes de restaurar (ver explicación en Ejercicio 11)

time zcat /tmp/lab62_db-20260603_0239.sql.gz | sudo mysql lab62_db
# time : Mide el tiempo total de la operación de restauración completa
# Este es el RTO real: desde que se inicia la restauración hasta que termina
```
![alt text](<Screenshot 2026-06-02 224153.png>)

**Tiempo de restauración (RTO):**

```
real    0m0,852s
user    0m0,094s
sys     0m0,070s
```

**Paso 3 — Verificación conjunta:**

```bash
sudo mysql -e "SHOW DATABASES;"
# Confirma que lab62_db aparece nuevamente

sudo mysql -e "SELECT * FROM lab62_db.productos;"
# Confirma que los 3 registros fueron recuperados íntegramente
```

![alt text](<Screenshot 2026-06-02 224322.png>)

| id | nombre  | precio  |
|----|---------|---------|
| 1  | Laptop  | 1200.00 |
| 2  | Mouse   | 25.50   |
| 3  | Teclado | 45.00   |

**Resultados de la verificación conjunta:**

| Criterio                       | Resultado |
|--------------------------------|-----------|
| ¿Se recuperó la BD?            | ✅ Sí     |
| ¿Registros recuperados?        | ✅ 3/3    |
| ¿Datos íntegros?               | ✅ Sí     |
| ¿Tiempo de restauración (RTO)? | ✅ 0.852s |
| ¿Backup consistente?           | ✅ Sí     |

---

## 7. Documento de Incidente Simulado

```
## Simulación de Recuperación - Lab 6.2

- Fecha:                  03/06/2026
- Rol:                    Integrante 2 (DB Server) - VM Lab6.2-DB
- Incidente:              Pérdida total de base de datos lab62_db
                          (DROP DATABASE simulado)
- Backup realizado por:   Integrante 1 (VM Lab6.2-Backup)
- Backup ubicado en:      /var/backups/compañero/lab62_db-20260603_0239.sql.gz
- Tiempo de restauración
  (RTO):                  0.852 segundos
- Registros recuperados:  3/3 (Laptop, Mouse, Teclado)
- Lección aprendida:      Un backup remoto automatizado con SSH y mysqldump
                          permite recuperar una base de datos completa en menos
                          de 1 segundo, demostrando que una política de backups
                          bien configurada minimiza el impacto de un desastre
                          sobre los datos críticos.
```

![
  
](<Screenshot 2026-06-02 224638.png>)

---

## 8. Problemas Encontrados y Soluciones

| # | Problema                                               | Causa                                                                      | Solución Aplicada                                                  |
|---|--------------------------------------------------------|----------------------------------------------------------------------------|--------------------------------------------------------------------|
| 1 | `[ERROR] Directorio fuente /var/www/html no existe`    | La VM Backup no tiene Nginx instalado, el directorio no existía            | `sudo mkdir -p /var/www/html` y creación manual del contenido      |
| 2 | `sudo: a password is required` en backup remoto        | El usuario jp no tenía permiso sudo sin contraseña para `mysqldump`        | Agregar regla en `/etc/sudoers` con `NOPASSWD: /usr/bin/mysqldump` |
| 3 | `Access denied for user 'root'@'localhost'`            | El archivo `.my.cnf` usaba `host = 127.0.0.1`; MariaDB usa socket Unix    | Eliminar la línea `host` del `.my.cnf` para usar socket por defecto|
| 4 | `No database selected` al restaurar                    | El backup no incluye `CREATE DATABASE` (mysqldump sin `--databases`)       | Crear la BD manualmente antes de restaurar: `CREATE DATABASE lab62_db;` |
| 5 | `Error writing ... No such file or directory` en nano  | El directorio `/opt/backup_scripts` no existía en la VM DB                | `sudo mkdir -p /opt/backup_scripts` antes de crear el script       |

---

## 9. Resumen de Scripts Implementados

| Script                   | VM     | Función                                             |
|--------------------------|--------|-----------------------------------------------------|
| `file_backup.sh`         | Backup | Backup comprimido de `/var/www/html` con `tar`      |
| `db_backup.sh`           | Backup | Backup remoto de MariaDB vía SSH + `mysqldump`      |
| `verify_backup.sh`       | Backup | Verificación de integridad del backup de BD         |
| `backup_report.sh`       | Backup | Reporte de estado de todos los backups              |
| `log_rotate.sh`          | DB     | Rotación manual de logs de Nginx                    |
| `cleanup_old_backups.sh` | Backup | Limpieza de backups y logs mayores a 7 días         |
| `db_backup_remoto.sh`    | Backup | Backup remoto grupal hacia `/var/backups/compañero` |

---

## 10. Conclusiones

- La combinación de `mysqldump` + SSH + `gzip` permite realizar backups remotos seguros y comprimidos sin exponer credenciales en texto plano. El tráfico viaja cifrado por el túnel SSH, y las credenciales de MariaDB se almacenan en `/etc/.my.cnf` con permisos `600`, inaccesibles para cualquier otro usuario del sistema.

- Almacenar credenciales en `/etc/.my.cnf` con permisos `600` (solo root puede leer) es una buena práctica que evita que la contraseña aparezca en la lista de procesos (`ps aux`), en el historial de bash (`~/.bash_history`), o en scripts visibles por otros usuarios.

- La verificación de integridad con `gzip -t` (checksum CRC32) combinada con la inspección del contenido SQL (`CREATE TABLE`, `INSERT INTO`) son pasos imprescindibles antes de confiar en un backup como única fuente de recuperación. Un archivo de 0 bytes o con checksum inválido sería inútil en el momento del desastre.

- Las tareas `cron` permiten automatizar completamente el ciclo de vida de los backups (creación, rotación y limpieza), eliminando la dependencia de intervención manual y garantizando que los backups se realicen incluso en horarios donde no hay operadores disponibles.

- El RTO medido de **0.852 segundos** para restaurar la base de datos demuestra que una política de backups bien implementada puede minimizar drásticamente el tiempo de inactividad ante un desastre real. Para bases de datos más grandes este tiempo escalaría linealmente, pero el procedimiento es idéntico.

- La rotación de logs de Nginx libera espacio en disco y mantiene un historial comprimido y organizado por fecha en `/var/log/nginx/archive/`. Sin rotación, en un servidor con alto tráfico los logs podrían ocupar varios GB por día, agotando el disco del sistema.

- La política de retención de 7 días con `find -mtime +7 -delete` es un balance entre seguridad (mantener copias recientes) y uso de almacenamiento. En producción esta política se combinaría con backups en almacenamiento externo (nube o NAS) para mayor resiliencia ante desastres físicos.

- La configuración de UFW en VM DB restringiendo SSH y MySQL únicamente a la IP de VM Backup (`192.168.50.2`) aplica el **principio de mínimo privilegio**: solo quien necesita acceder puede hacerlo, reduciendo la superficie de ataque del servidor de datos.

- Durante el laboratorio se presentaron varios errores comunes en producción real: permisos insuficientes, diferencias entre autenticación por socket vs TCP en MariaDB, y backups sin `CREATE DATABASE`. Resolver estos errores refuerza la comprensión de cómo funciona cada componente del sistema de backups.

---
