# Configuration Proxmox : NAT & Serveur DHCP

## 🔹 Activer le Forwarding  
Il suffit de **uncomment** la ligne suivante dans `/etc/sysctl.conf` :  
```ini
net.ipv4.ip_forward=1
```
Puis appliquer les changements avec :  
```bash
sysctl -p
```

---

## 🔹 Configuration du NAT  
Ajout des `post-up` et `post-down` pour permettre à `vmbr1` de communiquer avec le network via `vmbr0`.  
Fichier à edit : `/etc/network/interfaces`

```ini
auto lo
iface lo inet loopback

iface eno1 inet manual
iface eno2 inet manual

auto vmbr0
iface vmbr0 inet static
    address 145.239.64.15/24
    gateway 145.239.64.254
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    hwaddress A4:BF:01:46:BF:4E

iface vmbr0 inet6 static
    address 2001:41d0:303:2f0f::1/128
    gateway 2001:41d0:303:2fff:ff:ff:ff:ff

auto vmbr1
iface vmbr1 inet static
    address 192.168.1.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    # Active l'IP forwarding pour que le NAT fonctionne
    post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
    # Ajoute la règle NAT pour permettre aux VMs d'accéder à Internet
    post-up   iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o vmbr0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s 192.168.1.0/24 -o vmbr0 -j MASQUERADE
    # Ouvre les ports et redirections si nécessaire
    post-up   iptables -A FORWARD -i vmbr1 -o vmbr0 -j ACCEPT
    post-up   iptables -A FORWARD -i vmbr0 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
    post-down iptables -D FORWARD -i vmbr1 -o vmbr0 -j ACCEPT
    post-down iptables -D FORWARD -i vmbr0 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Après l'edit, appliquer la configuration réseau :
```bash
systemctl restart networking
```

---

## 🔹 Configuration du serveur DHCP  
### 📌 edit `/etc/dhcp/dhcpd.conf`  
```ini
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;  # IP attribuées dynamiquement
    option routers 192.168.1.1;         # Passerelle (IP de `vmbr1` sur Proxmox)
    option domain-name-servers 8.8.8.8, 8.8.4.4;  # DNS Google
    option broadcast-address 192.168.1.255;
    default-lease-time 600;
    max-lease-time 7200;
}
```

### 📌 edit `/etc/default/isc-dhcp-server`  
Remplacer cette ligne :
```ini
INTERFACESv4=""
```
par :
```ini
INTERFACESv4="vmbr1"
```

### 📌 Démarrer et activer le serveur DHCP  
```bash
systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
```
Si tout est ok, les VMs ou CTs doivent récup une **IP dans la plage 192.168.1.100-192.168.1.200**
