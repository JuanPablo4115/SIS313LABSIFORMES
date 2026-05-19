# Informe de Laboratorio 5.1
## Hardening Integral y Seguridad TLS

---

**Universidad Mayor Real y Pontificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnología**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiantes:**                                             **Carrera:**

- Juan Pablo Taboada Camacho (SERVIDOR WEB)                  Ing. Sistemas
- Franco Milton Jiménez Amachuy (SERVIDOR DB)                Ing. Sistemas

---

## 1. Introducción

El presente laboratorio tiene como objetivo implementar un sistema de seguridad integral en un entorno virtualizado con VirtualBox, aplicando técnicas de hardening (endurecimiento de seguridad) en servidores Ubuntu Server 24.04 LTS. Se configuraron dos máquinas virtuales conectadas en modo Puente (Bridge) a la red del aula: una actuando como servidor web con Nginx protegido con TLS, y otra como servidor de base de datos con MariaDB.

El concepto de **hardening** hace referencia al proceso de reducir la superficie de ataque de un sistema mediante la desactivación de servicios innecesarios, el ajuste de configuraciones por defecto inseguras, y la aplicación de restricciones de acceso. En lugar de depender de una sola medida de seguridad, se aplica el principio de **defensa en profundidad**: múltiples capas de protección que garantizan que si una falla, las demás continúan protegiendo el sistema.

Se aplicaron las siguientes capas de seguridad:

| Capa | Tecnología | Descripción |
|---|---|---|
| Autenticación | SSH + Clave pública | Acceso solo con clave criptográfica |
| Red | UFW Firewall | Bloqueo de todo el tráfico no autorizado |
| Sistema | sysctl (Kernel) | Parámetros de seguridad del kernel Linux |
| Transporte | TLS 1.2/1.3 | Cifrado de comunicaciones web |
| Aplicación | Cabeceras HTTP | Protección contra ataques del navegador |
| Base de datos | MariaDB hardened | Acceso restringido por IP |
| DNS | BIND9 | Resolución del dominio `jpfra.local` |

---

## 2. Objetivos del Laboratorio

- **Configurar hardening SSH** deshabilitando autenticación por contraseña y usando exclusivamente clave pública, cambiando el puerto por defecto al 2222.

- **Implementar firewall UFW** con política de denegación por defecto, permitiendo únicamente los servicios necesarios en cada servidor.

- **Aplicar endurecimiento del kernel** mediante parámetros sysctl para prevenir ataques de red como IP spoofing.

- **Instalar y configurar Nginx con HTTPS**, generando un certificado SSL autofirmado y habilitando únicamente TLS 1.2 y TLS 1.3.

- **Configurar cabeceras de seguridad HTTP** (HSTS, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection).

- **Instalar y asegurar MariaDB**, restringiendo el acceso a la base de datos solo desde el servidor web.

- **Configurar un servidor DNS con BIND9** para resolver el dominio `jpfra.local` dentro de la red del aula.

- **Realizar pruebas de verificación** de cada medida de seguridad implementada.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
                         INTERNET
                             │
                    ┌────────▼────────┐
                    │    Gateway      │
                    │  10.106.126.1   │
                    └────────┬────────┘
                             │
              ┌────────────────────────────┐
              │      Red del Aula          │
              │    10.106.126.0/24         │
              └──────────┬─────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
    ┌─────────▼──────────┐  ┌───────▼─────────────┐
    │     VM WEBB        │  │      VM BD          │
    │  10.106.126.200    │  │  10.106.126.201     │
    │                    │  │                     │
    │  • Nginx (80/443)  │  │  • MariaDB (3306)   │
    │  • BIND9 (53)      │──│  • Solo acepta      │
    │  • SSH (2222)      │  │    .200 en 3306     │
    │  • UFW activo      │  │  • SSH (2222)       │
    │  • TLS 1.2/1.3     │  │  • UFW activo       │
    │  • Cert. SSL       │  │                     │
    └────────────────────┘  └─────────────────────┘
              │
    ┌─────────▼──────────┐
    │   PC Anfitriona    │
    │     (Windows)      │
    │  DNS → .200        │
    │  Acceso HTTPS      │
    └────────────────────┘
```

### 3.2 Tabla de Direccionamiento

| Máquina Virtual | Rol | Dirección IP | Máscara | Gateway | Adaptador VirtualBox |
|---|---|---|---|---|---|
| `Lab5.1-WEBB` | Servidor Web + DNS | `10.106.126.200` | `/24` | `10.106.126.1` | Puente (Bridge) |
| `Lab5.1-BD` | Servidor Base de Datos | `10.106.126.201` | `/24` | `10.106.126.1` | Puente (Bridge) |
| PC Anfitriona | Cliente | DHCP | `/24` | `10.106.126.1` | - |

### 3.3 Tabla de Servicios y Puertos

| Servicio | VM | Puerto | Protocolo | Acceso permitido desde |
|---|---|---|---|---|
| SSH | Web | 2222 | TCP | Cualquier IP |
| HTTP | Web | 80 | TCP | Cualquier IP (redirige a HTTPS) |
| HTTPS | Web | 443 | TCP | Cualquier IP |
| DNS | Web | 53 | TCP/UDP | Red del aula `10.106.126.0/24` |
| SSH | DB | 2222 | TCP | Red del aula `10.106.126.0/24` |
| MariaDB | DB | 3306 | TCP | Solo VM Web `10.106.126.200` |

### 3.4 Configuración de Adaptadores en VirtualBox

#### VM Lab5.1-WEBB

```
┌─────────────────────────────────────────────────────┐
│         VirtualBox — VM Lab5.1-WEBB                 │
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

#### VM Lab5.1-BD

