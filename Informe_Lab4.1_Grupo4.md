# Informe de Laboratorio 4.1
## Plataforma HA, Balanceo de Carga y Monitoreo

---

**Universidad Mayor Real y Pontificia de San Francisco Xavier de Chuquisaca**

**Facultad de Ciencias y Tecnología**

**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes

**Docente:** Ing. Marcelo Quispe Ortega

**Estudiantes:**                                             **Carrera:**

- Juan Pablo Taboada Camacho (PROXY + MONITOREO)                      Ing. Sistemas
- Franco Milton Jimenez Amachuy (APLICACIÓN 1)                           Ing. Sistemas
- Jose Enrique Arce Vargas (APLICACIÓN 2)                           Ing. Sistemas
- Shirley Celene Arancibia Morales (BASE DE DATOS)                          Ing. Sistemas

---

## 1. Introducción

El presente laboratorio tiene como objetivo implementar una plataforma de Alta Disponibilidad (HA) en un entorno real de Centro de Datos universitario, utilizando cuatro máquinas virtuales con Ubuntu Server 24.04 LTS conectadas mediante VLAN. La arquitectura implementada sigue el patrón de segregación de servicios: un proxy inverso (Nginx) que balancea el tráfico entre dos servidores de aplicaciones Node.js independientes, ambos conectados a una base de datos MariaDB centralizada. Adicionalmente, se integró un sistema de monitoreo con Prometheus y Grafana para observar el estado de toda la plataforma en tiempo real.

---

## 2. Objetivos del Laboratorio

- **Configurar un Proxy Inverso con Nginx** para balancear el tráfico entre dos servidores de aplicaciones mediante Round-Robin.
- **Desplegar dos instancias independientes** de una aplicación Node.js gestionadas con PM2 en modo producción.
- **Instalar y asegurar una Base de Datos MariaDB** centralizada con acceso restringido por IP mediante UFW.
- **Integrar Prometheus y Grafana** para monitorear el estado de las cuatro VMs del grupo.
- **Demostrar el failover automático** al simular la caída de un servidor de aplicaciones.
- **Comprender el concepto de Alta Disponibilidad** y su implementación práctica en infraestructuras reales.

---

## 3. Topología de Red

### 3.1 Arquitectura General

```
                        INTERNET / CLIENTE
                               │
                    https://vlan104-app.rootcode.com.bo
                               │
                ┌──────────────▼──────────────────┐
                │     NGINX PROXY  192.168.104.2   │
                │     Balanceo Round-Robin (:80)   │
                │  Prometheus (:9090)              │
                │  Grafana (:3000)                 │
                │  Node Exporter (:9100)           │
                └──────────┬──────────────┬────────┘
                           │              │
              ┌────────────▼──┐       ┌───▼───────────┐
              │  APP 1        │       │  APP 2        │
              │ 192.168.104.3 │       │ 192.168.104.4 │
              │  Node.js :3000│       │  Node.js :3000│
              │  PM2          │       │  PM2          │
              │  Node Exp:9100│       │  Node Exp:9100│
              └────────┬──────┘       └──────┬────────┘
                       └──────────┬───────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │   MariaDB  192.168.104.5   │
                    │   db_movies  Puerto 3306   │
                    │   UFW: acceso restringido  │
                    │   Node Exporter (:9100)    │
                    └────────────────────────────┘
```

### 3.2 Tabla de Direccionamiento

| Rol | IP Asignada | Hostname | VLAN ID | VLAN IP | Servicios Instalados |
|-----|-------------|----------|---------|---------|----------------------|
| Gateway / Router |  | Server Docente | 104 | 192.168.104.1 | Enrutamiento |
| Proxy + Monitoreo | 192.168.100.163 | server-163 | 104 | 192.168.104.2 | Nginx, Prometheus, Grafana, Node Exporter |
| Aplicación 1 | 192.168.100.164 | server-164 | 104 | 192.168.104.3 | Node.js, PM2, Node Exporter |
| Aplicación 2 | 192.168.100.165 | server-165 | 104 | 192.168.104.4 | Node.js, PM2, Node Exporter |
| Base de Datos | 192.168.100.166 | server-DB (166) | 104 | 192.168.104.5 | MariaDB, Node Exporter |

| Parámetro | Valor |
|-----------|-------|
| Subred asignada | `192.168.104.0/29` |
| VLAN ID | `104` |
| Gateway | `192.168.104.1` |
| Rango de hosts | `192.168.104.1 – 192.168.104.6` |

### 3.3 Subdominios Asignados

| Servicio | URL pública | Apunta a |
|----------|-------------|----------|
| Aplicación (Proxy) | `https://vlan104-app.rootcode.com.bo` | `192.168.104.2:80` |
| Grafana (Monitoreo) | `https://vlan104-monitoring.rootcode.com.bo` | `192.168.104.2:3000` |

### 3.4 Diagrama de Infraestructura Real del Centro de Datos

El siguiente diagrama refleja la infraestructura física y virtual real utilizada durante la práctica, incluyendo el acceso desde internet, el servidor proxy del Centro de Datos y las VMs asignadas al grupo:

```
              ┌─────────────────┐
              │    INTERNET     │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │   PC FÍSICA     │
              │  (Anfitriona)   │
              └────────┬────────┘
                       │
                       │  SSH / Navegador
                       │
    ┌──────────────────▼──────────────────────┐
    │          SERVER PROXY USFX              │
    │       (Centro de Datos físico)          │
    │                                         │
    │   IP pública:  201.131.45.42            │
    │   user:        usrproxy                 │
    │   password:    1234abcd.$               │
    └──────────────────┬──────────────────────┘
                       │
                       │  
                       │  
                       │
    ┌──────────────────▼──────────────────────┐
    │         SERVER VM VIRTUAL               │
    │      (VM asignada al grupo)             │
    │                                         │
    │   IP base:  192.168.100.163-166/24      │
    │   user:     admin163  (sudoer)          │
    │   password: xxxx                        │
    └────────────┬──────────┬────────┬────────┘
                 │          │        │        
         ┌───────▼──┐  ┌────▼───┐  ┌▼───────┐ 
         │  VM      │  │  VM    │  │  VM    │
         │ Proxy    │  │ App1   │  │  DB    │
         │+Monitoreo│  │+App2   │  │MariaDB │
         │ .104.2   │  │.104.3/4│  │ .104.5 │
         └──────────┘  └────────┘  └────────┘
```

