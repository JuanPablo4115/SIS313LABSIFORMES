# Informe de Laboratorio 4.2
## Servidor DNS Primario e Integración con Servicio Web

---

**Universidad Mayor Real y Pontificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnología**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiantes:**                                             **Carrera:**

- Juan Pablo Taboada Camacho (SERVIDOR DNS)                  Ing. Sistemas
- Franco Milton Jiménez Amachuy (SERVIDOR WEB)               Ing. Sistemas

---

## 1. Introducción

El presente laboratorio tiene como objetivo implementar un servidor DNS primario utilizando BIND9 en un entorno virtualizado con VirtualBox, integrándolo con un servidor web Nginx para permitir el acceso a sitios por nombre de dominio en lugar de dirección IP.

Se configuraron dos máquinas virtuales con Ubuntu Server 24.04 LTS conectadas en modo Puente (Bridge) a la red del aula: una actuando como servidor DNS autoritativo para la zona `sis313lab42jpfranc.local`, y otra como servidor web que sirve contenido estático. Adicionalmente, se configuró una zona inversa para resolución de IPs a nombres, y se aplicaron reglas de firewall UFW para proteger el servicio DNS.

---

## 2. Objetivos del Laboratorio

- **Configurar un servidor DNS primario (maestro)** usando BIND9 para la red del aula con el dominio `sis313lab42jpfranc.local`.

- **Crear y administrar zonas DNS directa e inversa** con registros SOA, NS, A, CNAME y PTR.

- **Verificar la resolución de nombres** mediante herramientas como `dig` y `nslookup`.

- **Integrar el DNS con un servicio web (Nginx)**, permitiendo el acceso al sitio por nombre de dominio en lugar de dirección IP.

- **Configurar el firewall UFW** para permitir consultas DNS desde la red del aula.

- **Comprender el rol del DNS** en la infraestructura de servicios y su funcionamiento en redes reales.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
Internet
     |
  [Gateway — 10.54.119.76]
     |
  [Red del aula — 10.54.119.0/24]
     |
  ┌──┴──────────────────────────────┐
  |                                 |
[VM DNS — BIND9]          [VM Web — Nginx]
 10.54.119.100             10.54.119.101
 Adaptador Puente          Adaptador Puente
```

### 3.2 Tabla de Direccionamiento

| Máquina Virtual | Rol | IP | Adaptador VirtualBox |
|---|---|---|---|
| `Lab4.2-DNS` | Servidor DNS (BIND9) | `10.54.119.100/24` | Puente (Bridge) |
| `Lab4.2-Web` | Servidor Web (Nginx) | `10.54.119.101/24` | Puente (Bridge) |

| Parámetro | Valor |
|---|---|
| Dominio | `sis313lab42jpfranc.local` |
| Red del aula | `10.54.119.0/24` |
| Gateway | `10.54.119.76` |
| DNS externo | `8.8.8.8` |

### 3.3 Esquema de red completo

```
┌─────────────────────────────────────────────────────────────┐
│                  Red del aula — 10.54.119.0/24              │
│                                                             │
│             ┌──────────────────────┐                        │
│             │   Gateway            │                        │
│             │   10.54.119.76       │◄──── Internet          │
│             └──────────┬───────────┘                        │
│                        │                                    │
│           ┌────────────┴────────────┐                       │
│           │                         │                       │
│  ┌────────▼──────────┐   ┌──────────▼──────────┐           │
│  │    VM DNS         │   │    VM Web            │           │
│  │  10.54.119.100    │◄──│  10.54.119.101       │           │
│  │  BIND9 (puerto 53)│   │  Nginx (puerto 80)   │           │
│  │  UFW habilitado   │   │  Virtual Host        │           │
│  └────────▲──────────┘   └──────────────────────┘           │
│           │                                                  │
│  ┌────────┴──────────┐                                       │
│  │  PC Anfitriona    │                                       │
│  │  (Windows)        │                                       │
│  │  DNS → .100       │                                       │
│  └───────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 Configuración de Adaptadores en VirtualBox