```
┌─────────────────────────────────────────────────────┐
│         VirtualBox — VM Lab5.1-BD                   │
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

El modo **Puente (Bridge)** permite que cada VM obtenga una IP directamente en la red física del aula, siendo visible para todos los equipos como si fuera un equipo físico real. A diferencia del modo NAT donde la VM queda oculta detrás de la IP del anfitrión.

### 3.5 Diagrama de Capas de Seguridad

```
┌─────────────────────────────────────────────────────────┐
│              CAPAS DE SEGURIDAD IMPLEMENTADAS           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  CAPA 7 - APLICACIÓN                                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Cabeceras HTTP: HSTS, X-Frame, X-XSS, nosniff  │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  CAPA 6 - PRESENTACIÓN                                  │
│  ┌─────────────────────────────────────────────────┐    │
│  │  TLS 1.2 / TLS 1.3 — Cifrado de comunicación    │    │
│  │  Certificado SSL autofirmado (CN=jpfra.local)   │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  CAPA 4 - TRANSPORTE                                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  UFW Firewall — deny by default                 │    │
│  │  Solo puertos 80, 443, 2222, 53 abiertos        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  CAPA 3 - RED                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Kernel sysctl — rp_filter anti-spoofing        │    │
│  │  ip_forward controlado por rol del servidor     │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  CAPA 1 - ACCESO                                        │
│  ┌─────────────────────────────────────────────────┐    │
│  │  SSH puerto 2222 + clave pública Ed25519        │    │
│  │  PasswordAuthentication deshabilitado           │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Configuración de Red Estática (Netplan)

Antes de configurar cualquier servicio, es necesario asignar IPs estáticas a cada VM.

### 4.1 Configuración de Red — VM Web

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
      optional: true
      addresses:
        - 10.106.126.200/24
      nameservers:
        addresses:
          - 8.8.8.8
      routes:
        - to: default
          via: 10.106.126.21
```

```bash
sudo netplan apply
```

### 4.2 Configuración de Red — VM DB

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
      optional: true
      addresses:
        - 10.106.126.201/24
      nameservers:
        addresses:
          - 10.106.126.200
          - 8.8.8.8
      routes:
        - to: default
          via: 10.106.126.21
```

```bash
sudo netplan apply
```
### 4.5 Diagrama de Configuración de Red

```
┌─────────────────────────────────────────────────────────┐
│                   VM Web (enp0s3)                       │
├─────────────────────────────────────────────────────────┤
│  IP:          10.106.126.200/24                         │
│  Gateway:     10.106.126.21                             │
│  DNS primario: 10.106.126.200 (sí misma — BIND9)        │
│  DNS secundario: 8.8.8.8 (Google — respaldo externo)    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   VM DB (enp0s3)                        │
├─────────────────────────────────────────────────────────┤
│  IP:          10.106.126.201/24                         │
│  Gateway:     10.106.126.21                             │
│  DNS primario: 10.106.126.200 (VM Web — BIND9)          │
│  DNS secundario: 8.8.8.8 (Google — respaldo externo)    │
└─────────────────────────────────────────────────────────┘
```
---

## 5. Hardening SSH

SSH (Secure Shell) es el protocolo principal para administrar servidores de forma remota. Por defecto viene con configuraciones que priorizan la compatibilidad sobre la seguridad. El hardening SSH consiste en cambiar esas configuraciones para minimizar el riesgo de acceso no autorizado.

### 4.1 ¿Por qué hardening SSH?

| Problema por defecto | Solución aplicada |
|---|---|
| Puerto 22 conocido, atacado por bots | Cambio al puerto 2222 |
| Permite autenticación por contraseña | Solo clave pública |
| Permite login como root | `PermitRootLogin no` |
| Sin límite de intentos | `MaxAuthTries 3` |
| Cualquier usuario puede entrar | `AllowUsers jp` |

### 4.2 Generación de Clave SSH (PC Anfitriona Windows)

El primer paso es generar un par de claves criptográficas en la PC anfitriona. Desde CMD de Windows:

```bash
ssh-keygen -t ed25519 -C "usuario@lab51"
```
| Parámetro | Descripción |
|---|---|
| `ssh-keygen` | Herramienta para generar pares de claves SSH |
| `-t ed25519` | Tipo de algoritmo. Ed25519 usa curvas elípticas, más seguro y eficiente que RSA |
| `-C "usuario@lab51"` | Comentario descriptivo para identificar la clave, no afecta el funcionamiento |

El comando genera dos archivos en `C:\Users\USER\.ssh\`:

| Archivo | Tipo | Descripción |
|---|---|---|
| `id_ed25519` | Clave **privada** | Nunca se comparte. Permanece en la PC anfitriona |
| `id_ed25519.pub` | Clave **pública** | Se copia a todos los servidores que queremos acceder |

El funcionamiento es como una cerradura y su llave: la clave pública es la cerradura (puede estar en cualquier servidor), y la clave privada es la llave (solo la tiene el dueño). Sin la llave correcta, no se puede abrir la cerradura.

### 4.3 Copia de Clave Pública a los Servidores

En Linux existe el comando `ssh-copy-id` para copiar la clave pública, pero Windows no lo incluye. Se usó el siguiente comando equivalente:

**A la VM Web:**
```bash
type "C:\Users\USER\.ssh\id_ed25519.pub" | ssh jp@10.106.126.200 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**A la VM DB:**
```bash
type "C:\Users\USER\.ssh\id_ed25519.pub" | ssh franco@10.106.126.201 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Desglose del comando para Windows:

| Parte | Descripción |
|---|---|
| `type "C:\Users\USER\.ssh\id_ed25519.pub"` | Lee el contenido de la clave pública (equivalente a `cat` en Linux) |
| `\|` | Pipe: pasa la salida del comando anterior como entrada al siguiente |
| `ssh jp@10.106.126.200` | Se conecta al servidor como usuario `jp` |
| `mkdir -p ~/.ssh` | Crea el directorio `.ssh` en el servidor si no existe |
| `&&` | Ejecuta el siguiente comando solo si el anterior tuvo éxito |
| `cat >> ~/.ssh/authorized_keys` | Agrega la clave pública al archivo de claves autorizadas del servidor |

| Parte | Descripción |
|---|---|
| `type "C:\Users\USER\.ssh\id_ed25519.pub"` | Lee el contenido de la clave pública (equivalente a `cat` en Linux) |
| `\|` | Pipe: pasa la salida del comando anterior como entrada al siguiente |
| `ssh franco@10.106.126.201` | Se conecta al servidor como usuario `franco` |
| `mkdir -p ~/.ssh` | Crea el directorio `.ssh` en el servidor si no existe |
| `&&` | Ejecuta el siguiente comando solo si el anterior tuvo éxito |
| `cat >> ~/.ssh/authorized_keys` | Agrega la clave pública al archivo de claves autorizadas del servidor |

### 4.4 Configuración del Servidor SSH (En el WEB y en DB)

```bash
sudo nano /etc/ssh/sshd_config
```

`/etc/ssh/sshd_config` es el archivo de configuración principal del servidor SSH. Cada línea que comienza con `#` está comentada (ignorada). Para activar una opción se quita el `#` y se ajusta el valor.