- **PC Física** — computadora del laboratorio desde la cual cada integrante accede remotamente a su VM mediante SSH o navegador web.
- **Server Proxy USFX (201.131.45.42)** — servidor físico del Centro de Datos de la universidad que actúa como punto de entrada desde internet. Tiene configuradas las rutas que redirigen el tráfico de los subdominios `vlan104-app.rootcode.com.bo` y `vlan104-monitoring.rootcode.com.bo` hacia las VMs del grupo.
- **Server VM Virtual (192.168.100.163 - 166)** — servidor virtual asignado al grupo dentro de la infraestructura del Centro de Datos. Desde aquí se configuró la VLAN 104 y se desplegaron todos los servicios del laboratorio.
- **VLAN 104 (192.168.104.0/29)** — red virtual que interconecta todas las VMs del grupo dentro del Centro de Datos con el VLAN ID 104 configurado en el switch. Permite la comunicación entre Proxy, Apps y DB de forma aislada del resto de grupos.

---

## 4. Integrantes y Roles

| Alumno | Rol | VM | IP Asignada | IP VLAN |
|--------|-----|----|-------------|---------|
| Juan Pablo Taboada Camacho | Proxy + Monitoreo | VM Proxy | 192.168.100.163 | 192.168.104.2 |
| Franco Milton Jimenez Amachuy | Aplicación 1 | VM App1 | 192.168.100.164 | 192.168.104.3 |
| Jose Enrique Arce Varga | Aplicación 2 | VM App2 | 192.168.100.165 | 192.168.104.4 |
| Shirley Celene Arancibia Morales | Base de Datos | VM DB | 192.168.100.166 | 192.168.104.5

---

## 5. Configuración VM Proxy + Monitoreo (192.168.104.2)

### 5.1 Configuración de Red con VLAN 104

![alt text](<Screenshot 2026-05-14 224304.png>)

![alt text](<Screenshot 2026-05-14 224316.png>)

Abrimos el archivo de configuración de red de Ubuntu con el editor nano:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

![alt text](<Screenshot 2026-05-14 224409.png>)

- `/etc/netplan/50-cloud-init.yaml` — archivo donde Ubuntu guarda la configuración de red. Netplan es el sistema de configuración de red moderno de Ubuntu que reemplaza al antiguo `/etc/network/interfaces`.

El archivo quedó con el siguiente contenido:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      optional: true
      addresses:
        - "192.168.100.163/24"
  vlans:
    vlan104:
      id: 104
      link: ens18
      addresses:
        - "192.168.104.2/29"
      nameservers:
        addresses:
          - 8.8.8.8
      routes:
        - to: default
          via: 192.168.104.1
```

![alt text](<Screenshot 2026-05-14 224346.png>)

- `version: 2` — versión del formato de Netplan. Actualmente la versión 2 es la estándar y soporta todas las funcionalidades modernas.
- `renderer: networkd` — indica que se usará `systemd-networkd` como backend para aplicar la configuración. Es el recomendado para servidores Ubuntu sin interfaz gráfica.
- `ethernets` — sección donde se configuran las interfaces de red físicas del servidor.
- `ens18` — nombre de la interfaz de red física. El nombre varía según el hardware de cada VM (puede ser `ens18`, `enp0s3`, `eth0`, etc.).
- `dhcp4: no` — desactiva la asignación automática de IP por DHCP. Usamos IP fija para que el servidor siempre tenga la misma dirección y los otros servicios puedan apuntarle.
- `optional: true` — permite que el sistema arranque aunque esta interfaz no tenga conexión activa, evitando que el boot quede bloqueado esperando la red.
- `addresses: "192.168.100.163/24"` — IP estática de la interfaz física en la red de gestión del servidor. 
- `vlans` — sección donde se definen las interfaces VLAN virtuales que se crearán sobre la interfaz física.
- `vlan104` — nombre descriptivo de la interfaz VLAN virtual que se creará en el sistema operativo.
- `id: 104` — identificador numérico de la VLAN (VLAN ID). Este número debe coincidir exactamente con la VLAN configurada en el switch del Centro de Datos por el docente. Es la "etiqueta" que se agrega a los paquetes de red para identificarlos.
- `link: ens18` — indica que la VLAN se creará sobre la interfaz física `ens18`. El tráfico VLAN viajará etiquetado (tagged) por esta interfaz física.
- `addresses: "192.168.104.2/29"` — IP estática de esta VM dentro de la VLAN 104. La máscara `/29` significa que la subred tiene 6 hosts disponibles (`192.168.104.1 – 192.168.104.6`).
- `nameservers: addresses: 8.8.8.8` — servidor DNS de Google para resolver nombres de internet (como `github.com` al instalar paquetes).
- `routes: to: default via: 192.168.104.1` — define la ruta por defecto (gateway). Todo el tráfico que no sea de la subred local saldrá por `192.168.104.1` (el servidor del docente que actúa como router).

Aplicamos la configuración de red sin necesidad de reiniciar el servidor:

```bash
sudo netplan apply
```

![alt text](<Screenshot 2026-05-14 232153.png>)

- `netplan apply` — lee el archivo YAML y aplica la configuración de red inmediatamente. Si hay un error de sintaxis en el archivo (como indentación incorrecta), mostrará un mensaje de error y no aplicará los cambios.

Verificamos que la interfaz VLAN se creó y tiene la IP asignada:

```bash
ip addr show vlan104
```

![alt text](<Screenshot 2026-05-14 231144.png>)


---

### 5.2 Instalación de Nginx

Actualizamos la lista de paquetes disponibles en los repositorios de Ubuntu:

```bash
sudo apt update
```

![alt text](<Screenshot 2026-05-14 231218.png>)

- `apt update` — descarga la lista actualizada de paquetes disponibles desde los repositorios configurados. No instala nada, solo actualiza el índice local para saber qué versiones están disponibles. Es importante ejecutarlo antes de instalar cualquier paquete para obtener las versiones más recientes.

Instalamos Nginx:

```bash
sudo apt install nginx -y
```

![alt text](<Screenshot 2026-05-14 231244.png>)

- `nginx` — servidor web y proxy inverso de alto rendimiento, ampliamente usado en entornos de producción por su capacidad de manejar miles de conexiones simultáneas con bajo consumo de recursos.

---

### 5.3 Configuración de Nginx como Proxy Inverso y Balanceador de Carga

Reemplazamos la configuración por defecto de Nginx:

```bash
sudo nano /etc/nginx/sites-available/default
```

![alt text](<Screenshot 2026-05-14 231425.png>)

- `/etc/nginx/sites-available/` — directorio donde se guardan las configuraciones de todos los sitios disponibles en Nginx, estén activos o no.
- `default` — archivo de configuración del sitio por defecto. Al instalar Nginx contiene una configuración básica de servidor web estático que debemos reemplazar completamente por nuestra configuración de proxy.

El archivo quedó con el siguiente contenido:

```nginx
upstream loadbalancer {
    server 192.168.104.3:3000;
    server 192.168.104.4:3000;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://loadbalancer;
    }
}
```

![alt text](<Screenshot 2026-05-14 231418.png>)

- `upstream loadbalancer` — define un grupo de servidores backend entre los cuales Nginx distribuirá el tráfico. El nombre `loadbalancer` es arbitrario; se puede usar cualquier nombre descriptivo.
- `server 192.168.104.3:3000` — primer servidor de aplicaciones (App 1). Nginx enviará peticiones a esta IP y puerto. Por defecto usa el algoritmo Round-Robin: envía la primera petición a App 1, la segunda a App 2, la tercera a App 1, y así sucesivamente.
- `server 192.168.104.4:3000` — segundo servidor de aplicaciones (App 2). Ambas apps corren en el puerto 3000 porque están en VMs diferentes con IPs distintas, por lo que no hay conflicto de puertos.
- `server { ... }` — bloque que define cómo Nginx responde a las peticiones HTTP entrantes.
- `listen 80` — Nginx escuchará conexiones en el puerto 80, que es el puerto estándar para HTTP.
- `server_name _` — el guión bajo `_` es un comodín que indica que este bloque responderá a cualquier nombre de dominio o IP con la que sea accedido el servidor, sin importar cuál.
- `location /` — bloque que define cómo manejar las peticiones a cualquier ruta del sitio (la barra `/` abarca todas las rutas posibles).
- `proxy_pass http://loadbalancer` — redirige la petición al grupo upstream `loadbalancer`. Nginx actúa como intermediario: recibe la petición del cliente, la reenvía al servidor de aplicaciones elegido por Round-Robin, recibe la respuesta y la devuelve al cliente. A esto se llama "proxy inverso".

