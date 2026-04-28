# Informe de Laboratorio 3.2
## Infraestructura de Red de una Organización con VLANs

---

**Universidad Mayor Real y Ponitificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnologia**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiantes:**                      **Carrera:**

- Juan Pablo Taboada Camacho          Ing. Sistemas
- Franco Milton Jimenez Amachuy       Ing. Sistemas

---

## 1. Introducción

El presente laboratorio tiene como objetivo implementar una infraestructura de red segmentada mediante VLANs (Virtual Local Area Networks) en un entorno virtualizado con VirtualBox. Se configuró una maquina (representara al router) con Ubuntu Server 24.04 que actúa como gateway y firewall para cuatro segmentos de red distintos, aplicando políticas de acceso según el rol de cada área de la organización.

---

## 2. Objetivos del Laboratorio

- **Diseñar e implementar una arquitectura de red empresarial con VLANs** para segmentar los departamentos de una organización.

- **Configurar un router con Linux** para que gestione el enrutamiento inter-VLAN, el acceso a internet y las políticas de seguridad.

- **Aplicar reglas de firewall (UFW)** para controlar el flujo de tráfico entre las diferentes VLANs y la red externa.

- **Comprender y configurar interfaces** `trunk` y etiquetado de VLANs en máquinas virtuales.

- **Demostrar el funcionamiento de las políticas de acceso** establecidas entre los diferentes departamentos.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
Internet (NAT)
      |
   [Router - Ubuntu Server]
   enp0s3 (NAT) | enp0s8 (trunk VLAN)
      |
   [Switch Virtual - VirtualBox Red Interna]
      |
  ┌───┼───┬───────┐
  |   |   |       |
VLAN10 VLAN20 VLAN30 VLAN40
 DMZ   TI  Ventas  Contab
```

### 3.2 Esquema de la infraestructura

| Máquina Virtual | Departamento | VLAN ID | Subred |
| ----------------|--------------|---------|--------|
| `Router`        | -            | -       | 192.168.10.1/29 <br> 192.168.20.1/29 <br> 192.168.30.1/27 <br> 192.168.40.1/29 |
| `Server-DMZ1`   | DMZ          | 10      | 192.168.10.2/29 |
| `Server-DMZ2`	  | DMZ	         | 10	   | 192.168.10.3/29 |
| `PC-TI`         | TI           | 20      | 192.168.20.2/29 |
| `PC-Ventas`     | Ventas       | 30      | 192.168.30.2/27 |
| `PC-Contabilidad`| Contabilidad       | 40      | 192.168.40.2/29 |

### 3.2 Tabla de Direccionamiento

| VLAN | Nombre | Red | Máscara | Gateway | Hosts |
|------|--------|-----|---------|---------|-------|
| 10 | DMZ | 192.168.10.0/29 | 255.255.255.248 | 192.168.10.1 | Server-DMZ1 (.2), Server-DMZ2 (.3) |
| 20 | TI | 192.168.20.0/29 | 255.255.255.248 | 192.168.20.1 | PC-TI (.2) |
| 30 | Ventas | 192.168.30.0/27 | 255.255.255.224 | 192.168.30.1 | PC-Ventas (.2) |
| 40 | Contabilidad | 192.168.40.0/29 | 255.255.255.248 | 192.168.40.1 | PC-Contabilidad (.2) |

---

## 4. Preparacion del entorno

1. **Máquina Virtual: Router (Ubuntu Server 24.04)**

    - **Interfaz 1 (`enp0s3`)**: NAT. Se conecta a la red del anfitrión para acceso a internet.

    - **Interfaz 2 (`enp0s8`)**: Red Interna (Tipo "Red Interna"). Esta interfaz se configurará como un `trunk` para transportar las VLANs.

2. **Máquinas Virtuales: PC TI, PC Ventas, PC Contabilidad, Servers DMZ (Alpine Linux)**

    - Cada una de estas VMs tendrá una única interfaz de red conectada a la misma "Red Interna" que la interfaz `enp0s8` del Router.

    - **Importante:** En la configuración de la red del hipervisor, la "Red Interna" esta configurada como un switch que permite el paso de tramas etiquetadas (VLAN-aware).

---

## 5. Configuración de VLANs y Router

### 5.1 Configuracion del router

---

- Instalamos el servicio VLAN

```bash
sudo apt install vlan -y
```
- Cargamos el modulo de kernel para VLANs

```bash
sudo modprobe 8021q
```
- Configuramos las interfaces de de red VLAN. Para el enrutamiento inter-VLAN, cada sub-interfaz tendrá un gateway que será su propia dirección IP. Entramos al archivo /etc/netplan/50-cloud-init.yaml

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Archivo: `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      optional: true
  vlans:
    vlan10:
      link: enp0s8
      id: 10
      addresses: [192.168.10.1/29]
      nameservers:
        addresses: [8.8.8.8]
    vlan20:
      link: enp0s8
      id: 20
      addresses: [192.168.20.1/29]
      nameservers:
        addresses: [8.8.8.8]
    vlan30:
      link: enp0s8
      id: 30
      addresses: [192.168.30.1/27]
      nameservers:
        addresses: [8.8.8.8]
    vlan40:
      link: enp0s8
      id: 40
      addresses: [192.168.40.1/29]
      nameservers:
        addresses: [8.8.8.8]