Las directivas modificadas fueron:

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers jp
```

| Directiva | Valor | ¿Por qué? |
|---|---|---|
| `Port` | `2222` | El puerto 22 es constantemente escaneado por bots en internet. Cambiar el puerto reduce drásticamente el ruido de ataques automatizados |
| `PermitRootLogin` | `no` | El usuario root tiene privilegios totales. Si un atacante lo compromete, tiene control absoluto del sistema |
| `PasswordAuthentication` | `no` | Las contraseñas pueden ser adivinadas por fuerza bruta. Las claves criptográficas son matemáticamente imposibles de adivinar |
| `PubkeyAuthentication` | `yes` | Habilita explícitamente la autenticación por clave pública como único método |
| `MaxAuthTries` | `3` | Después de 3 intentos fallidos, SSH cierra la conexión, dificultando ataques de fuerza bruta |
| `AllowUsers` | `jp` | Lista blanca de usuarios permitidos. Aunque existan otros usuarios en el sistema, no pueden autenticarse por SSH |


Ubuntu 24.04 introdujo un cambio importante: gestiona SSH mediante activación por socket (`ssh.socket`), lo que hace que el servicio SSH ignore el cambio de puerto en `sshd_config` y continúe escuchando en el puerto 22.

```
Problema:
ssh.socket → controla el puerto → ignora sshd_config → siempre usa puerto 22

Solución:
Deshabilitar ssh.socket → habilitar ssh.service → lee sshd_config → usa puerto 2222
```

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh
sudo systemctl restart ssh
```

| Comando | Descripción |
|---|---|
| `systemctl disable --now ssh.socket` | Desactiva el socket de activación y lo detiene inmediatamente (`--now` equivale a `disable` + `stop`) |
| `systemctl enable --now ssh` | Habilita el servicio SSH clásico para iniciar en cada arranque y lo inicia ahora |
| `systemctl restart ssh` | Reinicia el servicio para que aplique la nueva configuración del `sshd_config` |

**Verificación:**

```bash
sudo ss -tulnp | grep ssh
```

| Parámetro | Descripción |
|---|---|
| `ss` | Herramienta para mostrar sockets de red activos (reemplaza al antiguo `netstat`) |
| `-t` | Muestra sockets TCP |
| `-u` | Muestra sockets UDP |
| `-l` | Muestra solo sockets en estado LISTEN (escuchando) |
| `-n` | Muestra números de puerto en lugar de nombres de servicio |
| `-p` | Muestra el proceso que usa cada socket |
| `grep ssh` | Filtra solo las líneas que contienen "ssh" |



---

## 6. Firewall UFW

UFW (Uncomplicated Firewall) es una interfaz simplificada para gestionar las reglas de iptables en Linux. Implementa el principio de **mínimo privilegio**: por defecto todo el tráfico entrante está bloqueado y solo se habilita lo estrictamente necesario.

### 5.1 ¿Cómo funciona UFW?

```
Sin UFW:                      Con UFW (deny by default):
                            
Cualquier puerto              Solo puertos habilitados
┌────────────┐                ┌────────────┐
│ Puerto 22  │ ← accesible    │ Puerto 22  │ ← BLOQUEADO
│ Puerto 80  │ ← accesible    │ Puerto 80  │ ← PERMITIDO
│ Puerto 443 │← accesible     │ Puerto 443 │← PERMITIDO
│ Puerto 3306│← accesible     │ Puerto 3306│← BLOQUEADO
│ Puerto ... │ ← accesible    │ Puerto ... │ ← BLOQUEADO
└────────────┘                └────────────┘
```

### 5.2 Configuración UFW — VM Web
Inslalamos UFW
```bash
sudo apt install ufw -y
```
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

| Comando | Descripción |
|---|---|
| `ufw default deny incoming` | Política por defecto para tráfico entrante: bloquear todo. Cualquier conexión que no tenga una regla `allow` explícita será rechazada |
| `ufw default allow outgoing` | Política por defecto para tráfico saliente: permitir todo. El servidor puede iniciar conexiones hacia afuera libremente |

```bash
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 10.106.126.0/24 to any port 53 proto tcp
sudo ufw allow from 10.106.126.0/24 to any port 53 proto udp
```

| Regla | Puerto | Motivo |
|---|---|---|
| `allow 2222/tcp` | SSH | Acceso de administración remota por el nuevo puerto hardened |
| `allow 80/tcp` | HTTP | Necesario para recibir el tráfico y redirigirlo a HTTPS (301) |
| `allow 443/tcp` | HTTPS | Puerto principal del sitio web cifrado con TLS |
| `allow port 53 tcp/udp from red_aula` | DNS | Consultas DNS solo desde equipos de la red del aula |

```bash
sudo ufw enable
```

Activamos el firewall y lo configuramos para iniciarse automáticamente en cada arranque. Pregunta confirmación porque puede interrumpir conexiones SSH activas.