#### VM Lab4.2-DNS

```
┌─────────────────────────────────────────────────────┐
│         VirtualBox — VM Lab4.2-DNS                  │
├─────────────────────────────────────────────────────┤
│  Adaptador 1                                        │
│  ┌───────────────────────────────────────────────┐  │
│  │ Habilitar adaptador de red                    │  │
│  │                                               │  │
│  │ Conectado a:  Adaptador puente (Bridge)       │  │
│  │ Nombre:       Tarjeta de red de la PC         │  │
│  │ Modo promiscuo: Denegar                       │  │
│  └───────────────────────────────────────────────┘  │
│  Adaptador 2, 3, 4 → Deshabilitados                 │
└─────────────────────────────────────────────────────┘
```

#### VM Lab4.2-Web

```
┌─────────────────────────────────────────────────────┐
│         VirtualBox — VM Lab4.2-Web                  │
├─────────────────────────────────────────────────────┤
│  Adaptador 1                                        │
│  ┌───────────────────────────────────────────────┐  │
│  │ Habilitar adaptador de red                    │  │
│  │                                               │  │
│  │ Conectado a:  Adaptador puente (Bridge)       │  │
│  │ Nombre:       Tarjeta de red de la PC         │  │
│  │ Modo promiscuo: Denegar                       │  │
│  └───────────────────────────────────────────────┘  │
│  Adaptador 2, 3, 4 → Deshabilitados                 │
└─────────────────────────────────────────────────────┘
```

El modo Puente (Bridge) permite que cada VM obtenga una IP directamente en la red del aula, siendo visible para todas las PCs como si fuera un equipo físico más, a diferencia del modo NAT donde la VM queda oculta detrás de la IP del anfitrión.

---

## 4. Preparación del Entorno

**Máquina Virtual: Lab4.2-DNS (Ubuntu Server 24.04)**

- **Interfaz 1 (`enp0s3`)**: Adaptador Puente (Bridge). Conectada directamente a la red del aula para tener IP visible por todos los equipos y acceso a internet a través del gateway.

**Máquina Virtual: Lab4.2-Web (Ubuntu Server 24.04)**

- **Interfaz 1 (`enp0s3`)**: Adaptador Puente (Bridge). Conectada a la misma red del aula, con el gateway apuntando al router del aula.

---

## 5. Configuración del Servidor DNS (VM Lab4.2-DNS)

### 5.1 Configuración de Red Estática

Abrimos el archivo de configuración de red de Ubuntu con el editor nano:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

![alt text](<Screenshot 2026-05-13 231107.png>)

- `/etc/netplan/50-cloud-init.yaml` es el archivo donde Ubuntu guarda la configuración de red. Netplan es el sistema de configuración de red de Ubuntu que reemplaza al antiguo `/etc/network/interfaces`.