```

- Aplicamos la configuracion que hicimos 

```bash
sudo netplan apply
```

- Habilitamos el reenvio de paquetes en el kernel

  Editamos el archivo `sysctl.conf`

```bash
sudo nano /etc/sysctl.conf
```
- El archivo debe estar asi (esto habilitara el reenvio de paquetes IP)

```bash
#
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

#kernel.domainname = example.com

# Uncomment the following to stop low-level messages on console
#kernel.printk = 3 4 1 3

###############################################################
# Functions previously found in netbase
#

# Uncomment the next two lines to enable Spoof protection (reverse-path filter)
# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks
#net.ipv4.conf.default.rp_filter=1
#net.ipv4.conf.all.rp_filter=1

# Uncomment the next line to enable TCP/IP SYN cookies
# See http://lwn.net/Articles/277146/
# Note: This may impact IPv6 TCP sessions too
#net.ipv4.tcp_syncookies=1

# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
#net.ipv6.conf.all.forwarding=1


###############################################################
# Additional settings - these settings can improve the network
# security of the host and prevent against some network attacks
# including spoofing attacks and man in the middle attacks through
# redirection. Some network environments, however, require that these
# settings are disabled so review and enable them as needed.
#
# Do not accept ICMP redirects (prevent MITM attacks)
#net.ipv4.conf.all.accept_redirects = 0
#net.ipv4.conf.default.accept_redirects = 0
# _or_
# Accept ICMP redirects only for gateways listed in our default
# gateway list (enabled by default)
# net.ipv4.conf.all.secure_redirects = 1
#
# Do not send ICMP redirects (we are not a router)
#net.ipv4.conf.all.send_redirects = 0
#
# Log Martian Packets
#net.ipv4.conf.all.log_martians = 1
#

###############################################################
# Magic system request Key
# 0=disable, 1=enable all, >1 bitmask of sysrq functions
# See https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html
# for what other values do
#kernel.sysrq=438
```

- Aplicamos los cambios

```bash 
sudo sysctl -p
```

---

### 5.2 Configuracion del servidor DMZ1

---

- Una vez instalado el Alpine Linux necesitamos instalar la herramienta nano para editar archivos, y la herramienta vlan

```bash
apk update
apk add nano
apk vlan
```

- Instalamos las herramientas cuando aun la maquina tenia como adaptador tipo NAT. Apagamos la maquina configuramos el adaptador a "Red Interna" y volvemos a encender la maquina.

- Una vez la maquina encendida configuramos las interfaces entonces entramos al archivo /etc/network/interfaces

```bash 
nano /etc/network/interfaces
```
- Dentro cambiaremos la configuracion a la siguiente:

```nano
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto eth0.10
iface eth0.10 inet static
    address 192.168.10.2
    netmask 255.255.255.248
    gateway 192.168.10.1
    vlan-id 10
```

- Aqui estamos asignando la IP que tendra el servidor DMZ1 (`192.168.10.2`), con mascara `/29`.

- Tambien configuramos el gateaway con la IP del router en esa VLAN (`192.168.10.1`).

- Entonces reiniciaremos el servicio para que se apliquen los cambios que hicimos

```bash
rc-service networking restart
```

- Realizando pruebas se tiene conectividad con ping hacia el router (`192.168.10.1`), pero aun no tiene acceso a internet (`google.com`).

---

### 5.3 Configuracion del servidor DMZ2

---

- Aplicaremos lo mismo que hicimos con el servidor DMZ1 para instalar las herramientas nano y vlan

```bash
apk update
apk add nano
apk add vlan
```

- Cambiamos la configuracion del adaptador de NAT a Red Interna

- Configuramos las interfaces entrando a /etc/network/interfaces

```bash
nano /etc/network/interfaces
```

- Cambiamos la configuracion

```nano
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto eth0.10
iface eth0.10 inet static
    address 192.168.10.3
    netmask 255.255.255.248
    gateway 192.168.10.1
    vlan-id 10