Verificamos la sintaxis antes de aplicar:

```bash
sudo nginx -t
```

![alt text](<Screenshot 2026-05-14 231538.png>)


Reiniciamos Nginx para aplicar la nueva configuración:

```bash
sudo systemctl restart nginx
```

![alt text](<Screenshot 2026-05-14 231633.png>)

---

### 5.4 Instalación de Prometheus y Node Exporter

Instalamos ambos servicios en la VM Proxy:

```bash
sudo apt install prometheus prometheus-node-exporter -y
```

![alt text](<Screenshot 2026-05-14 232622.png>)

- `prometheus` — sistema de monitoreo de código abierto que recolecta métricas periódicamente. Funciona con un modelo "pull": cada ciertos segundos (por defecto 15s) consulta los endpoints de métricas de cada servidor configurado y almacena los datos en su base de datos de series temporales propia (TSDB).
- `prometheus-node-exporter` — agente ligero que se instala en cada servidor a monitorear. Recolecta métricas del sistema operativo (uso de CPU, RAM disponible, espacio en disco, tráfico de red, temperatura, etc.) y las expone en el puerto `9100` en un formato de texto que Prometheus puede leer.

Verificamos que Node Exporter esté exponiendo métricas:

```bash
curl http://localhost:9100/metrics
```

- `curl` — herramienta de línea de comandos para hacer peticiones HTTP y ver el resultado en la terminal.
- `http://localhost:9100/metrics` — endpoint donde Node Exporter publica las métricas del sistema. Si responde con una larga lista de métricas en formato texto (líneas que comienzan con `#` y nombres de métricas), el agente está funcionando correctamente.

![alt text](<Screenshot 2026-05-14 232941.png>)

---

### 5.5 Configuración de Prometheus

Editamos el archivo de configuración principal de Prometheus:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

![alt text](<Screenshot 2026-05-14 233029.png>)

- `/etc/prometheus/prometheus.yml` — archivo de configuración de Prometheus en formato YAML. Define los intervalos de scraping, reglas de alertas y los servidores de los que recolectar métricas.

Reemplazamos la sección `scrape_configs` con el siguiente contenido:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-proxy'
    static_configs:
      - targets: ['192.168.104.2:9100']

  - job_name: 'node-app1'
    static_configs:
      - targets: ['192.168.104.3:9100']

  - job_name: 'node-app2'
    static_configs:
      - targets: ['192.168.104.4:9100']

  - job_name: 'node-db'
    static_configs:
      - targets: ['192.168.104.5:9100']
```

![alt text](<Screenshot 2026-05-14 233108.png>)

- `scrape_configs` — sección principal que define todos los "trabajos" de recolección de métricas. Cada `job_name` es un grupo lógico de servidores.
- `job_name: 'prometheus'` — Prometheus también monitorea sus propias métricas internas para saber si él mismo está funcionando correctamente.
- `job_name: 'node-proxy'` — trabajo de monitoreo para la VM del Proxy. El nombre del job aparecerá como etiqueta en Grafana, permitiendo filtrar métricas por VM.
- `job_name: 'node-app1'` — trabajo para la VM de la Aplicación 1.
- `job_name: 'node-app2'` — trabajo para la VM de la Aplicación 2.
- `job_name: 'node-db'` — trabajo para la VM de la Base de Datos.
- `static_configs` — indica que las IPs de los servidores están definidas de forma fija, no de forma dinámica.
- `targets: ['192.168.104.X:9100']` — dirección IP y puerto del Node Exporter de cada VM. Prometheus hará una petición HTTP a `http://192.168.104.X:9100/metrics` cada 15 segundos para recolectar las métricas.

Reiniciamos Prometheus para que cargue la nueva configuración:

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```
![alt text](<Screenshot 2026-05-14 233245.png>)

---

### 5.6 Instalación de Grafana

Instalamos las dependencias necesarias para agregar repositorios externos:

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
```

![alt text](<Screenshot 2026-05-14 233313.png>)

- `apt-transport-https` — permite que el gestor de paquetes `apt` descargue paquetes desde repositorios que usan HTTPS en lugar de HTTP plano, garantizando la seguridad de la descarga.
- `software-properties-common` — proporciona herramientas para gestionar repositorios de software adicionales de terceros.
- `wget` — herramienta para descargar archivos desde internet directamente en la terminal.