El archivo quedó con el siguiente contenido:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [10.54.119.100/24]
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 10.54.119.76
```

![alt text](<Screenshot 2026-05-13 231036.png>)

- `version: 2` — versión del formato de Netplan que se está usando.
- `ethernets` — sección donde se configuran las interfaces de red físicas.
- `enp0s3` — nombre de la interfaz de red (el adaptador Bridge en VirtualBox).
- `dhcp4: no` — desactiva la asignación automática de IP. Usaremos IP fija.
- `addresses: [10.54.119.100/24]` — asigna la IP estática `10.54.119.100` con máscara `/24` (255.255.255.0), lo que permite comunicarse con cualquier equipo en `10.54.119.0` a `10.54.119.255`.
- `nameservers: addresses: [8.8.8.8]` — indica que el servidor DNS para resolver nombres externos será Google DNS (8.8.8.8).
- `routes: to: default via: 10.54.119.76` — define la ruta por defecto (gateway) hacia internet. Todo el tráfico que no sea local saldrá por `10.54.119.76`.

Aplicamos los cambios sin necesidad de reiniciar:

```bash
sudo netplan apply
```

![alt text](<Screenshot 2026-05-13 231151.png>)

- `netplan apply` lee el archivo YAML y aplica la configuración de red inmediatamente. Si hay un error en el archivo, mostrará un mensaje y no aplicará los cambios.

### 5.2 Instalación de BIND9

Actualizamos la lista de paquetes disponibles:

```bash
sudo apt update
```

Instalamos BIND9:

```bash
sudo apt install bind9 bind9utils bind9-doc -y
```

- `bind9` — el servidor DNS propiamente dicho (BIND = Berkeley Internet Name Domain).
- `bind9utils` — herramientas auxiliares como `named-checkconf` y `named-checkzone` para verificar la configuración.
- `bind9-doc` — documentación de BIND9.

### 5.3 Declaración de Zonas en BIND9

Editamos el archivo de configuración local de BIND9 donde se declaran las zonas que gestionará este servidor:

```bash
sudo nano /etc/bind/named.conf.local
```

![alt text](<Screenshot 2026-05-13 231555.png>)

- `/etc/bind/named.conf.local` — archivo de configuración local de BIND9. Aquí se declaran las zonas DNS que este servidor va a gestionar. Es separado del archivo principal `named.conf` para que las actualizaciones del paquete no sobreescriban nuestra configuración.

Agregamos las siguientes zonas al final del archivo:

```
zone "sis313lab42jpfranc.local" {
    type master;
    file "/etc/bind/db.sis313lab42jpfranc.local";
};

zone "119.54.10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10.54.119";
};
```

![alt text](<Screenshot 2026-05-13 231531.png>)

- `zone "sis313lab42jpfranc.local"` — declara la zona directa para nuestro dominio. Cuando alguien pregunte por un nombre dentro de `sis313lab42jpfranc.local`, este servidor responderá.
- `zone "119.54.10.in-addr.arpa"` — declara la zona inversa para la red `10.54.119.x`. Se escribe al revés porque así funciona el estándar DNS para zonas inversas: la IP `10.54.119.x` se invierte como `119.54.10.in-addr.arpa`.
- `type master` — indica que este servidor es el primario (autoritativo) para esta zona. Tiene la copia original de los registros y puede responder consultas directamente sin preguntar a otro servidor.
- `file` — ruta al archivo que contiene los registros DNS de esa zona.

### 5.4 Creación del Archivo de Zona Directa

Copiamos la plantilla de zona que viene incluida en BIND9 para usarla como base:

```bash
sudo cp /etc/bind/db.local /etc/bind/db.sis313lab42jpfranc.local
```

- `/etc/bind/db.local` — plantilla de zona que viene con BIND9, ya tiene la estructura correcta de un archivo de zona.
- `/etc/bind/db.sis313lab42jpfranc.local` — nombre del archivo que crearemos para nuestra zona.

Editamos el archivo de zona:

```bash
sudo nano /etc/bind/db.sis313lab42jpfranc.local
```

![alt text](<Screenshot 2026-05-13 231817.png>)

El archivo quedó con el siguiente contenido:

```
$TTL    604800
@       IN      SOA     dns.sis313lab42jpfranc.local. admin.sis313lab42jpfranc.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.sis313lab42jpfranc.local.
dns     IN      A       10.54.119.100
www     IN      A       10.54.119.101
web     IN      CNAME   www.sis313lab42jpfranc.local.
```

![alt text](<Screenshot 2026-05-13 231748.png>)

Explicación detallada de cada línea:

- `$TTL 604800` — Time To Live en segundos (equivale a 7 días). Es el tiempo que otros servidores DNS guardarán en caché los registros de esta zona antes de volver a consultarlos.
- `@` — símbolo que representa el nombre de la zona (`sis313lab42jpfranc.local`).
- `IN` — clase Internet, estándar para todos los registros DNS en internet.
- `SOA` (Start of Authority) — registro de inicio de autoridad. Define quién es el servidor principal de esta zona y parámetros de sincronización:
  - `dns.sis313lab42jpfranc.local.` — nombre del servidor DNS primario. El punto final es obligatorio, indica que es un nombre absoluto (FQDN).
  - `admin.sis313lab42jpfranc.local.` — correo del administrador en formato DNS (el primer punto equivale a `@`, es decir `admin@sis313lab42jpfranc.local`).
  - `Serial: 1` — número de versión de la zona. Se debe incrementar cada vez que se modifica el archivo para que los servidores secundarios detecten el cambio.
  - `Refresh: 604800` — cada cuántos segundos los servidores secundarios comprueban si hay cambios (7 días).
  - `Retry: 86400` — cada cuántos segundos reintenta si falla la verificación (1 día).
  - `Expire: 2419200` — después de cuántos segundos los secundarios dejan de responder si no pueden contactar al primario (28 días).
  - `Negative Cache TTL: 604800` — tiempo que se guarda en caché una respuesta negativa (cuando un nombre no existe).
- `@ IN NS dns.sis313lab42jpfranc.local.` — registro NS (Name Server): indica cuál es el servidor de nombres autoritativo para esta zona.
- `dns IN A 10.54.119.100` — registro A: el nombre `dns.sis313lab42jpfranc.local` resuelve a la IP `10.54.119.100` (nuestra VM DNS).
- `www IN A 10.54.119.101` — registro A: el nombre `www.sis313lab42jpfranc.local` resuelve a la IP `10.54.119.101` (nuestra VM Web).
- `web IN CNAME www.sis313lab42jpfranc.local.` — registro CNAME (Canonical Name): `web.sis313lab42jpfranc.local` es un alias de `www.sis313lab42jpfranc.local`, apunta al mismo servidor. El punto final es obligatorio.

### 5.5 Creación del Archivo de Zona Inversa

Copiamos la plantilla de zona inversa:

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.10.54.119
```