```

- Estamos asignando la IP que tendra el servidor DMZ1 (`192.168.10.3`), con mascara `/29`.

- Tambien configuramos el gateaway con la IP del router en esa VLAN (`192.168.10.1`).

- Reiniciaremos el servicio para que se apliquen los cambios que hicimos

```bash
rc-service networking restart
```

- Realizando pruebas se tiene conectividad con ping hacia el router (`192.168.10.1`), pero aun no tiene acceso a internet (`google.com`).

---

### 5.4 Configuracion de la PC-TI

---

- Aplicamos lo mismo que hicimos con las maquinas anteriores

```bash
apk update
apk add nano
apk add vlan
```

- Cambiamos la configuracion del adaptador de NAT a Red Interna

- Configuramos las interfaces entrando a /etc/network/interfaces

```bash
nano /etc/network/interfaces
```

- Cambiamos la configuracion

```nano
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto eth0.20
iface eth0.20 inet static
    address 192.168.20.2
    netmask 255.255.255.248
    gateway 192.168.20.1
    vlan-id 20
```

- Estamos asignando la IP que tendra el servidor DMZ1 (`192.168.20.2`), con mascara `/29`.

- Tambien configuramos el gateaway con la IP del router en esa VLAN (`192.168.20.1`).

- Reiniciaremos el servicio para que se apliquen los cambios que hicimos

```bash
rc-service networking restart
```

- Realizando pruebas se tiene conectividad con ping hacia el router (`192.168.20.1`), pero aun no tiene acceso a internet (`google.com`).

---

### 5.5 Configuracion de la PC-VENTAS

---

- Aplicamos lo mismo que hicimos con las maquinas anteriores

```bash
apk update
apk add nano
apk add vlan
```

- Cambiamos la configuracion del adaptador de NAT a Red Interna

- Configuramos las interfaces entrando a /etc/network/interfaces

```bash
nano /etc/network/interfaces
```

- Cambiamos la configuracion

```nano
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto eth0.30
iface eth0.30 inet static
    address 192.168.30.2
    netmask 255.255.255.224
    gateway 192.168.30.1
    vlan-id 30
```

- Estamos asignando la IP que tendra el servidor DMZ1 (`192.168.30.2`), con mascara `/27`.

- Tambien configuramos el gateaway con la IP del router en esa VLAN (`192.168.30.1`).

- Reiniciaremos el servicio para que se apliquen los cambios que hicimos

```bash
rc-service networking restart
```

- Realizando pruebas se tiene conectividad con ping hacia el router (`192.168.30.1`), pero aun no tiene acceso a internet (`google.com`).

---

### 5.6 Configuracion de la PC-CONTABILIDAD

---

- Aplicamos lo mismo que hicimos con las maquinas anteriores

```bash
apk update
apk add nano
apk add vlan
```

- Cambiamos la configuracion del adaptador de NAT a Red Interna

- Configuramos las interfaces entrando a /etc/network/interfaces

```bash
nano /etc/network/interfaces
```

- Cambiamos la configuracion

```nano
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto eth0.40
iface eth0.40 inet static
    address 192.168.40.2
    netmask 255.255.255.248
    gateway 192.168.40.1
    vlan-id 40