Creamos el directorio para las claves GPG de repositorios externos:

```bash
sudo mkdir -p /etc/apt/keyrings/
```

![alt text](<Screenshot 2026-05-14 233345.png>)

- `/etc/apt/keyrings/` — ubicación estándar moderna para almacenar las claves de autenticación de repositorios de terceros, separadas de las claves del sistema.

Descargamos e importamos la clave GPG de Grafana:

```bash
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

![alt text](<Screenshot 2026-05-14 233453.png>)

- `wget -q -O -` — descarga la clave GPG silenciosamente (`-q`) y envía el resultado a la salida estándar (`-O -`) en lugar de guardarlo en archivo.
- `gpg --dearmor` — convierte la clave del formato ASCII (texto legible) al formato binario que `apt` puede usar para verificar la autenticidad de los paquetes.
- `sudo tee /etc/apt/keyrings/grafana.gpg` — guarda la clave binaria en el archivo. `tee` escribe tanto al archivo como a la salida estándar.
- `> /dev/null` — descarta la salida en pantalla para no mostrar el contenido binario de la clave.

Agregamos el repositorio oficial de Grafana:

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

![alt text](<Screenshot 2026-05-14 233528.png>)

- `echo "deb ..."` — genera la línea de configuración del repositorio en el formato que `apt` entiende.
- `deb` — indica que es un repositorio de paquetes binarios listos para instalar.
- `[signed-by=/etc/apt/keyrings/grafana.gpg]` — le dice a `apt` que verifique la autenticidad de los paquetes con la clave que descargamos. Garantiza que los paquetes son genuinos y no fueron modificados por terceros.
- `https://apt.grafana.com` — URL del repositorio oficial de Grafana Labs.
- `stable main` — rama `stable` del repositorio, que contiene versiones probadas y recomendadas para producción.
- `sudo tee -a /etc/apt/sources.list.d/grafana.list` — agrega (`-a`) la línea al archivo de fuentes de Grafana.

Actualizamos e instalamos Grafana:

```bash
sudo apt update
sudo apt install grafana -y
```

![alt text](<Screenshot 2026-05-14 233627.png>)


Habilitamos e iniciamos Grafana:

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

- `systemctl daemon-reload` — recarga la configuración del gestor de servicios systemd para que reconozca el nuevo servicio de Grafana instalado.
- `systemctl enable grafana-server` — configura Grafana para iniciarse automáticamente en cada arranque del servidor.
- `systemctl start grafana-server` — inicia el servicio de Grafana inmediatamente. Estará disponible en el puerto `3000`.

![alt text](<Screenshot 2026-05-14 233716.png>)

---

### 5.7 Configuración de Grafana en la Interfaz Web

Accedemos a Grafana desde el navegador:

```
https://vlan104-monitoring.rootcode.com.bo
```

![alt text](<Screenshot 2026-05-13 201712.png>)

- Usuario: `admin` / Contraseña: `admin`
- Grafana solicitará cambiar la contraseña al primer ingreso.

**Agregar Prometheus como Data Source:**

Navegamos a **Connections → Add new connection → Prometheus** y configuramos:

![alt text](<Screenshot 2026-05-13 201855.png>)

![alt text](<Screenshot 2026-05-13 201945.png>)

![alt text](<Screenshot 2026-05-13 202032.png>)

![alt text](<Screenshot 2026-05-13 202054.png>)

- **URL:** `http://localhost:9090`
  - Se usa `localhost` porque Prometheus está en la misma VM que Grafana. Esto es más eficiente que usar la IP de la VLAN porque el tráfico no sale por la red física.
- Click en **Save & Test** — Grafana hace una petición de prueba a Prometheus. El mensaje `"Successfully queried the Prometheus API"` confirma que la conexión funciona correctamente.

![alt text](<Screenshot 2026-05-13 202141.png>)

![alt text](<Screenshot 2026-05-13 202205.png>)

![alt text](<Screenshot 2026-05-13 202259.png>)


**Importar Dashboard 1 — Node Exporter Full (ID: 1860):**

Navegamos a **Dashboards → New → Import**, ingresamos `1860` en el campo de ID y hacemos click en **Load** → **Import**.

- Este dashboard muestra métricas detalladas de cada VM: uso de CPU, memoria RAM, espacio en disco, tráfico de red y uptime. Es uno de los dashboards más populares de la comunidad Grafana para monitoreo de servidores Linux.

**Importar Dashboard 2 — Node Exporter Full with Node Name (ID: 10242):**

Ingresamos `10242`, hacemos click en **Load**, seleccionamos `prometheus` en el campo **DS_PROMETHEUS** y hacemos click en **Import**.

- Este dashboard permite seleccionar y comparar métricas de múltiples nodos usando los filtros `Job` y `Host` en la parte superior, útil para ver el estado de todas las VMs del grupo desde una sola vista.

![alt text](<Screenshot 2026-05-13 202332.png>)

![alt text](<Screenshot 2026-05-13 202414.png>)

![alt text](<Screenshot 2026-05-13 202447.png>)

![alt text](<Screenshot 2026-05-13 202559.png>)

![alt text](<Screenshot 2026-05-13 202654.png>)

![alt text](<Screenshot 2026-05-13 202729.png>)

![alt text](<Screenshot 2026-05-14 123301.png>)

![alt text](<Screenshot 2026-05-14 123400.png>)

---

## 6. Configuración VM Aplicación 1 (192.168.104.3)

### 6.1 Configuración de Red con VLAN 104

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      optional: true
      addresses:
        - "192.168.100.164/24"
  vlans:
    vlan104:
      id: 104
      link: ens18
      addresses:
        - "192.168.104.3/29"
      nameservers:
        addresses:
          - 8.8.8.8
      routes:
        - to: default
          via: 192.168.104.1