### 5.3 Configuración UFW — VM DB
Instalamos UFW
```bash
sudo apt install ufw -y
```
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.106.126.0/24 to any port 2222 proto tcp
sudo ufw allow from 10.106.126.200 to any port 3306 proto tcp
sudo ufw enable
```

| Regla | Descripción |
|---|---|
| `allow from 10.106.126.0/24 to any port 2222` | SSH solo desde equipos de la red del aula (`/24` = 256 IPs). Desde internet no se puede acceder por SSH |
| `allow from 10.106.126.200 to any port 3306` | MariaDB **únicamente** desde la IP `10.106.126.200` (VM Web). Ningún otro equipo, ni siquiera de la red del aula, puede conectarse a la base de datos |

### 5.4 Tabla Comparativa de Políticas UFW

#### VM Web — Puertos habilitados

| Puerto | Servicio | Protocolo | Origen permitido | Motivo |
|---|---|---|---|---|
| 2222 | SSH | TCP | Cualquier IP | Administración remota |
| 80 | HTTP | TCP | Cualquier IP | Redirección a HTTPS |
| 443 | HTTPS | TCP | Cualquier IP | Sitio web cifrado |
| 53 | DNS | TCP + UDP | `10.106.126.0/24` | Solo consultas internas |
| Resto | - | - | Bloqueado | Política deny by default |

#### VM DB — Puertos habilitados

| Puerto | Servicio | Protocolo | Origen permitido | Motivo |
|---|---|---|---|---|
| 2222 | SSH | TCP | `10.106.126.0/24` | Solo desde la red del aula |
| 3306 | MariaDB | TCP | `10.106.126.200` | Solo desde VM Web |
| Resto | - | - | Bloqueado | Política deny by default |

---

## 6. Kernel Hardening

El kernel de Linux expone parámetros configurables en tiempo de ejecución a través del sistema de archivos virtual `/proc/sys/`. El comando `sysctl` permite leer y modificar estos parámetros. Guardándolos en `/etc/sysctl.conf` se aplican automáticamente en cada reinicio.

### 6.1 ¿Por qué hardening del kernel?

El firewall protege a nivel de red, pero el kernel también toma decisiones sobre cómo manejar el tráfico de red. Ajustar estos parámetros añade una capa adicional de protección.

```bash
sudo nano /etc/sysctl.conf
```

`/etc/sysctl.conf` es el archivo de configuración persistente de parámetros del kernel. Los cambios aquí sobreviven a reinicios del sistema.

### 6.2 Parámetros aplicados

#### VM Web

```
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=1
kernel.sysrq=0
fs.suid_dumpable=0
```

#### VM DB

```
net.ipv4.ip_forward=0
net.ipv4.conf.all.rp_filter=1
kernel.sysrq=0
fs.suid_dumpable=0
```

| Parámetro | VM Web | VM DB | Descripción |
|---|---|---|---|
| `net.ipv4.ip_forward` | `1` | `0` | **Reenvío de paquetes IP.** En `1` el servidor puede reenviar paquetes entre interfaces (actúa como router). En `0` los paquetes destinados a otras IPs son descartados. La VM Web necesita `1` para funcionar como gateway; la DB no reenvía nada |
| `net.ipv4.conf.all.rp_filter` | `1` | `1` | **Reverse Path Filtering (filtrado de ruta inversa).** Verifica que los paquetes entrantes lleguen por la interfaz correcta según la tabla de rutas. Previene ataques de **IP spoofing** donde un atacante falsifica la IP de origen |
| `kernel.sysrq` | `0` | `0` | **Magic SysRq Key.** Tecla especial que permite ejecutar comandos de bajo nivel del kernel (reinicio forzado, volcado de memoria, etc.) desde el teclado. En servidores de producción debe estar deshabilitada (`0`) para evitar su uso malicioso con acceso físico |
| `fs.suid_dumpable` | `0` | `0` | **SUID dumpable.** Controla si los procesos con bit SUID (que se ejecutan con privilegios elevados) pueden generar archivos de volcado de memoria (core dumps). En `0` se prohíbe, evitando que información sensible como contraseñas o claves en memoria quede grabada en disco |

```bash
sudo sysctl -p
```

| Parte | Descripción |
|---|---|
| `sysctl` | Herramienta para leer y modificar parámetros del kernel en tiempo de ejecución |
| `-p` | Lee el archivo `/etc/sysctl.conf` y aplica todos los parámetros definidos ahí. Si no se especifica archivo, usa el por defecto |

---

## 8. Instalación y Configuración de Nginx con TLS

### 7.1 ¿Qué es TLS y por qué es importante?

TLS (Transport Layer Security) es el protocolo que cifra las comunicaciones entre el navegador y el servidor. Sin TLS, toda la información viaja en texto plano y puede ser interceptada.

```
Sin TLS (HTTP):                  Con TLS (HTTPS):
                                 
Cliente ──── "contraseña123" ───► Servidor    Cliente ──── "X8kP#2mQ..." ───► Servidor
         ← texto visible →                            ← cifrado ←
         ← cualquiera puede                           ← solo servidor
           interceptar →                                puede leer →
```

| Versión TLS | Estado | Motivo |
|---|---|---|
| SSL 3.0 | ❌ Bloqueado | Vulnerable a POODLE (2014) |
| TLS 1.0 | ❌ Bloqueado | Vulnerable a BEAST (2011) |
| TLS 1.1 | ❌ Bloqueado | Obsoleto, sin soporte desde 2020 |
| TLS 1.2 | ✅ Permitido | Seguro con cifrados modernos |
| TLS 1.3 | ✅ Permitido | Más rápido y seguro, estándar actual |

### 7.2 Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

### 7.3 Creación del Sitio Web del Grupo

```bash
sudo mkdir -p /var/www/jpfra.local
sudo nano /var/www/jpfra.local/index.html
```

| Comando | Descripción |
|---|---|
| `mkdir -p /var/www/jpfra.local` | Crea el directorio para los archivos del sitio. `-p` crea directorios intermedios si no existen y no da error si ya existe |
| `nano /var/www/jpfra.local/index.html` | Edita el archivo HTML principal del sitio |

Se creó una página HTML con diseño moderno que muestra la información del grupo, IPs de los servidores y el estado de todas las medidas de seguridad implementadas.

```/var/www/jpfra.local/index.html