```

- Estamos asignando la IP que tendra el servidor DMZ1 (`192.168.40.2`), con mascara `/29`.

- Tambien configuramos el gateaway con la IP del router en esa VLAN (`192.168.40.1`).

- Reiniciaremos el servicio para que se apliquen los cambios que hicimos

```bash
rc-service networking restart
```

- Realizando pruebas se tiene conectividad con ping hacia el router (`192.168.40.1`), pero aun no tiene acceso a internet (`google.com`).

---

## 6. Configuración del Firewall UFW

---

- **En la VM Router (Ubuntu):**

- Instalamos ufw

```bash
sudo apt install ufw -y
```

- Habilitamos ssh en ufw

```bash
sudo ufw allow ssh
```

- Habilitamos ufw desde el arranque

```bash
sudo ufw enable
```

- Configuramos las politicas para controlar el trafico entre VLANs


- En `/etc/default/ufw` se mantuvo:

```
DEFAULT_FORWARD_POLICY="DROP"
```

- Esto asegura que todo el tráfico entre VLANs esté bloqueado por defecto, y solo se permite lo explícitamente declarado.

- **Reglas de acceso entre VLANs**

- La VLAN de TI (VLAN 20) puede acceder a todos y a internet

```bash
sudo ufw route allow in on vlan20 out on vlan10
sudo ufw route allow in on vlan20 out on vlan30
sudo ufw route allow in on vlan20 out on vlan40
sudo ufw route allow in on vlan20 out on enp0s3
```

- La VLAN de VENTAS (VLAN 30) solo puede acceder a los servidores DMZ (VLAN 10)

```bash
sudo ufw route allow in on vlan30 out on vlan10
```

- La VLAN de CONTABILIDAD (VLAN 40) puede acceder a los servidores DMZ (VLAN 10), a ventas y a Internet

```bash
sudo ufw route allow in on vlan40 out on vlan10
sudo ufw route allow in on vlan40 out on vlan30
sudo ufw route allow in on vlan40 out on enp0s3
```

- Para la VLAN de los servidores DMZ (VLAN 10) bloqueamos el acceso a todos los demas

```bash 
sudo ufw route deny in on vlan10 out on vlan20
sudo ufw route deny in on vlan10 out on vlan30
sudo ufw route deny in on vlan10 out on vlan40
```

- Para la VLAN de VENTAS (VLAN 30) bloqueamos el acceso a TI y CONTABILIDAD

```bash
sudo ufw route deny in on vlan30 out on vlan20
sudo ufw route deny in on vlan30 out on vlan40
```

- Para la VLAN de CONTABILIDAD (VLAN 40) bloqueamos el acceso a TI

```bash 
sudo ufw route deny in on vlan40 out on vlan20
```

- Configuramos el NAT para el acceso a Internet desde TI y CONTABILIDAD

- Entramos al rchivo: `/etc/ufw/before.rules` 

```bash 
sudo nano /etc/ufw/before.rules
```

- Al inicio de archivo anadimos

```nano
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.20.0/29 -o enp0s3 -j MASQUERADE
-A POSTROUTING -s 192.168.40.0/29 -o enp0s3 -j MASQUERADE
COMMIT
```

- El archivo quedaria asi:

```bash
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.20.0/29 -o enp0s3 -j MASQUERADE
-A POSTROUTING -s 192.168.40.0/29 -o enp0s3 -j MASQUERADE
COMMIT
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# Don't delete these required lines, otherwise there will be errors
*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]
# End required lines

# allow all on loopback
-A ufw-before-input -i lo -j ACCEPT
-A ufw-before-output -o lo -j ACCEPT

# quickly process packets for which we already have a connection
-A ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-output -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-forward -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# drop INVALID packets (logs these in loglevel medium and higher)
-A ufw-before-input -m conntrack --ctstate INVALID -j ufw-logging-deny
-A ufw-before-input -m conntrack --ctstate INVALID -j DROP

# ok icmp codes for INPUT
-A ufw-before-input -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-input -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-input -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-input -p icmp --icmp-type echo-request -j ACCEPT

# ok icmp code for FORWARD
-A ufw-before-forward -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type echo-request -j ACCEPT

# allow dhcp client to work
-A ufw-before-input -p udp --sport 67 --dport 68 -j ACCEPT

#
# ufw-not-local
#
-A ufw-before-input -j ufw-not-local

# if LOCAL, RETURN
-A ufw-not-local -m addrtype --dst-type LOCAL -j RETURN

# if MULTICAST, RETURN
-A ufw-not-local -m addrtype --dst-type MULTICAST -j RETURN

# if BROADCAST, RETURN
-A ufw-not-local -m addrtype --dst-type BROADCAST -j RETURN

# all other non-local packets are dropped
-A ufw-not-local -m limit --limit 3/min --limit-burst 10 -j ufw-logging-deny
-A ufw-not-local -j DROP

# allow MULTICAST mDNS for service discovery (be sure the MULTICAST line above
# is uncommented)
-A ufw-before-input -p udp -d 224.0.0.251 --dport 5353 -j ACCEPT

# allow MULTICAST UPnP for service discovery (be sure the MULTICAST line above
# is uncommented)
-A ufw-before-input -p udp -d 239.255.255.250 --dport 1900 -j ACCEPT

# don't delete the 'COMMIT' line or these rules won't be processed
COMMIT
```

- Reiniciamos ufw 

```bash 
sudo systemctl restart ufw
```

- Para verificar la configuracion del NAT que pusimos

```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

- Vimos que la configuracion esta bien

--- 

## 7. Tabla de Políticas de Acceso

---