```
![alt text](<Screenshot 2026-05-14 234843.png>)

- `addresses: "192.168.104.3/29"` — IP estática de esta VM (Aplicación 1) dentro de la VLAN 104. Es la única diferencia respecto a la configuración del Proxy; el resto de parámetros tienen el mismo significado explicado en la sección 5.1.
- `via: 192.168.104.1` — el gateway de la VLAN (servidor del docente) por donde saldrá el tráfico hacia internet para descargar paquetes.

```bash
sudo netplan apply
```

---

### 6.2 Instalación de Node.js con NVM

Descargamos e instalamos NVM (Node Version Manager):

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

- `curl -o-` — descarga el script y envía su contenido a la salida estándar (`-o-`) en lugar de guardarlo en un archivo local.
- La URL apunta al script de instalación oficial de NVM versión 0.40.3 directamente desde GitHub.
- `| bash` — el pipe `|` toma la salida del `curl` (el contenido del script) y lo envía como entrada a `bash`, que lo ejecuta directamente. NVM se instala en `~/.nvm` (directorio personal del usuario).
- **NVM** (Node Version Manager) — herramienta que permite instalar y gestionar múltiples versiones de Node.js, facilitando cambiar entre versiones según los requisitos de cada proyecto.

Cargamos NVM en la sesión actual sin cerrar la terminal:

```bash
. "$HOME/.nvm/nvm.sh"
```

- `.` (punto) — equivalente al comando `source`. Ejecuta el script en el contexto de la sesión actual, haciendo que los comandos de NVM estén disponibles inmediatamente sin necesidad de cerrar y reabrir la terminal.
- `"$HOME/.nvm/nvm.sh"` — ruta al script principal de NVM. `$HOME` es la variable que contiene el directorio personal del usuario actual.

Instalamos Node.js versión 22 (LTS):

```bash
nvm install 22
```

- `nvm install 22` — descarga e instala la versión 22 de Node.js. NVM gestiona automáticamente la descarga, instalación y configuración del PATH.
- La versión 22 es LTS (Long Term Support): recomendada para producción por su estabilidad y soporte garantizado por varios años.

Verificamos las instalaciones:

```bash
node -v
npm -v
```

- `node -v` — muestra la versión de Node.js instalada (debe ser `v22.x.x`).
- `npm -v` — muestra la versión de NPM (Node Package Manager), que se instala automáticamente junto con Node.js y se usa para gestionar las dependencias del proyecto.

![alt text](<Screenshot 2026-05-14 235047.png>)

---

### 6.3 Instalación de PM2

Instalamos PM2 de forma global:

```bash
npm install pm2@latest -g
```

- `npm install` — instala un paquete de Node.js.
- `pm2@latest` — instala siempre la versión más reciente del paquete `pm2`.
- `-g` — instalación global: PM2 se instala para todo el sistema y el comando `pm2` estará disponible desde cualquier directorio, no solo desde el proyecto actual.
- **PM2** (Process Manager 2) — gestor de procesos para aplicaciones Node.js en producción. Permite ejecutar aplicaciones en segundo plano, reiniciarlas automáticamente si fallan, ver logs en tiempo real y configurar el arranque automático al reiniciar el servidor.

![alt text](<Screenshot 2026-05-14 235239.png>)

---

### 6.4 Clonación y Configuración de la Aplicación

Creamos el directorio de trabajo y clonamos el repositorio:

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app1
```

- `mkdir ~/apps` — crea el directorio `apps` en el directorio personal del usuario. 
- `cd ~/apps` — entra al directorio recién creado.
- `git clone <url> app1` — descarga una copia completa del repositorio en la carpeta `app1`. Usamos ese nombre para identificar que es la primera instancia de la aplicación.

Instalamos las dependencias:

```bash
cd ~/apps/app1 && npm install
```

- `npm install` — lee el archivo `package.json` y descarga todas las dependencias del proyecto en la carpeta `node_modules/`. Sin este paso, la aplicación no puede ejecutarse.

Configuramos las variables de entorno:

```bash
cp .env.example .env
nano .env
```

![alt text](<Screenshot 2026-05-14 235445.png>)

- `cp .env.example .env` — crea una copia del archivo de ejemplo de variables de entorno. Los archivos `.env` contienen configuraciones sensibles (contraseñas, IPs) que no deben incluirse en el repositorio Git.
- `nano .env` — abrimos el archivo para editar los valores con los datos reales de nuestra instalación.

El archivo `.env` quedó con el siguiente contenido:

```env
PORT=3000
DB_HOST=192.168.104.5
DB_USER=usr_movies
DB_PASSWORD=secret
DB_NAME=db_movies
```

![alt text](<Screenshot 2026-05-14 235517.png>)

- `PORT=3000` — puerto en el que la aplicación Node.js escuchará. Debe coincidir con el puerto configurado en el `upstream` de Nginx (`192.168.104.3:3000`).
- `DB_HOST=192.168.104.5` — IP de la VM donde está instalada MariaDB. La aplicación se conectará a esta dirección para consultar y guardar datos.
- `DB_USER=usr_movies` — usuario de MariaDB con permisos sobre `db_movies`.
- `DB_PASSWORD=secret` — contraseña del usuario de MariaDB.
- `DB_NAME=db_movies` — nombre de la base de datos que la aplicación usará para almacenar los datos de películas.

---

### 6.5 Prueba Manual y Lanzamiento con PM2

Probamos que la aplicación funciona correctamente antes de usar PM2:

```bash
node app.js
```

![alt text](<Screenshot 2026-05-14 235639.png>)

- `node app.js` — ejecuta directamente el archivo principal de la aplicación. Debe mostrar `"Servidor ejecutándose en el puerto 3000"` y `"Conexión a MariaDB exitosa"`. Presionamos `Ctrl+C` para detenerla.

Lanzamos la aplicación con PM2 en segundo plano:

```bash
pm2 start app.js --name app1
```

![alt text](<Screenshot 2026-05-14 235711.png>)

- `pm2 start app.js` — inicia el archivo como un proceso gestionado por PM2, liberando la terminal. La aplicación corre en segundo plano.
- `--name app1` — asigna el nombre `app1` al proceso. Este nombre se usa para identificarlo en `pm2 status`, `pm2 stop app1`, `pm2 restart app1`, etc.

Configuramos el arranque automático:

```bash
pm2 startup
pm2 save
```

![alt text](<Screenshot 2026-05-14 235755.png>)

![alt text](<Screenshot 2026-05-14 235824.png>)

- `pm2 startup` — genera e instala un servicio systemd que iniciará PM2 y todos sus procesos automáticamente cuando el servidor arranque. Puede pedir ejecutar un comando adicional con `sudo`.
- `pm2 save` — guarda la lista actual de procesos gestionados en un archivo de configuración que se restaurará en cada arranque del sistema.

Verificamos el estado:

```bash
pm2 status
```

- Muestra una tabla con todos los procesos de PM2, su estado (`online`, `stopped`, `errored`), uso de CPU y memoria y número de reinicios.

---

