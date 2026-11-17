# Análisis de la red con `arp` y `nmap`

**Unidad:** U3.- Monitorización  
**Sesión/práctica:** Análisis de red local

---

## 1.- Identificar la configuración de red

### 1.- Indica la IP y la interfaz de red de tu equipo
- **Interfaz de red principal:** `eno1`
- **Dirección IP:** `172.26.12.104`

Comprobado con `ip -br addr`:
```text
eno1             UP             172.26.12.104/16 fe80::f6b5:20ff:fe40:51a1/64
```

### 2.- Anota la puerta de enlace predeterminada
- **Puerta de enlace predeterminada:** `172.26.0.1` (a través de `eno1`)

Comprobado con `ip route`:
```text
default via 172.26.0.1 dev eno1 proto dhcp src 172.26.12.104 metric 100
```

### 3.- Indica la dirección de red y su prefijo o máscara
- **Dirección de red:** `172.26.0.0`
- **Prefijo:** `/16`
- **Máscara de red:** `255.255.0.0`

---

## 2.- Analizar y renovar una entrada ARP

### 1.- IP del ordenador del profesor y comprobación de red local IPv4
- **IP del profesor:** `IP_PROFESOR` *(pendiente de sustituir por la IP que facilite el profesor)*.
- **Comprobación:** Como nuestra red es `172.26.0.0/16` (máscara `255.255.0.0`), cualquier IP que empiece por **`172.26.X.Y`** (entre `172.26.0.1` y `172.26.255.254`) pertenece a la misma red local IPv4.

### 2.- Ejecuta los siguientes comandos e indica qué hace cada uno

#### `arp -a`
Muestra la tabla ARP actual del sistema en formato BSD, intentando resolver los nombres de host (DNS inverso) junto a su dirección IP, dirección MAC y la interfaz asociada.
```text
argos.rodrigocaro.fw (172.26.0.1) en 64:62:66:25:50:b0 [ether] en eno1
```

#### `arp -n`
Muestra la misma tabla ARP pero en formato numérico (`-n`), es decir, **sin resolver nombres de host**, lo que hace que sea más rápido y muestre directamente las direcciones IP.
```text
Dirección                TipoHW  DirecciónHW         Indic Máscara         Interfaz
172.26.0.1               ether   64:62:66:25:50:b0   C                     eno1
```

#### `ip neigh`
Comando moderno de la suite `iproute2` que muestra la tabla de vecinos (caché ARP en IPv4 y NDP en IPv6), indicando la IP, la interfaz (`dev`), la dirección MAC (`lladdr`) y el estado de la entrada (por ejemplo, `REACHABLE`, `STALE`, `FAILED`).
```text
172.26.0.1 dev eno1 lladdr 64:62:66:25:50:b0 REACHABLE
```

### 3.- ¿Qué MAC aparece asociada a la puerta de enlace? ¿Por qué la tabla ARP no muestra todos los equipos de la red?
- **MAC asociada a la puerta de enlace (`172.26.0.1`):** `64:62:66:25:50:b0` (interfaz `eno1`).
- **¿Por qué no muestra todos los equipos de la red?**  
  Porque la tabla ARP funciona como una **caché dinámica**: el sistema operativo solo aprende y guarda las asociaciones IP-MAC de los equipos con los que se ha comunicado recientemente. Si no ha habido tráfico previo con otros equipos de la red, no aparecen en la tabla (y además las entradas caducan pasado un tiempo si no se utilizan).

### 4.- Genera y consulta la entrada del profesor
Sustituyendo `IP_PROFESOR` por la IP real del profesor:
```bash
ping -c 2 IP_PROFESOR
ip neigh show IP_PROFESOR
```
- `ping -c 2 IP_PROFESOR` envía 2 paquetes ICMP Echo Request; para poder enviarlos en la red local, el equipo resuelve primero la MAC mediante ARP y la guarda en la caché.
- `ip neigh show IP_PROFESOR` muestra la entrada recién generada con su dirección MAC y estado (`REACHABLE`).

### 5.- Elimina únicamente esa asociación en tu ordenador
Sustituyendo `IP_PROFESOR` por la IP del profesor y `eno1` como interfaz:
```bash
sudo ip neigh del IP_PROFESOR dev eno1
ip neigh show IP_PROFESOR
```
*(Nota: `ip neigh del` requiere privilegios de superusuario con `sudo`).*  
Al ejecutar `ip neigh show IP_PROFESOR` inmediatamente después, la entrada aparece eliminada (o en estado `FAILED`/vacía hasta que vuelva a producirse tráfico).

### 6.- Si no se genera automáticamente, puedes hacerlo:
```bash
ping -c 2 IP_PROFESOR
ip neigh show IP_PROFESOR
```
Al volver a hacer `ping`, el sistema lanza de nuevo una petición ARP y regenera la entrada en la tabla de vecinos con la MAC del profesor.

---

## 3.- Descubrir equipos con Nmap

### 1.- Escanea desde la `172.26.20.0` hasta `172.26.20.255` con `nmap -sn`
```bash
nmap -sn 172.26.20.0-255
```
*(Nota: en el enunciado aparece `172.26.0-255`, pero para el rango indicado `172.26.20.0`–`172.26.20.255` el comando correcto es `nmap -sn 172.26.20.0-255` o `nmap -sn 172.26.20.0/24`).*

**Resultado obtenido:**
```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-17 10:43 +0100
Nmap scan report for 172.26.20.3
Host is up (0.00039s latency).
Nmap scan report for 172.26.20.6
Host is up (0.00043s latency).
Nmap done: 256 IP addresses (2 hosts up) scanned in 4.01 seconds
```
Se han detectado **2 equipos activos** (`172.26.20.3` y `172.26.20.6`) de las 256 direcciones IP analizadas.

### 2.- ¿Qué hace exactamente la opción `-sn`?
La opción **`-sn`** (*Ping Scan* / descubrimiento de hosts sin escaneo de puertos) indica a `nmap` que **únicamente compruebe qué equipos están activos (encendidos/responden)** en la red o rango indicado, **sin realizar ningún escaneo de puertos** sobre ellos.

---

## 4.- Examinar puertos y servicios

### 1.- Escanea exclusivamente la máquina objetivo indicada por el profesor
Sustituyendo `IP_OBJETIVO` por la IP de la máquina objetivo:

```bash
nmap IP_OBJETIVO
```
Realiza un escaneo básico de los **1000 puertos TCP más comunes** para detectar cuáles están abiertos, cerrados o filtrados, y muestra el servicio estándar asociado a cada puerto.

```bash
nmap -sV IP_OBJETIVO
```
Escanea los puertos abiertos y además intenta identificar el **servicio exacto y su versión** que está escuchando en cada puerto.

#### ¿Qué hace la opción `-sV`?
La opción **`-sV`** (*Service/Version detection*) activa la **detección de versiones**: `nmap` se conecta a los puertos abiertos encontrados, envía sondas específicas y analiza las respuestas o *banners* para averiguar qué aplicación y versión concreta está ejecutando ese servicio (por ejemplo, `OpenSSH 8.9p1`, `Apache httpd 2.4.52`, etc.).