- `/etc/bind/db.127` — plantilla de zona inversa que viene con BIND9 (usada originalmente para `127.0.0.1`). Tiene la estructura correcta para zonas inversas.
- `/etc/bind/db.10.54.119` — nombre del archivo de zona inversa para nuestra red `10.54.119.x`.

Editamos el archivo:

```bash
sudo nano /etc/bind/db.10.54.119
```

![alt text](<Screenshot 2026-05-13 232119.png>)

El archivo quedó con el siguiente contenido:

```
$TTL    604800
@       IN      SOA     dns.sis313lab42jpfranc.local. admin.sis313lab42jpfranc.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.sis313lab42jpfranc.local.
100     IN      PTR     dns.sis313lab42jpfranc.local.
101     IN      PTR     www.sis313lab42jpfranc.local.
```

![alt text](<Screenshot 2026-05-13 232058.png>)

- `PTR` (Pointer Record) — registro de puntero, es el inverso del registro A. Dado el último octeto de una IP, devuelve el nombre de dominio completo.
- `100 IN PTR dns.sis313lab42jpfranc.local.` — la IP `10.54.119.100` resuelve al nombre `dns.sis313lab42jpfranc.local`. Solo se escribe el último octeto (`100`) porque los tres primeros octetos ya están definidos en el nombre de la zona (`119.54.10.in-addr.arpa`).
- `101 IN PTR www.sis313lab42jpfranc.local.` — la IP `10.54.119.101` resuelve al nombre `www.sis313lab42jpfranc.local`.

### 5.6 Verificación y Reinicio de BIND9

Verificamos que la sintaxis de todos los archivos de configuración de BIND9 sea correcta:

```bash
sudo named-checkconf
```

![alt text](<Screenshot 2026-05-13 232226.png>)

- `named-checkconf` — herramienta de BIND9 que verifica la sintaxis del archivo principal `named.conf` y todos los archivos que incluye. Si no muestra nada, significa que no hay errores.

Verificamos la validez de cada archivo de zona:

```bash
sudo named-checkzone sis313lab42jpfranc.local /etc/bind/db.sis313lab42jpfranc.local
sudo named-checkzone 119.54.10.in-addr.arpa /etc/bind/db.10.54.119
```

- `named-checkzone` — verifica la sintaxis y consistencia de un archivo de zona específico.
- El primer argumento es el nombre de la zona y el segundo es la ruta al archivo. Si todo está correcto muestra `OK` al final.

![alt text](<Screenshot 2026-05-13 232314.png>)

![alt text](<Screenshot 2026-05-13 232416.png>)

Reiniciamos el servicio BIND9 para que cargue los nuevos archivos de zona:

```bash
sudo systemctl restart bind9
```

![alt text](<Screenshot 2026-05-13 232603.png>)

Verificamos que el servicio esté corriendo correctamente:

```bash
sudo systemctl status bind9
```

![alt text](<Screenshot 2026-05-13 232817.png>)

Verificamos que BIND9 esté escuchando en el puerto 53:

```bash
sudo ss -tulnp | grep named
```

![alt text](<Screenshot 2026-05-13 232903.png>)


### 5.7 Deshabilitar systemd-resolved

Ubuntu 24.04 usa `systemd-resolved` por defecto como intermediario DNS del sistema. Este servicio tiene un comportamiento especial: intercepta todas las consultas DNS para dominios `.local` y las trata como Multicast DNS (mDNS), impidiendo que lleguen a BIND9. Por eso debemos deshabilitarlo.

Deshabilitamos el inicio automático del servicio:

```bash
sudo systemctl disable systemd-resolved
```

- `systemctl disable` — elimina los enlaces simbólicos que hacen que el servicio se inicie automáticamente al arrancar el sistema. El servicio seguirá corriendo hasta que lo detengamos o reiniciemos.

Detenemos el servicio inmediatamente:

```bash
sudo systemctl stop systemd-resolved
```

- `systemctl stop` — detiene el servicio de inmediato sin esperar al reinicio.

Eliminamos el archivo `resolv.conf` que era un enlace simbólico gestionado por `systemd-resolved`:

```bash
sudo rm /etc/resolv.conf
```

- `/etc/resolv.conf` — archivo que el sistema operativo usa para saber a qué servidor DNS consultar. Con `systemd-resolved` activo, era un enlace simbólico a un archivo temporal. Al eliminarlo podemos crear uno estático.

Creamos un nuevo `resolv.conf` estático apuntando directamente a nuestra VM DNS:

```bash
sudo bash -c 'echo "nameserver 10.54.119.100" > /etc/resolv.conf'
```

![alt text](<Screenshot 2026-05-13 233047.png>)

- `bash -c` — ejecuta el comando entre comillas en un subshell con privilegios de root (necesario para redirigir a un archivo del sistema).
- `echo "nameserver 10.54.119.100"` — escribe la línea de configuración.
- `> /etc/resolv.conf` — redirige la salida al archivo, creándolo o sobreescribiéndolo.
- `nameserver 10.54.119.100` — indica al sistema que use `10.54.119.100` (nuestra VM DNS) para resolver todos los nombres de dominio.

### 5.8 Configuración de UFW (Firewall)

Instalamos UFW (Uncomplicated Firewall), el firewall de Ubuntu:

```bash
sudo apt install ufw -y
```

![alt text](<Screenshot 2026-05-13 233156.png>)

- `ufw` — herramienta de firewall simplificada de Ubuntu que facilita la gestión de reglas iptables sin necesidad de conocer la sintaxis compleja de iptables directamente.

Permitimos consultas DNS desde la red del aula:

```bash
sudo ufw allow from 10.54.119.0/24 to any port 53
```

![alt text](<Screenshot 2026-05-13 233221.png>)

- `ufw allow` — agrega una regla que permite el tráfico.
- `from 10.54.119.0/24` — solo permite conexiones que provengan de la red `10.54.119.0/24` (la red del aula). Esto evita que equipos de otras redes puedan hacer consultas a nuestro DNS.
- `to any port 53` — permite el tráfico hacia el puerto 53 (puerto estándar de DNS), tanto TCP como UDP.