### 6.6 Instalación de Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
curl http://localhost:9100/metrics
```

![alt text](<Screenshot 2026-05-14 235910.png>)

- `prometheus-node-exporter` — agente de métricas del sistema. Debe instalarse en **cada VM** que queremos que Prometheus monitoree. Expone sus métricas en el puerto `9100`.
- Si `curl` devuelve cientos de líneas de métricas, el agente está funcionando y Prometheus podrá recolectarlas desde la VM Proxy.

![alt text](<Screenshot 2026-05-14 235948.png>)

---

## 7. Configuración VM Aplicación 2 (192.168.104.4)

### 7.1 Configuración de Red con VLAN 104

La configuración de Netplan es idéntica a la de App 1, con una única diferencia:

```yaml
addresses:
  - "192.168.104.4/29"
```

- `192.168.104.4` — IP asignada a esta VM (Aplicación 2). El resto de la configuración (VLAN ID, gateway, DNS) es exactamente igual que en App 1.

```bash
sudo netplan apply
```

![alt text](<Screenshot 2026-05-15 000125.png>)

### 7.2 Instalación de Node.js, PM2 y la Aplicación

Se siguieron exactamente los mismos pasos que en la VM App 1 (secciones 6.2, 6.3 y 6.4). El archivo `.env` es idéntico:

```env
PORT=3000
DB_HOST=192.168.104.5
DB_USER=usr_movies
DB_PASSWORD=secret
DB_NAME=db_movies
```

- Ambas instancias usan el mismo puerto `3000` porque están en VMs diferentes con IPs distintas: no existe conflicto de puertos.
- Ambas se conectan a la misma base de datos en `192.168.104.5`. Esto garantiza que, sin importar a qué instancia llegue la petición del cliente, los datos devueltos serán siempre los mismos y consistentes.

```bash
pm2 start app.js --name app1
pm2 startup && pm2 save
```

![alt text](<Screenshot 2026-05-15 000252.png>)

![alt text](<Screenshot 2026-05-15 000316.png>)

### 7.3 Instalación de Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
```

![alt text](<Screenshot 2026-05-15 000348.png>)

![alt text](<Screenshot 2026-05-15 000415.png>)

![alt text](<Screenshot 2026-05-15 000433.png>)

![alt text](<Screenshot 2026-05-15 000504.png>)

![alt text](<Screenshot 2026-05-15 000528.png>)

![alt text](<Screenshot 2026-05-15 000551.png>)

![alt text](<Screenshot 2026-05-15 000618.png>)

![alt text](<Screenshot 2026-05-15 000641.png>)

![alt text](<Screenshot 2026-05-15 000723.png>)

![alt text](<Screenshot 2026-05-15 000739.png>)

![alt text](<Screenshot 2026-05-15 000804.png>)

![alt text](<Screenshot 2026-05-15 000826.png>)

![alt text](<Screenshot 2026-05-15 000844.png>)

![alt text](<Screenshot 2026-05-15 000905.png>)

---

## 8. Configuración VM Base de Datos (192.168.104.5)

### 8.1 Configuración de Red con VLAN 104

Misma estructura de Netplan, con la IP de esta VM:

```yaml
addresses:
  - "192.168.104.5/29"
```

![alt text](<Screenshot 2026-05-15 001009.png>)

```bash
sudo netplan apply
```

![alt text](<Screenshot 2026-05-15 001045.png>)

---

### 8.2 Instalación de MariaDB

```bash
sudo apt install mariadb-server -y
```

![alt text](<Screenshot 2026-05-15 001111.png>)

- `mariadb-server` — sistema de gestión de bases de datos relacionales. MariaDB es un fork comunitario de MySQL, completamente compatible con sus instrucciones SQL y con mejoras en rendimiento y licenciamiento libre.

---

### 8.3 Hardening de MariaDB

```bash
sudo mysql_secure_installation
```

![alt text](<Screenshot 2026-05-15 001141.png>)



- `mysql_secure_installation` — script interactivo que guía al administrador por varias configuraciones de seguridad básicas:
  - **Contraseña de root:** Establece una contraseña fuerte para el administrador de la base de datos.
  - **Eliminar usuarios anónimos:** Por defecto MariaDB tiene un usuario sin nombre que permite acceso sin autenticación. Esta opción lo elimina.
  - **Deshabilitar login remoto de root:** Impide que el usuario `root` se conecte desde otras máquinas; solo puede hacerlo localmente.
  - **Eliminar base de datos de prueba:** Elimina la base de datos `test` que cualquier usuario puede acceder por defecto.
  - **Recargar tablas de privilegios:** Aplica todos los cambios inmediatamente.

---

### 8.4 Configuración del Acceso Remoto

Editamos la configuración de MariaDB:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

![alt text](<Screenshot 2026-05-15 001241.png>)

- `/etc/mysql/mariadb.conf.d/50-server.cnf` — archivo de configuración principal del servidor MariaDB. Los archivos en este directorio se cargan en orden numérico al iniciar el servicio.

Buscamos y modificamos la línea `bind-address`:

```
# Antes:
bind-address = 127.0.0.1

# Después:
bind-address = 192.168.104.5
```

- `bind-address = 127.0.0.1` — configuración por defecto: MariaDB solo acepta conexiones desde la misma máquina (localhost). Las VMs de aplicación no pueden conectarse remotamente con esta configuración.
- `bind-address = 192.168.104.5` — cambiamos a la IP de la VLAN de esta VM. Ahora MariaDB escuchará conexiones entrantes en esa IP, permitiendo que App 1 y App 2 se conecten desde la red interna.

![alt text](<Screenshot 2026-05-15 001336.png>)

Reiniciamos para aplicar el cambio:

```bash
sudo systemctl restart mariadb
```

![alt text](<Screenshot 2026-05-15 001413.png>)

---

### 8.5 Creación de la Base de Datos y el Usuario

Accedemos al cliente de MariaDB como root:

```bash
mysql -u root -h localhost -p
```

![alt text](<Screenshot 2026-05-15 001526.png>)

![alt text](<Screenshot 2026-05-15 001554.png>)

- `mysql` — cliente de línea de comandos de MariaDB/MySQL.
- `-u root` — nos conectamos como usuario administrador `root`.
- `-h localhost` — conexión local a la misma máquina.
- `-p` — solicita la contraseña de forma interactiva, sin escribirla en el comando (más seguro).

```sql
CREATE DATABASE db_movies;
```

- `CREATE DATABASE db_movies` — crea la base de datos que usarán las aplicaciones Node.js para almacenar los datos de películas.