| Origen | DMZ | TI | Ventas | Contab | Internet |
|--------|-----|----|--------|--------|----------|
| **TI** | ✅ | - | ✅ | ✅ | ✅ |
| **Ventas** | ✅ | ❌ | - | ❌ | ❌ |
| **Contabilidad** | ✅ | ❌ | ✅ | - | ✅ |
| **DMZ** | - | ❌ | ❌ | ❌ | ❌ |

---

## 8. Verificación y Pruebas

--- 

- Verificar interfaces en el Router
```bash
ip addr show vlan10
ip addr show vlan20
ip addr show vlan30
ip addr show vlan40
```

- Verificar NAT aplicado

```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

- Verificar reglas UFW

```bash
sudo ufw status verbose
```

- Pruebas de conectividad 

```bash
ping 192.168.10.2
ping 192.168.10.3  
ping 192.168.20.2   
ping 192.168.30.2
ping 192.168.40.2   
ping google.com     
```

- Pruebas de acceso mediante ssh entre VLANs

  - Desde los servidores DMZ

```bash 
ssh root@192.168.20.2
ssh root@192.168.30.2
ssh root@192.168.40.2
```

  - Desde la PC-TI

```bash 
ssh root@192.168.10.2
ssh root@192.168.10.3
ssh root@192.168.30.2
ssh root@192.168.40.2
```

  - Desde la PC-VENTAS

```bash 
ssh root@192.168.10.2
ssh root@192.168.10.3
ssh root@192.168.20.2
ssh root@192.168.40.2
```

  - Desde la PC-CONTABILIDAD

```bash
ssh root@192.168.10.2
ssh root@192.168.10.3
ssh root@192.168.20.2
ssh root@192.168.30.2
```

---

## 9. Conclusiones

---

- La segmentación mediante VLANs permite aislar el tráfico entre departamentos de una organización, mejorando la seguridad y el control del acceso.
- UFW con política `DEFAULT_FORWARD_POLICY="DROP"` es una estrategia más segura que permitir todo y luego bloquear, ya que por defecto niega todo tráfico no autorizado.
- Podriamos configurar UFW (`/etc/default/ufw`) con politica `DEFAULT_FORWARD_POLICY="ACCEPT", todo el tráfico entre VLANs estaría permitido por defecto, incluyendo ping y SSH.
La diferencia sería:
Con ACCEPT (lo que no hicimos):

Todo el tráfico entre VLANs pasa libremente
Las reglas deny que pusimos bloquearían solo lo específico
Pero el problema es que si olvidamos poner algún deny, el tráfico pasa igual
Es menos seguro

Con DROP (lo que hicimos):

Todo el tráfico entre VLANs está bloqueado por defecto
Solo pasa lo que explícitamente permitimos con allow
Si olvidamos poner alguna regla, el tráfico se bloquea automáticamente
Es más seguro

En el caso de ACCEPT, las reglas deny que pusimos sí funcionarían para bloquear SSH, pero cualquier protocolo que no hayamos bloqueado explícitamente pasaría libremente entre todas las VLANs.
Con DROP en cambio, solo SSH, ping (por el before.rules) y los protocolos de las rutas que permitimos con allow pueden circular.
- El NAT permite que solo los segmentos autorizados (TI y Contabilidad) tengan acceso a internet, manteniendo aislados a DMZ y Ventas.
- VirtualBox con Red Interna actúa como switch virtual, permitiendo que las tramas etiquetadas con VLAN sean correctamente gestionadas por el Router.
- Se cumplen todas las reglas de acceso entre VLANS.
- Aun asi hayamos bloqueado el acceso, solo bloqueamos el acceso por ssh, pero sigue existiendo comunicacion entre las VLANs por medio de ping, esto es porque en las reglas que se puso solo bloqueamos el acceso por ssh, pero el ping (ICMP) no la bloqueamos. Podriamos establecer las reglas de la siguiente forma para que se bloquee el ping segun las reglas

```bash
# DMZ no puede hacer ping a nadie
sudo ufw route deny in on vlan10 out on vlan20 proto icmp
sudo ufw route deny in on vlan10 out on vlan30 proto icmp
sudo ufw route deny in on vlan10 out on vlan40 proto icmp
```

- Otra forma de bloquear el ping es cambiando el archivo `/etc/ufw/before.rules`, la linea que indica:
```bash
-A ufw-before-forward -p icmp --icmp-type echo-request -j ACCEPT
```
Permite el ping (echo-request) en todo el tráfico forward antes de que se evalúen tus reglas de deny.
Para bloquearlo podriamos comentar esa linea

```bash 
#-A ufw-before-forward -p icmp --icmp-type echo-request -j ACCEPT
```
De esta forma estariamos bloqueando el ping pero en todas las rutas

---