```

### 7.4 Generación del Certificado SSL Autofirmado

Un certificado SSL tiene dos componentes:

```
┌────────────────────────────────────────────────────┐
│              Par de Claves SSL                     │
├───────────────────────┬────────────────────────────┤
│   Clave Privada       │   Certificado Público      │
│   (.key)              │   (.crt)                   │
│                       │                            │
│   • Permanece en      │   • Se envía al navegador  │
│     el servidor       │   • Contiene información   │
│   • Firma y descifra  │     del servidor           │
│   • Nunca se comparte │   • Verifica identidad     │
└───────────────────────┴────────────────────────────┘
```

```bash
sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj "/C=BO/ST=Chuquisaca/L=Sucre/O=USFX/OU=SIS313/CN=jpfra.local"
```

| Parámetro | Descripción |
|---|---|
| `openssl req` | Comando OpenSSL para trabajar con solicitudes de certificado (CSR) y certificados |
| `-x509` | Genera directamente un certificado autofirmado en lugar de una solicitud de firma (CSR). En producción real se enviaría el CSR a una CA como Let's Encrypt |
| `-nodes` | "No DES" — no cifra la clave privada con contraseña. Necesario para que Nginx la cargue automáticamente al iniciar sin intervención humana |
| `-days 365` | El certificado expirará en 365 días. Después deberá renovarse |
| `-newkey rsa:2048` | Genera simultáneamente una nueva clave RSA de 2048 bits. Es el tamaño mínimo recomendado para RSA |
| `-keyout` | Ruta donde se guardará la clave privada generada |
| `-out` | Ruta donde se guardará el certificado público generado |
| `-subj` | Información del sujeto del certificado sin modo interactivo |

Desglose del campo `-subj`:

| Campo | Valor | Descripción |
|---|---|---|
| `C` | `BO` | Country — País (código ISO de 2 letras) |
| `ST` | `Chuquisaca` | State — Departamento o estado |
| `L` | `Sucre` | Locality — Ciudad |
| `O` | `USFX` | Organization — Nombre de la organización |
| `OU` | `SIS313` | Organizational Unit — Unidad o departamento |
| `CN` | `jpfra.local` | Common Name — Nombre del dominio al que aplica el certificado |

**Verificación del certificado:**

```bash
sudo openssl x509 -in /etc/nginx/ssl/nginx-selfsigned.crt -text -noout | grep -E "Subject:|Not After"
```

| Parámetro | Descripción |
|---|---|
| `openssl x509` | Herramienta para procesar certificados en formato X.509 |
| `-in` | Archivo de certificado de entrada |
| `-text` | Muestra el certificado en formato texto legible |
| `-noout` | No imprime la versión codificada en Base64, solo el texto |
| `grep -E "Subject:\|Not After"` | Filtra solo las líneas con el sujeto y fecha de expiración |

Resultado obtenido:
```
Not After : May 19 03:57:07 2027 GMT
Subject: C = BO, ST = Chuquisaca, L = Sucre, O = USFX, OU = SIS313, CN = jpfra.local
```

### 7.5 Configuración del Virtual Host con HTTPS

```bash
sudo nano /etc/nginx/sites-available/jpfra.local
```

`/etc/nginx/sites-available/` es el directorio donde se almacenan las configuraciones de todos los sitios (activos o no). Un Virtual Host define cómo Nginx responde para un dominio específico.

```nginx
# Bloque 1: Redirección HTTP → HTTPS
server {
    listen 80;
    server_name jpfra.local www.jpfra.local;
    return 301 https://$server_name$request_uri;
}