```sql
CREATE USER 'usr_movies'@'192.168.104.3' IDENTIFIED BY 'secret';
CREATE USER 'usr_movies'@'192.168.104.4' IDENTIFIED BY 'secret';
```

- `CREATE USER 'usr_movies'@'192.168.104.3'` — crea un usuario `usr_movies` que solo puede conectarse desde la IP `192.168.104.3` (App 1). Si intenta conectarse desde cualquier otra IP, MariaDB rechazará la conexión.
- Se necesitan crear dos usuarios (uno por cada IP de App) porque MariaDB trata `usr_movies@192.168.104.3` y `usr_movies@192.168.104.4` como usuarios completamente distintos.
- `IDENTIFIED BY 'secret'` — establece la contraseña del usuario.

```sql
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'192.168.104.3';
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'192.168.104.4';
```

- `GRANT ALL PRIVILEGES` — otorga todos los permisos (SELECT, INSERT, UPDATE, DELETE, CREATE, etc.).
- `ON db_movies.*` — los permisos aplican a todas las tablas (`*`) de la base de datos `db_movies`. El usuario no podrá acceder a otras bases de datos del servidor.
- `TO 'usr_movies'@'...'` — especifica a qué usuario se otorgan los permisos.

```sql
FLUSH PRIVILEGES;
quit
```

- `FLUSH PRIVILEGES` — recarga las tablas de privilegios para que los cambios surtan efecto inmediatamente sin reiniciar MariaDB.
- `quit` — sale del cliente de línea de comandos.


---

### 8.6 Creación de la Tabla e Inserción de Datos

Entramos con el nuevo usuario para verificar que funciona:

```bash
mysql -u usr_movies -h 192.168.104.5 -p
```

- `-h 192.168.104.5` — aunque estamos en la misma VM, usamos la IP de la VLAN para verificar que la conexión remota funciona correctamente con el bind-address que configuramos.

```sql
USE db_movies;
```

- `USE db_movies` — selecciona `db_movies` como la base de datos activa. Todos los comandos siguientes operarán sobre ella.

```sql
CREATE TABLE movies (
    id serial PRIMARY KEY,
    title character varying(150) NOT NULL,
    year integer,
    UNIQUE(title)
);
```

- `id serial PRIMARY KEY` — columna identificadora autoincrementable. MariaDB asigna automáticamente un número único a cada registro nuevo sin que la aplicación tenga que calcularlo.
- `title character varying(150) NOT NULL` — columna de texto variable (máximo 150 caracteres) que no puede estar vacía. Impide insertar películas sin título.
- `year integer` — columna numérica para el año de estreno de la película.
- `UNIQUE(title)` — restricción que garantiza que no pueden existir dos películas con el mismo título en la base de datos.

```sql
INSERT INTO movies (title, year) VALUES
('Inception', 2010),
('The Matrix', 1999),
('Pulp Fiction', 1994),
('The Dark Knight', 2008),
('Eternal Sunshine of the Spotless Mind', 2004),
('Forrest Gump', 1994),
('Fight Club', 1999),
('The Godfather', 1972),
('Interstellar', 2014),
('Parasite', 2019);
```

- `INSERT INTO movies (title, year) VALUES (...)` — inserta múltiples registros en una sola instrucción. La columna `id` se omite porque es autoincrementable y MariaDB la asigna automáticamente.

![alt text](<Screenshot 2026-05-15 001647.png>)

![alt text](<Screenshot 2026-05-15 001741.png>)

---

### 8.7 Configuración del Firewall UFW

```bash
sudo apt install ufw -y
```
![alt text](<Screenshot 2026-05-15 001947.png>)

- `ufw` (Uncomplicated Firewall) — herramienta de firewall simplificada de Ubuntu que facilita la gestión de reglas de filtrado de tráfico sin necesidad de conocer la sintaxis compleja de `iptables`.

```bash
sudo ufw allow from 192.168.104.3 to any port 3306
sudo ufw allow from 192.168.104.4 to any port 3306
sudo ufw allow from 192.168.104.2 to any port 9100
sudo ufw allow 22
sudo ufw enable
```

- `ufw allow from 192.168.104.3 to any port 3306` — permite conexiones al puerto `3306` (MariaDB) solo desde la IP `192.168.104.3` (App 1). Cualquier intento desde otra IP será rechazado automáticamente.
- `ufw allow from 192.168.104.4 to any port 3306` — mismo permiso para App 2 (`192.168.104.4`).
- `ufw allow from 192.168.104.2 to any port 9100` — permite que Prometheus (`192.168.104.2`) recolecte métricas del Node Exporter por el puerto `9100`.
- `ufw allow 22` — permite conexiones SSH desde cualquier IP. Sin esta regla, al activar UFW se perdería el acceso remoto a la VM.
- `ufw enable` — activa el firewall. A partir de este momento solo se permite el tráfico que coincida con las reglas; todo lo demás es bloqueado.

![alt text](<Screenshot 2026-05-15 002019.png>)

---

### 8.8 Instalación de Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
curl http://localhost:9100/metrics
```

![alt text](<Screenshot 2026-05-15 002110.png>)

![alt text](<Screenshot 2026-05-15 002159.png>)

---

## 9. Flujo de una Petición en el Sistema

```
Cliente (Navegador)      VM Proxy (Nginx)      VM App 1 o App 2       VM DB (MariaDB)
        │                       │                      │                      │
        │  1. GET /movies        │                      │                      │
        │  vlan104-app.root...   │                      │                      │
        │──────────────────────►│                      │                      │
        │                       │  2. Round-Robin:     │                      │
        │                       │     elige App 1 o 2  │                      │
        │                       │─────────────────────►│                      │
        │                       │                      │  3. SELECT *         │
        │                       │                      │     FROM movies      │
        │                       │                      │─────────────────────►│
        │                       │                      │  4. Devuelve datos   │
        │                       │                      │◄─────────────────────│
        │                       │  5. JSON con movies  │                      │
        │                       │◄─────────────────────│                      │
        │  6. Responde al cliente│                      │                      │
        │◄──────────────────────│                      │                      │
