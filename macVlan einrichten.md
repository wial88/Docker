ACHTUNG: funtioniert nicht mit dem WLAN-Adapter!

Orignial-Anleitung hier:
https://blog.ivansmirnov.name/set-up-pihole-using-docker-macvlan-network/

IP-Subnetmask-Rechner:
https://www.calculator.net/ip-subnet-calculator.html

# 1. Schritt - macVlan im Docker erzeugen:
docker network create -d macvlan \     "netzwerk erstellen"
  -o parent=eth0 \                     "welcher adapter? wlan0? eth0?" auslesen über "ifconfig"
  --subnet 192.168.0.0/24 \            "adressbereich vom normalen Netzwerk"
  --gateway 192.168.0.1 \              "gateway vom normalen Netzwerk"
  --ip-range 192.168.0.96/27 \        "adressbereich vom macVlan, IP-Subnetmask-Rechner, im Beispiel sind dann die IP-Adressen von .97-126 für die Container frei zu verwenden"
  --aux-address 'host=192.168.0.126' \ "adresse die im macVlan nicht verwendet wird und am Host als Brigde eingetragen wird, damit das macVlan den Host pingen kann"
  macVlan

zum kopieren:
docker network create -d macvlan \
  -o parent=eth0 \
  --subnet 192.168.0.0/24 \
  --gateway 192.168.0.1 \
  --ip-range 192.168.0.96/27 \
  --aux-address 'host=192.168.0.126' \
  macVlan

# 2. Schritt - Parent-Adapter damit alle Parkete akzeptiert werden, nicht nur für die Host-MAC-Adresse
sudo ip link set eth0 promisc on

# 3. Schritt - Schnittstelle am Host erzeugen:
sudo ip link add NAME link ADAPTER type macvlan  mode bridge
Name: Name der Schnittstelle zb. macVlanAdapter
Adapter: Welcher Adapter am Host? zb. wlan0, eth0; auslesen über "ifconfig"

# 4. Schritt - der Schnittstellen am Host eine IP zuweisen:
sudo ip addr add 192.168.0.126/27 dev NAME
IP: vom 1. Schritt die "aux-adress"; wichtig /32!
NAME: vom 2. Schritt der NAME

# 5. Schritt - die Schnittstelle am Host aktiviern
sudo ip link set NAME up
NAME: vom 2. Schritt der NAME

# 6. Schritt - Route hinzufügen:
sudo ip route add 192.168.0.96/27 dev NAME
IP: Adressbereich vom macVlan
NAME: vom 2. Schritt der NAME

Wenn das nicht funktioniert, dann: sudo ip addr flush dev macVlan-Adapter

# 7. Schritt - im Container verwenden:
MAC-Adresse eintragen
IP-Adresse vergeben
Netzwerk eintragen

Beispiel:
services:
  iobroker:
    container_name: iobroker
    hostname: iobroker
    mac_address: 2c:cf:67:00:00:01
    image: buanet/iobroker
    networks:
      macVlan:
        ipv4_address: 192.168.0.241
    restart: unless-stopped
    ports:
      - 8081:8081
    volumes:
      - /mnt/nvme/docker/volumes/iobroker:/opt/iobroker
networks:
  macVlan:
    external: true

# 8. Schritt - Host-Netzwerkeinstellungen dauerhaft laden:
Skript unter /usr/local/bin/ erstellen und Rechte vergeben:
sudo touch /usr/local/bin/macVlan.sh
sudo chmod +x  /usr/local/bin/macVlan.sh
sudo nano /usr/local/bin/macVlan.sh
------------------------------------------
#!/usr/bin/env bash
ip link add macVlanAdapter link eth0 type macvlan mode bridge
ip addr add 192.168.0.126/27 dev macVlanAdapter
ip link set macVlanAdapter up
ifconfig macVlanAdapter
------------------------------------------
Autostart einrichten:
sudo touch /etc/systemd/system/macVlan.service
sudo nano /etc/systemd/system/macVlan.service
------------------------------------------
[Unit]
After=network.target

[Service]
ExecStart=/usr/local/bin/pi-vlan.sh

[Install]
WantedBy=default.target
------------------------------------------