# Bloque 2: Servidor HTTPS con hardening TLS
server {
    listen 443 ssl;
    server_name jpfra.local www.jpfra.local;

    root /var/www/jpfra.local;
    index index.html;

    ssl_certificate     /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    server_tokens off;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

| Directiva | Valor | Descripción |
|---|---|---|
| `listen 80` | - | Nginx escucha conexiones HTTP en el puerto 80 |
| `return 301 https://...` | - | Redirección permanente a HTTPS. El código 301 indica al navegador y buscadores que la URL cambió definitivamente |
| `listen 443 ssl` | - | Nginx escucha HTTPS en el puerto 443 con SSL/TLS activado |
| `ssl_protocols` | `TLSv1.2 TLSv1.3` | Solo acepta estas versiones. TLS 1.0 y 1.1 tienen vulnerabilidades conocidas |
| `ssl_ciphers` | Lista de cifrados | Define los algoritmos de cifrado aceptados, priorizando los más seguros (AESGCM, ECDH) |
| `ssl_prefer_server_ciphers on` | - | El servidor impone su lista de cifrados en lugar de aceptar el que proponga el cliente |
| `ssl_session_cache` | `shared:SSL:10m` | Caché compartida de 10MB para sesiones SSL, mejora el rendimiento evitando renegociaciones |
| `ssl_session_timeout` | `1d` | Las sesiones en caché son válidas por 1 día |
| `server_tokens off` | - | Oculta la versión de Nginx en las cabeceras HTTP, dificultando el reconocimiento de vulnerabilidades específicas de versión |

| Cabecera | Valor | Ataque que previene |
|---|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` | **HSTS** — Obliga al navegador a usar HTTPS para este dominio durante 2 años. Previene ataques de downgrade a HTTP |
| `X-Frame-Options` | `SAMEORIGIN` | **Clickjacking** — Impide que el sitio sea incrustado en un `<iframe>` de otro dominio para engañar al usuario |
| `X-Content-Type-Options` | `nosniff` | **MIME Sniffing** — Impide que el navegador intente adivinar el tipo de contenido, previniendo ejecución de scripts disfrazados como imágenes |
| `X-XSS-Protection` | `1; mode=block` | **Cross-Site Scripting (XSS)** — Activa el filtro XSS del navegador y bloquea la página completa si detecta un ataque |

**Activación del sitio:**

```bash
sudo ln -s /etc/nginx/sites-available/jpfra.local /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

| Comando | Descripción |
|---|---|
| `ln -s` | Crea un enlace simbólico (acceso directo). Nginx lee los sitios activos desde `sites-enabled/`. Al usar un enlace en lugar de copiar el archivo, cualquier modificación en `sites-available/` se refleja automáticamente |
| `rm -f /etc/nginx/sites-enabled/default` | Elimina el sitio por defecto de Nginx para evitar conflictos y que muestre su página de bienvenida |
| `nginx -t` | Verifica la sintaxis de toda la configuración de Nginx sin reiniciar el servicio. Si hay errores los muestra sin afectar el servicio en producción |
| `systemctl restart nginx` | Reinicia Nginx para cargar la nueva configuración |

---

## 9. Instalación y Configuración de MariaDB

### 8.1 Instalación

```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
```

MariaDB es un sistema de gestión de bases de datos relacionales derivado de MySQL. Por defecto se instala con configuraciones permisivas que deben ajustarse para producción.

### 8.2 Script de Seguridad mysql_secure_installation

```bash
sudo mysql_secure_installation
```

Este script interactivo guía al administrador para aplicar las configuraciones de seguridad básicas de MariaDB:

| Pregunta | Respuesta | Motivo |
|---|---|---|
| Enter current password for root | Enter (vacío) | Sin contraseña en instalación nueva |
| Switch to unix_socket authentication | `n` | Mantener autenticación por contraseña para compatibilidad |
| Change the root password | `Y` | Establecer contraseña fuerte para root |
| Remove anonymous users | `Y` | Los usuarios anónimos permiten acceso sin credenciales |
| Disallow root login remotely | `Y` | Root solo debe acceder localmente, nunca por red |
| Remove test database | `Y` | La base de datos test es accesible sin privilegios |
| Reload privilege tables | `Y` | Aplica todos los cambios inmediatamente |

### 8.3 Restricción de bind-address

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```
bind-address = 10.106.126.201
```

| Configuración | Descripción |
|---|---|
| `bind-address` | Dirección IP en la que MariaDB acepta conexiones entrantes |
| Valor `127.0.0.1` (por defecto) | Solo acepta conexiones locales |
| Valor `0.0.0.0` | Acepta conexiones desde cualquier IP (inseguro) |
| Valor `10.106.126.201` | Solo acepta conexiones que lleguen a esta interfaz específica |

```bash
sudo systemctl restart mariadb
```

### 8.4 Creación de Usuario y Base de Datos

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE app_db;
CREATE USER 'appuser'@'10.106.126.200' IDENTIFIED BY 'Segura123!';
GRANT ALL PRIVILEGES ON app_db.* TO 'appuser'@'10.106.126.200';
FLUSH PRIVILEGES;
EXIT;
```

| Comando SQL | Descripción |
|---|---|
| `CREATE DATABASE app_db` | Crea la base de datos para la aplicación |
| `CREATE USER 'appuser'@'10.106.126.200'` | Crea un usuario que **solo puede conectarse desde la IP `10.106.126.200`** (VM Web). Desde cualquier otra IP, este usuario no existe para MariaDB |
| `IDENTIFIED BY 'Segura123!'` | Contraseña del usuario, debe ser fuerte |
| `GRANT ALL PRIVILEGES ON app_db.*` | Otorga permisos completos **solo sobre la base de datos `app_db`**, sin acceso a `mysql`, `information_schema` u otras bases de datos del sistema |
| `FLUSH PRIVILEGES` | Recarga la tabla de permisos en memoria para que los cambios surtan efecto inmediatamente sin reiniciar el servicio |

### 8.5 Diagrama de Seguridad de MariaDB

```
                    ACCESO A MARIADB (puerto 3306)
                    
  VM Web (.200) ──────────────────────────► PERMITIDO ✅
  (appuser@10.106.126.200)                  (UFW + bind-address)
  
  PC Anfitriona ──────────────────────────► BLOQUEADO ❌
  (cualquier usuario)                       (UFW deny)
  
  Internet ────────────────────────────────► BLOQUEADO ❌
  (cualquier usuario)                       (UFW deny + bind-address)
  
  Otros equipos aula ─────────────────────► BLOQUEADO ❌
  (cualquier usuario)                       (UFW: solo .200 en 3306)
```

---

## 10. Configuración del Servidor DNS (BIND9)

Para resolver el dominio `jpfra.local` dentro de la red del aula sin modificar el archivo `hosts` de cada PC, se instaló BIND9 en la VM Web.

### 9.1 ¿Por qué BIND9?

| Opción | Ventaja | Desventaja |
|---|---|---|
| Archivo `hosts` en cada PC | Simple, no requiere servidor | Debe editarse en cada equipo manualmente |
| BIND9 en servidor | Un solo punto de configuración, todos los equipos se benefician | Requiere configuración del servidor DNS |

### 9.2 Instalación de BIND9

```bash
sudo apt update
sudo apt install bind9 bind9utils -y
```

| Paquete | Descripción |
|---|---|
| `bind9` | El servidor DNS BIND (Berkeley Internet Name Domain). Es el servidor DNS de código abierto más utilizado en internet |
| `bind9utils` | Herramientas auxiliares: `named-checkconf` (verifica la config), `named-checkzone` (verifica zonas), `rndc` (controla el servicio) |

### 9.3 Declaración de Zona

```bash
sudo nano /etc/bind/named.conf.local
```

```
zone "jpfra.local" {
    type master;
    file "/etc/bind/db.jpfra.local";
};
```

| Directiva | Descripción |
|---|---|
| `zone "jpfra.local"` | Declara que este servidor gestionará la zona DNS para el dominio `jpfra.local` |
| `type master` | Este servidor es el primario (autoritativo). Tiene la copia original de los registros y responde consultas directamente |
| `file` | Ruta al archivo que contiene los registros DNS de esta zona |

### 9.4 Creación del Archivo de Zona

```bash
sudo cp /etc/bind/db.local /etc/bind/db.jpfra.local
sudo nano /etc/bind/db.jpfra.local
```

```
$TTL    604800
@       IN      SOA     jpfra.local. admin.jpfra.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      jpfra.local.
@       IN      A       10.106.126.200
www     IN      A       10.106.126.200
db      IN      A       10.106.126.201
```

| Registro | Tipo | Valor | Descripción |
|---|---|---|---|
| `$TTL 604800` | - | 7 días | Time To Live: tiempo que otros servidores guardan en caché estos registros |
| `@ IN SOA` | SOA | `jpfra.local. admin.jpfra.local.` | Start of Authority: define el servidor primario y parámetros de zona |
| `Serial: 1` | - | `1` | Número de versión. Debe incrementarse con cada cambio para que los secundarios detecten actualizaciones |
| `Refresh: 604800` | - | 7 días | Cada cuánto los servidores secundarios verifican cambios |
| `@ IN NS` | NS | `jpfra.local.` | Name Server: servidor autoritativo para esta zona |
| `@ IN A` | A | `10.106.126.200` | El dominio raíz `jpfra.local` resuelve a la VM Web |
| `www IN A` | A | `10.106.126.200` | `www.jpfra.local` resuelve a la VM Web |
| `db IN A` | A | `10.106.126.201` | `db.jpfra.local` resuelve a la VM DB |

### 9.6 Verificación y Reinicio

```bash
sudo named-checkconf
sudo named-checkzone jpfra.local /etc/bind/db.jpfra.local
sudo systemctl restart bind9
sudo systemctl status bind9
```

| Comando | Descripción |
|---|---|
| `named-checkconf` | Verifica la sintaxis del archivo principal `named.conf` y todos sus includes. Sin salida = sin errores |
| `named-checkzone` | Verifica la sintaxis y consistencia del archivo de zona. Primer argumento: nombre de zona; Segundo: ruta al archivo |
| `systemctl restart bind9` | Reinicia BIND9 para cargar las nuevas zonas |
| `systemctl status bind9` | Muestra el estado actual del servicio |

### 9.7 Deshabilitar systemd-resolved

Ubuntu 24.04 usa `systemd-resolved` como intermediario DNS del sistema. Este servicio tiene un comportamiento especial: intercepta todas las consultas para dominios `.local` y las trata como **Multicast DNS (mDNS)**, impidiendo que lleguen a BIND9.

```
Sin deshabilitar:                    Después de deshabilitar:
                                     
Cliente consulta jpfra.local         Cliente consulta jpfra.local
         │                                    │
         ▼                                    ▼
systemd-resolved                     /etc/resolv.conf
(intercepta .local)                  nameserver 10.106.126.200
         │                                    │
         ▼                                    ▼
mDNS (no encuentra)                  BIND9 en VM Web
         │                                    │
         ▼                                    ▼
NXDOMAIN (error)                     10.106.126.200 ✅
```

```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
sudo bash -c 'echo "nameserver 10.106.126.200" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 8.8.8.8" >> /etc/resolv.conf'
```

| Comando | Descripción |
|---|---|
| `systemctl disable systemd-resolved` | Elimina los enlaces simbólicos que hacen que el servicio inicie automáticamente |
| `systemctl stop systemd-resolved` | Detiene el servicio inmediatamente |
| `rm /etc/resolv.conf` | Elimina el enlace simbólico a `/run/systemd/resolve/stub-resolv.conf` que gestionaba `systemd-resolved` |
| `echo "nameserver 10.106.126.200" > /etc/resolv.conf` | Crea un nuevo `resolv.conf` estático apuntando a BIND9 |
| `echo "nameserver 8.8.8.8" >> /etc/resolv.conf` | Agrega Google DNS como respaldo para nombres externos |

### 9.8 Configuración DNS en PC Anfitriona (Windows)

```
Panel de Control → Centro de redes → Cambiar configuración del adaptador
→ Propiedades → Protocolo de Internet versión 4 (TCP/IPv4)
→ Usar las siguientes direcciones de servidor DNS:
   DNS preferido:   10.106.126.200
   DNS alternativo: 8.8.8.8
```

Con esta configuración, cuando el navegador escribe `https://jpfra.local`, Windows consulta primero al servidor `10.106.126.200` (BIND9 en la VM Web), que responde con `10.106.126.200`, y el navegador se conecta a esa IP por HTTPS.

---

## 11. Flujo Completo de una Solicitud

```
  PC CLIENTE (Windows)
  Escribe: https://jpfra.local
       │
       │ 1. Consulta DNS: ¿Cuál es la IP de jpfra.local?
       ▼
  VM WEB — BIND9 (puerto 53)
  Responde: jpfra.local → 10.106.126.200
       │
       │ 2. Conexión HTTP al puerto 80 de 10.106.126.200
       ▼
  VM WEB — Nginx (puerto 80)
  UFW: permite 80/tcp ✅
  Nginx: return 301 https://jpfra.local
       │
       │ 3. Redirección 301 → HTTPS
       ▼
  VM WEB — Nginx (puerto 443)
  UFW: permite 443/tcp ✅
  TLS Handshake:
    → Nginx envía certificado (CN=jpfra.local)
    → Navegador acepta (autofirmado, usuario confirma)
    → Se establece canal cifrado TLS 1.2/1.3
       │
       │ 4. Respuesta HTTP/1.1 200 OK
       │    Con cabeceras: HSTS, X-Frame, X-XSS, nosniff
       │    Con contenido HTML cifrado
       ▼
  NAVEGADOR muestra la página de jpfra.local
  con candado HTTPS en la barra de direcciones
       │
       │ (Opcional) VM Web consulta la base de datos
       │ 5. Conexión TCP puerto 3306 a 10.106.126.201
       ▼
  VM DB — MariaDB (puerto 3306)
  UFW: permite solo desde 10.106.126.200 ✅
  MariaDB: verifica usuario appuser@10.106.126.200 ✅
  Responde con datos de app_db
```

---

## 12. Verificación y Pruebas

### 11.1 Verificar SSH en puerto 2222

```bash
sudo ss -tulnp | grep ssh
```

Resultado:
```
tcp  LISTEN  0  128  0.0.0.0:2222  0.0.0.0:*  users:(("sshd",pid=1385,fd=3))
tcp  LISTEN  0  128  [::]:2222     [::]:*      users:(("sshd",pid=1385,fd=4))
```
✅ SSH escuchando en puerto 2222, no en el 22.

### 11.2 Prueba redirección HTTP → HTTPS

```bash
curl -I http://10.106.126.200
```

| Parámetro | Descripción |
|---|---|
| `curl` | Herramienta de línea de comandos para transferir datos con URLs |
| `-I` | Muestra solo las cabeceras HTTP de la respuesta, sin el cuerpo |

Resultado:
```
HTTP/1.1 301 Moved Permanently
Server: nginx
Location: https://jpfra.local/
```
✅ HTTP redirige automáticamente a HTTPS.

### 11.3 Verificar cabeceras de seguridad

```bash
curl -k -I https://10.106.126.200
```

| Parámetro | Descripción |
|---|---|
| `-k` | Ignora errores de certificado SSL (necesario porque es autofirmado) |
| `-I` | Muestra solo las cabeceras de respuesta |

Resultado:
```
HTTP/1.1 200 OK
Server: nginx
Strict-Transport-Security: max-age=63072000; includeSubDomains
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
```
✅ Todas las cabeceras de seguridad presentes.

### 11.4 Prueba TLS 1.2 (debe FUNCIONAR)

```bash
openssl s_client -connect 10.106.126.200:443 -tls1_2 </dev/null
```

| Parámetro | Descripción |
|---|---|
| `openssl s_client` | Cliente SSL/TLS de línea de comandos para probar conexiones |
| `-connect IP:puerto` | Servidor al que conectarse |
| `-tls1_2` | Fuerza el uso de TLS 1.2 exclusivamente |
| `</dev/null` | Cierra la entrada inmediatamente para que el comando termine solo |

Resultado: `CONNECTED` — TLS 1.2 aceptado ✅

### 11.5 Prueba TLS 1.1 (debe FALLAR)

```bash
openssl s_client -connect 10.106.126.200:443 -tls1_1 </dev/null
```

Resultado:
```
no protocols available
Cipher is (NONE)
```
✅ TLS 1.1 correctamente bloqueado.

### 11.6 Verificar puerto 3306 bloqueado desde exterior

```bash
curl -v telnet://10.106.126.201:3306
```

Resultado:
```
connect to 10.106.126.201 port 3306 failed: Timed out
```
✅ MariaDB no accesible desde equipos externos.

### 11.7 Verificar resolución DNS

```bash
dig @10.106.126.200 jpfra.local
dig @10.106.126.200 www.jpfra.local
```

| Parámetro | Descripción |
|---|---|
| `dig` | Domain Information Groper — herramienta para consultas DNS |
| `@10.106.126.200` | Servidor DNS al que enviar la consulta (nuestra VM Web con BIND9) |
| `jpfra.local` | Nombre a resolver |

Resultado:
```
;; ANSWER SECTION:
jpfra.local.      604800  IN  A  10.106.126.200
www.jpfra.local.  604800  IN  A  10.106.126.200

;; Query time: 0 msec
;; status: NOERROR
```
✅ BIND9 resuelve correctamente el dominio.

### 11.8 Resumen de Pruebas

| # | Prueba | Comando | Resultado esperado | Estado |
|---|---|---|---|---|
| 1 | SSH en puerto 2222 | `ss -tulnp \| grep ssh` | `0.0.0.0:2222` | ✅ |
| 2 | SSH puerto 22 bloqueado | `ssh jp@IP` (sin -p) | Connection timeout | ✅ |
| 3 | Redirección HTTP→HTTPS | `curl -I http://IP` | `301 Moved Permanently` | ✅ |
| 4 | Cabeceras de seguridad | `curl -k -I https://IP` | HSTS, X-Frame, X-XSS | ✅ |
| 5 | TLS 1.2 funciona | `openssl ... -tls1_2` | `CONNECTED` | ✅ |
| 6 | TLS 1.1 bloqueado | `openssl ... -tls1_1` | `no protocols available` | ✅ |
| 7 | Puerto 3306 bloqueado | `curl -v telnet://IP:3306` | `Timed out` | ✅ |
| 8 | DNS resuelve dominio | `dig @IP jpfra.local` | `NOERROR` → `10.106.126.200` | ✅ |
| 9 | Acceso web por dominio | Navegador `https://jpfra.local` | Página del grupo con HTTPS | ✅ |

---

## 13. Resumen de Medidas de Seguridad Implementadas

| Capa | Medida | Herramienta | VM Web | VM DB |
|---|---|---|---|---|
| Acceso remoto | Puerto SSH cambiado a 2222 | sshd_config | ✅ | ✅ |
| Acceso remoto | Solo autenticación por clave pública | sshd_config | ✅ | ✅ |
| Acceso remoto | Root login deshabilitado | sshd_config | ✅ | ✅ |
| Acceso remoto | Máximo 3 intentos de autenticación | sshd_config | ✅ | ✅ |
| Acceso remoto | Lista blanca de usuarios (AllowUsers) | sshd_config | ✅ | ✅ |
| Red | Firewall UFW activo con deny by default | UFW | ✅ | ✅ |
| Red | Puerto 3306 solo desde VM Web | UFW | - | ✅ |
| Red | Puerto 53 solo desde red del aula | UFW | ✅ | - |
| Sistema | Filtrado de ruta inversa anti-spoofing | sysctl | ✅ | ✅ |
| Sistema | SysRq deshabilitado | sysctl | ✅ | ✅ |
| Sistema | SUID dumpable deshabilitado | sysctl | ✅ | ✅ |
| Transporte | HTTPS con TLS 1.2/1.3 únicamente | Nginx + OpenSSL | ✅ | - |
| Transporte | Certificado SSL autofirmado | OpenSSL | ✅ | - |
| Aplicación | Cabecera HSTS (2 años) | Nginx | ✅ | - |
| Aplicación | Cabecera X-Frame-Options | Nginx | ✅ | - |
| Aplicación | Cabecera X-Content-Type-Options | Nginx | ✅ | - |
| Aplicación | Cabecera X-XSS-Protection | Nginx | ✅ | - |
| Aplicación | server_tokens off (versión oculta) | Nginx | ✅ | - |
| Base de datos | bind-address restringido a IP interna | MariaDB | - | ✅ |
| Base de datos | Usuario sin privilegios root | MariaDB | - | ✅ |
| Base de datos | Acceso solo desde VM Web por IP | MariaDB | - | ✅ |
| DNS | Resolución de jpfra.local | BIND9 | ✅ | - |

---

## 14. Conclusiones

- El hardening SSH con autenticación por clave pública Ed25519 elimina matemáticamente la posibilidad de ataques de fuerza bruta por contraseña. Una clave Ed25519 de 256 bits es equivalente en seguridad a una clave RSA de 3072 bits, siendo además más rápida de procesar.

- El cambio del puerto SSH del estándar 22 al 2222, aunque no es una medida de seguridad por sí sola (seguridad por oscuridad), reduce drásticamente el ruido de bots automáticos que escanean continuamente el puerto 22 en busca de servidores vulnerables.

- UFW con política `deny` por defecto implementa el principio de **mínimo privilegio**: todo está bloqueado a menos que se habilite explícitamente. Esto significa que cualquier servicio nuevo instalado por error no estará automáticamente expuesto.

- El endurecimiento del kernel mediante `sysctl` añade una capa de protección a nivel de sistema operativo que opera independientemente del firewall. El parámetro `rp_filter` previene ataques de IP spoofing donde un atacante falsifica la dirección de origen de los paquetes para evadir controles de acceso.

- TLS 1.2 y 1.3 son los únicos protocolos de transporte seguro aceptados. Las versiones anteriores tienen vulnerabilidades críticas documentadas: POODLE afecta a SSL 3.0, BEAST a TLS 1.0, y tanto TLS 1.0 como 1.1 fueron oficialmente deprecados por la IETF en 2021 (RFC 8996).

- Las cabeceras de seguridad HTTP (HSTS, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection) protegen a los usuarios del sitio contra ataques comunes del lado del cliente. HSTS en particular es crucial porque una vez que el navegador lo recibe, fuerza HTTPS durante 2 años incluso si el usuario escribe `http://` manualmente.

- Restringir el `bind-address` de MariaDB a la IP interna combinado con reglas UFW que solo permiten conexiones desde la VM Web crea una **defensa en profundidad**: incluso si un atacante logra evadir el firewall, MariaDB no acepta conexiones desde IPs no autorizadas. Dos capas independientes deben fallar para que el ataque tenga éxito.

- La configuración de BIND9 en la VM Web demuestra la integración de servicios de infraestructura: el servidor web no solo sirve contenido, sino que también actúa como punto de resolución DNS para la red del aula, mostrando cómo en ambientes reales los servicios se complementan entre sí.

- La segmentación de servicios (web en una VM, base de datos en otra) sigue el principio de **separación de responsabilidades**. Si la VM Web fuera comprometida por una vulnerabilidad en Nginx, el atacante no tendría acceso directo a la base de datos desde el exterior, ya que el UFW de la VM DB solo permite el puerto 3306 desde la IP de la VM Web, no desde internet.

---