```

---

## 10. Pruebas y Verificación

### 10.1 Verificación de Conectividad de la VLAN

```bash
ping 192.168.104.3   # Desde VM Proxy hacia App 1
ping 192.168.104.5   # Desde VM App 1 hacia DB
```

![alt text](<Screenshot 2026-05-15 002328.png>)

---

### 10.2 Verificación del Balanceo de Carga

```bash
curl https://vlan104-app.rootcode.com.bo
curl https://vlan104-app.rootcode.com.bo
curl https://vlan104-app.rootcode.com.bo
curl https://vlan104-app.rootcode.com.bo
```

![alt text](<Screenshot 2026-05-15 002406.png>)

Resultados obtenidos:

| Petición | Hostname en Respuesta | Servidor que atendió |
|----------|----------------------|----------------------|
| 1ra | server-165 | App 1 (192.168.104.3) |
| 2da | server-164 | App 2 (192.168.104.4) |
| 3ra | server-165 | App 1 (192.168.104.3) |
| 4ta | server-164 | App 2 (192.168.104.4) |

- El hostname diferente en respuestas alternadas confirma que Nginx está distribuyendo las peticiones usando Round-Robin entre App 1 y App 2.

![alt text](<Screenshot 2026-05-15 002503.png>)

![alt text](<Screenshot 2026-05-15 002525.png>)

---

### 10.3 Verificación de Conexión a la Base de Datos

```bash
curl https://vlan104-app.rootcode.com.bo/movies
```

- `/movies` — endpoint que consulta la tabla `movies` en MariaDB y devuelve los registros en JSON. Si responde con la lista de películas, toda la cadena funciona: Nginx → App → MariaDB.

![alt text](<Screenshot 2026-05-15 002554.png>)


---

### 10.4 Demostración de Failover (Alta Disponibilidad)

#### Paso 1 — Estado inicial: ambas apps activas

```bash
curl https://vlan104-app.rootcode.com.bo   # → server-165
curl https://vlan104-app.rootcode.com.bo   # → server-164
```

![alt text](<Screenshot 2026-05-15 083403.png>)

![alt text](<Screenshot 2026-05-15 083412.png>)


#### Paso 2 — Detener App 1 simulando un fallo

```bash
pm2 stop app1
```

- `pm2 stop app1` — detiene el proceso `app1`. Simula que el servidor de aplicaciones tuvo un fallo o fue reiniciado para mantenimiento.

![alt text](<Screenshot 2026-05-15 085122.png>)

#### Paso 3 — Verificar continuidad del servicio

```bash
curl https://vlan104-app.rootcode.com.bo
curl https://vlan104-app.rootcode.com.bo
```

- Nginx detecta automáticamente que App 1 no responde y redirige todo el tráfico a App 2. El cliente no experimenta ningún error ni interrupción.

![alt text](<Screenshot 2026-05-15 083107.png>)

![alt text](<Screenshot 2026-05-15 083053.png>)

![alt text](<Screenshot 2026-05-15 084352.png>)

#### Paso 4 — Restaurar App 1

```bash
pm2 start app1
```

![alt text](<Screenshot 2026-05-15 085213.png>)

- Nginx detecta que App 1 está disponible nuevamente y retoma el balanceo Round-Robin automáticamente.

![alt text](<Screenshot 2026-05-15 083345.png>)

![alt text](<Screenshot 2026-05-15 083403-1.png>)

![alt text](<Screenshot 2026-05-15 083412-1.png>)


---

## 11. Tabla Resumen de Servicios y Puertos

| VM | IP | Servicio | Puerto | Acceso Permitido Desde |
|----|-----|---------|--------|------------------------|
| Proxy | 192.168.104.2 | Nginx (HTTP) | 80 | Internet (via docente) |
| Proxy | 192.168.104.2 | Grafana | 3000 | Internet (via docente) |
| Proxy | 192.168.104.2 | Prometheus | 9090 | Local (localhost) |
| Proxy | 192.168.104.2 | Node Exporter | 9100 | 192.168.104.2 (Prometheus) |
| App 1 | 192.168.104.3 | Node.js API | 3000 | 192.168.104.2 (Nginx) |
| App 1 | 192.168.104.3 | Node Exporter | 9100 | 192.168.104.2 (Prometheus) |
| App 2 | 192.168.104.4 | Node.js API | 3000 | 192.168.104.2 (Nginx) |
| App 2 | 192.168.104.4 | Node Exporter | 9100 | 192.168.104.2 (Prometheus) |
| DB | 192.168.104.5 | MariaDB | 3306 | 192.168.104.3 y 192.168.104.4 |
| DB | 192.168.104.5 | Node Exporter | 9100 | 192.168.104.2 (Prometheus) |

---

## 12. Conclusiones

- **VLAN como mecanismo de segmentación:** La configuración de la VLAN 104 con Netplan permitió aislar el tráfico del grupo dentro de la subred `192.168.104.0/29`. Las VLANs son fundamentales en entornos de Centro de Datos para separar el tráfico de diferentes equipos o proyectos que comparten la misma infraestructura física.

- **Proxy inverso y balanceo Round-Robin:** Nginx distribuyó el tráfico de manera equitativa entre las dos instancias de la aplicación usando el algoritmo Round-Robin por defecto. Este enfoque es simple y efectivo para cargas de trabajo uniformes, y Nginx detecta automáticamente cuándo un servidor backend deja de responder.

- **Segregación de servicios:** Cada rol de la arquitectura corre en una VM dedicada (proxy, app, base de datos). Esta separación aísla los fallos, facilita el escalado independiente de cada capa y mejora la seguridad al reducir la superficie de ataque de cada servidor.

- **PM2 para producción:** El uso de PM2 garantizó que las aplicaciones Node.js se mantengan activas y se reinicien automáticamente si fallan. La configuración de `pm2 startup` asegura que los procesos sobrevivan a reinicios del servidor.

- **Seguridad en capas con UFW:** La restricción de acceso a MariaDB mediante UFW aplicó el principio de mínimo privilegio: solo las IPs de las VMs de aplicación pueden conectarse al puerto 3306. Un atacante que comprometiera el proxy no podría acceder directamente a la base de datos.

- **Observabilidad con Prometheus y Grafana:** La integración del stack de monitoreo proporcionó visibilidad completa de toda la plataforma. Poder observar métricas de CPU, memoria, disco y red de las cuatro VMs desde un único dashboard facilita la detección temprana de problemas antes de que afecten a los usuarios.

- **Alta Disponibilidad demostrada:** La prueba de failover confirmó que la arquitectura cumple con el objetivo principal: al detener una instancia de la aplicación, Nginx redirigió automáticamente todo el tráfico a la instancia disponible sin interrupciones. Al restaurar la instancia caída, el balanceo se retomó automáticamente sin intervención manual.

---