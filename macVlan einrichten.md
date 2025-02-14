> [!CAUTION]
> Funtioniert nicht mit dem WLAN-Adapter!

Orignial-Anleitung hier:
https://blog.ivansmirnov.name/set-up-pihole-using-docker-macvlan-network/

IP-Subnetmask-Rechner:
https://www.calculator.net/ip-subnet-calculator.html

## 1. Schritt - macVlan im Docker erzeugen:
`docker network create -d macvlan \ `-> netzwerk erstellen mit Driver "macvlan" <br/>
`  -o parent=eth0 \`-> welcher adapter? auslesen über "ifconfig" <br/>
`  --subnet 10.0.0.0/24 \`-> adressbereich vom normalen Netzwerk <br/>
`  --gateway 10.0.0.1 \`-> gateway vom normalen Netzwerk <br/>
`  --ip-range 10.0.0.16/28 \`-> adressbereich vom macVlan, IP-Subnetmask-Rechner, im Beispiel sind dann die IP-Adressen von .97-126 für die Container frei zu verwenden <br/>
`  --aux-address 'host=10.0.0.30' \`-> adresse die im macVlan nicht verwendet wird und am Host als Brigde eingetragen wird, damit das macVlan den Host pingen kann <br/>
`  macVlan` -> Name des Netzwerkes

zum kopieren:
```
docker network create -d macvlan \
  -o parent=eth0 \
  --subnet 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  --ip-range 10.0.0.16/28 \
  --aux-address 'host=10.0.0.30' \
  macVlan
```

## 2. Schritt - Host-Adapter damit alle Pakete akzeptiert werden, nicht nur für die Host-MAC-Adresse
`sudo ip link set eth0 promisc on`

## 3. Schritt - Schnittstelle am Host erzeugen:
`sudo ip link add NAME link ADAPTER type macvlan  mode bridge`  
Name: Name der Schnittstelle zb. macVlanAdapter  
Adapter: Welcher Adapter am Host? zb. wlan0, eth0; auslesen über "ifconfig"

## 4. Schritt - der Schnittstellen am Host eine IP zuweisen:
`sudo ip addr add 10.0.0.30/28 dev NAME`  
IP: vom 1. Schritt die "aux-adress";  
NAME: vom 2. Schritt der NAME

## 5. Schritt - die Schnittstelle am Host aktiviern
`sudo ip link set NAME up`  
NAME: vom 2. Schritt der NAME

## 6. Schritt - Route hinzufügen:
`sudo ip route add 10.0.0.16/28 dev NAME`  
IP: Adressbereich vom macVlan  
NAME: vom 2. Schritt der NAME  

Wenn das nicht funktioniert, dann `sudo ip addr flush dev macVlan-Adapter` bzw. `ip route` und überprüfen ob Route schon eingetragen

## 7. Schritt - im Container verwenden:
MAC-Adresse eintragen  
IP-Adresse vergeben  
Netzwerk eintragen  

Beispiel:  
```
services:
  iobroker:
    container_name: iobroker
    hostname: iobroker
    mac_address: 2c:cf:67:00:00:01
    image: buanet/iobroker
    networks:
      macVlan:
        ipv4_address: 10.0.0.20
    restart: unless-stopped
    ports:
      - 8081:8081
    volumes:
      - /mnt/nvme/docker/volumes/iobroker:/opt/iobroker
networks:
  macVlan:
    external: true
```

## 8. Schritt - Host-Netzwerkeinstellungen dauerhaft laden:
Skript unter `/usr/local/bin/` erstellen und Rechte vergeben: <br/>
`sudo touch /usr/local/bin/macVlan.sh` <br/>
`sudo chmod +x  /usr/local/bin/macVlan.sh` <br/>
`sudo nano /usr/local/bin/macVlan.sh` <br/>
Inhalt Skript: <br/>
```
#!/usr/bin/env bash
ip link add macVlanAdapter link eth0 type macvlan mode bridge
ip addr add 10.0.0.30/28 dev macVlanAdapter
ip link set macVlanAdapter up
ifconfig macVlanAdapter
```
Autostart einrichten: <br/>
`sudo touch /etc/systemd/system/macVlan.service` <br/>
`sudo nano /etc/systemd/system/macVlan.service` <br/>
Inhalt Service: <br/>
```
[Unit]
After=network.target

[Service]
ExecStart=/usr/local/bin/pi-vlan.sh

[Install]
WantedBy=default.target
```