Permitimos conexiones SSH para no perder acceso remoto al activar el firewall:

```bash
sudo ufw allow 22
```

![alt text](<Screenshot 2026-05-13 233258.png>)

- `ufw allow 22` — permite conexiones TCP al puerto 22 (SSH) desde cualquier origen. Sin esta regla, al activar UFW perderíamos el acceso SSH a la VM.

Activamos el firewall:

```bash
sudo ufw enable
```
![alt text](<Screenshot 2026-05-13 233335.png>)

---

## 6. Configuración del Servidor Web (VM Lab4.2-Web)

### 6.1 Configuración de Red Estática

Editamos el archivo de configuración de red:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

El archivo quedó con el siguiente contenido:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [10.54.119.101/24]
      nameservers:
        addresses: [8.8.8.8, 10.54.119.100]
      routes:
        - to: default
          via: 10.54.119.76
```

![alt text](<Screenshot 2026-05-14 094055.png>)

- `addresses: [10.54.119.101/24]` — IP estática de la VM Web en la red del aula.
- `nameservers: addresses: [8.8.8.8, 10.54.119.100]` — configura dos servidores DNS: primero Google (8.8.8.8) para nombres externos, y luego nuestra VM DNS (10.54.119.100) para resolver el dominio `sis313lab42jpfranc.local`.
- `routes: via: 10.54.119.76` — el gateway del aula para salir a internet.

Aplicamos los cambios:

```bash
sudo netplan apply
```

### 6.2 Instalación de Nginx

Actualizamos los repositorios e instalamos Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

- `nginx` — servidor web de alto rendimiento y bajo consumo de memoria. Es uno de los servidores web más usados en el mundo, especialmente en entornos de producción.

Habilitamos Nginx para que inicie automáticamente y lo arrancamos:

```bash
sudo systemctl enable --now nginx
```

![alt text](<Screenshot 2026-05-14 094228.png>)

- `systemctl enable` — configura el servicio para que se inicie automáticamente en cada arranque del sistema.
- `--now` — además de configurarlo, lo inicia inmediatamente sin necesidad de ejecutar `systemctl start` por separado.

### 6.3 Configuración del Virtual Host

Creamos el archivo de configuración del Virtual Host para nuestro dominio:

```bash
sudo nano /etc/nginx/sites-available/sis313lab42jpfranc.local
```

![alt text](<Screenshot 2026-05-14 094350.png>)

- `/etc/nginx/sites-available/` — directorio donde se guardan las configuraciones de todos los sitios disponibles en Nginx (estén activos o no).
- Un Virtual Host le dice a Nginx cómo responder cuando recibe una petición para un dominio específico.

El archivo quedó con el siguiente contenido:

```nginx
server {
    listen 80;
    server_name www.sis313lab42jpfranc.local sis313lab42jpfranc.local;

    root /var/www/sis313lab42jpfranc.local;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

![alt text](<Screenshot 2026-05-14 094425.png>)

- `listen 80` — Nginx escuchará conexiones en el puerto 80 (HTTP estándar).
- `server_name` — lista de nombres de dominio para los que este bloque responderá. Cuando Nginx recibe una petición, compara el dominio de la petición con este campo para decidir qué bloque usar.
- `root /var/www/sis313lab42jpfranc.local` — directorio raíz del sitio web. Nginx buscará los archivos HTML aquí.
- `index index.html` — archivo que se cargará cuando se acceda a la raíz del sitio (sin especificar un archivo concreto).
- `location /` — bloque que define cómo manejar todas las peticiones a cualquier ruta del sitio.
- `try_files $uri $uri/ =404` — intenta encontrar el archivo pedido, si no existe devuelve error 404. `$uri` es la ruta pedida por el cliente.

### 6.4 Creación del Sitio Web

Creamos el directorio donde vivirán los archivos del sitio:

```bash
sudo mkdir -p /var/www/sis313lab42jpfranc.local
```

![alt text](<Screenshot 2026-05-14 094452.png>)

Activamos el sitio creando un enlace simbólico en `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/sis313lab42jpfranc.local /etc/nginx/sites-enabled/
```

![alt text](<Screenshot 2026-05-14 094530.png>)

- `ln -s` — crea un enlace simbólico (acceso directo). Nginx lee los sitios activos desde `/etc/nginx/sites-enabled/`. En lugar de copiar el archivo, creamos un enlace que apunta al original en `sites-available/`. Así si modificamos el archivo en `sites-available/`, el cambio se refleja automáticamente.

Desactivamos el sitio por defecto de Nginx para evitar conflictos:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

![alt text](<Screenshot 2026-05-14 094608.png>)

- Elimina el enlace simbólico del sitio por defecto de Nginx. Sin esto, Nginx podría mostrar su página de bienvenida en lugar de nuestro sitio cuando se accede por IP o dominio no configurado.

Creamos la página HTML del sitio:

```bash
sudo nano /var/www/sis313lab42jpfranc.local/index.html
```

![alt text](<Screenshot 2026-05-14 094646.png>)

El contenido del archivo:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>SIS313 Lab 4.2</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #1a1a2e, #16213e, #0f3460);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
        }
        .card {
            background: rgba(255,255,255,0.1);
            border-radius: 20px;
            padding: 50px;
            text-align: center;
            color: white;
            backdrop-filter: blur(10px);
        }
        h1 { color: #e94560; margin-bottom: 20px; }
        .badge {
            background: #e94560;
            border-radius: 10px;
            padding: 10px 20px;
            margin: 10px;
            display: inline-block;
            font-weight: bold;
        }
        .info { margin-top: 30px; color: #a8b2d8; }
    </style>
</head>
<body>
    <div class="card">
        <h1>🌐 SIS313 - Lab 4.2</h1>
        <h2>Servidor DNS Primario</h2>
        <br>
        <div class="badge">👤 JP</div>
        <div class="badge">👤 Franco</div>
        <br><br>
        <p>Dominio: <strong>sis313lab42jpfranc.local</strong></p>
        <div class="info">
            <p>🖥️ Servidor Web: 10.54.119.101</p>
            <p>🔧 Servidor DNS: 10.54.119.100</p>
            <p>📚 Universidad San Francisco Xavier</p>
        </div>
    </div>
</body>
</html>
```

![alt text](<Screenshot 2026-05-14 094721.png>)

Verificamos que la sintaxis de la configuración de Nginx sea correcta:

```bash
sudo nginx -t
```

![alt text](<Screenshot 2026-05-14 094800.png>)

Debe mostrar `syntax is ok` y `test is successful`.

Reiniciamos Nginx para que cargue la nueva configuración:

```bash
sudo systemctl restart nginx
```

![alt text](<Screenshot 2026-05-14 094825.png>)

---

## 7. Flujo de Resolución DNS

```
PC Cliente          VM DNS (BIND9)          VM Web (Nginx)
    │                     │                       │
    │  1. ¿www.sis313...? │                       │
    │────────────────────►│                       │
    │                     │  2. Busca en zona     │
    │                     │     directa           │
    │                     │◄──────────────────    │
    │  3. → 10.54.119.101 │                       │
    │◄────────────────────│                       │
    │                     │                       │
    │  4. HTTP GET /       │                       │
    │──────────────────────────────────────────►  │
    │                     │                       │
    │  5. Responde HTML    │                       │
    │◄──────────────────────────────────────────  │
```

---

## 9. Configuración del Firewall UFW

### 9.1 Tabla de políticas UFW

| Regla | Protocolo | Origen | Puerto | Acción |
|---|---|---|---|---|
| SSH | TCP | Cualquiera | 22 | ALLOW |
| DNS | UDP/TCP | `10.54.119.0/24` | 53 | ALLOW |

---

## 10. Verificación y Pruebas

### 10.1 Verificar que BIND9 escucha en el puerto 53

```bash
sudo ss -tulnp | grep named
```

El resultado muestra que BIND9 (`named`) está escuchando en el puerto 53 tanto en UDP como en TCP sobre la IP `10.54.119.100`, confirmando que el servidor DNS está listo para recibir consultas.

### 10.2 Resolución directa (Zona Directa)

```bash
dig @10.54.119.100 www.sis313lab42jpfranc.local
```

- `dig` — herramienta para hacer consultas DNS manualmente.
- `@10.54.119.100` — indica a qué servidor DNS enviar la consulta (nuestra VM DNS).
- `www.sis313lab42jpfranc.local` — nombre que queremos resolver.

```bash
nslookup www.sis313lab42jpfranc.local
```

- `nslookup` — herramienta alternativa para consultas DNS. Usa el servidor DNS configurado en el sistema (`/etc/resolv.conf`).

Resultado obtenido:

```
www.sis313lab42jpfranc.local. 604800 IN A 10.54.119.101
status: NOERROR
```

### 10.3 Resolución inversa (Zona Inversa)

```bash
dig -x 10.54.119.101
```

- `-x` — indica que se trata de una consulta inversa. `dig` construye automáticamente la consulta PTR para `101.119.54.10.in-addr.arpa`.

```bash
nslookup 10.54.119.101
```

Resultado obtenido:

```
101.119.54.10.in-addr.arpa. 604800 IN PTR www.sis313lab42jpfranc.local.
name = www.sis313lab42jpfranc.local.
```

### 10.4 Acceso web por nombre de dominio

Desde la VM DNS verificamos que el sitio responde por nombre:

```bash
curl http://www.sis313lab42jpfranc.local
```

![alt text](<Screenshot 2026-05-14 093708.png>)

Resultado: se muestra el HTML completo del sitio web servido desde `10.54.119.101`.

### 10.5 Acceso desde PC Anfitriona (Windows)

Se configuró el DNS preferido en Windows apuntando a `10.54.119.100`:

```
Panel de Control → Red → Propiedades → IPv4 → DNS preferido: 10.54.119.100
```

![alt text](<Screenshot 2026-05-13 234358.png>)

Acceso desde el navegador:

```
http://www.sis313lab42jpfranc.local
```

![alt text](<Screenshot 2026-05-14 093931.png>)

El navegador resuelve el nombre usando nuestra VM DNS y carga correctamente el sitio web desde la VM Web.

---


## 11. Conclusiones

- La implementación de BIND9 como servidor DNS primario permite gestionar la resolución de nombres dentro de una red local, eliminando la necesidad de recordar direcciones IP para acceder a los servicios.

- La zona directa (`sis313lab42jpfranc.local`) traduce nombres de dominio a IPs, mientras que la zona inversa (`119.54.10.in-addr.arpa`) hace lo contrario, siendo útil para auditorías y diagnóstico de red.

- Ubuntu 24.04 incluye `systemd-resolved` por defecto, el cual intercepta consultas `.local` antes de que lleguen a BIND9. Deshabilitarlo fue necesario para que el servidor DNS funcione correctamente con dominios `.local`.

- El modo Puente (Bridge) en VirtualBox permite que las VMs sean visibles en la red física del aula como equipos reales, lo cual es fundamental para que otros equipos puedan hacer consultas al servidor DNS.

- Nginx con Virtual Hosts permite servir diferentes sitios web desde el mismo servidor, identificando por el nombre del dominio en la petición HTTP cuál sitio debe mostrar.

- UFW protege el servidor DNS permitiendo solo las consultas desde la red del aula (puerto 53) y acceso SSH para administración (puerto 22), bloqueando cualquier otro tráfico no autorizado.

- La integración DNS + Web demuestra cómo funciona la infraestructura básica de internet: primero se resuelve el nombre a una IP mediante DNS, y luego se establece la conexión HTTP con el servidor web en esa IP.

---
