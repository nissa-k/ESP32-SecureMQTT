# ESP32-SecureMQTT — Accès Distant Sécurisé via MQTTs/TLS

> **Auteur :** Nissa Karadag  
> **Contexte :** Travaux Pratiques — IoT Sécurisé  

---

## Présentation

Ce dépôt documente la mise en place d'une **infrastructure IoT sécurisée** permettant à un microcontrôleur **ESP32 (MicroPython)** de communiquer avec un serveur central via le protocole **MQTT over TLS (MQTTs)**, sans exposer directement l'objet sur Internet.

L'objectif est de répondre aux trois contraintes fondamentales de sécurité :

| Besoin | Solution retenue |
|--------|-----------------|
| Accès distant à l'ESP32 | Broker MQTT (Mosquitto) sur VM Ubuntu |
| Communication chiffrée | TLS 1.2+ via certificats auto-signés |
| Authentification & contrôle d'accès | Auth username/password + ACL |

---

## Architecture

```
ESP32 (MicroPython)
    │
    │  Connexion sortante MQTTs (port 8883, TLS)
    │  Authentification : user/password
    │  ACL : topic esp32/123/#
    ▼
VM Ubuntu (192.168.1.183)
    ├── Broker Mosquitto (port 8883, TLS only)
    ├── Certificats : CA auto-signée + certificat serveur
    ├── Firewall UFW (deny incoming, allow 22 & 8883)
    └── SSH (port 22) ← Administration distante
    ▲
    │  Subscribe/Publish (mosquitto_sub / mosquitto_pub)
    │
PC Windows (client de test / administrateur)
```

---

## Structure du dépôt

```
ESP32-SecureMQTT/
├── README.md                    ← Ce fichier
├── COMPTE_RENDU.md              ← Rapport technique complet
├── src/
│   └── main.py                  ← Code MicroPython ESP32
├── config/
│   └── secure.conf              ← Configuration Mosquitto TLS
├── docs/
│   └── screenshots/             ← Captures des étapes
└── scripts/
    └── commandes_reference.sh   ← Toutes les commandes utilisées
```

---

## ⚙️ Prérequis

### Côté serveur (VM Ubuntu)
- Ubuntu 22.04+ (mode bridge VirtualBox)
- `mosquitto` + `mosquitto-clients`
- `openssl`
- `ufw`

### Côté client (ESP32)
- ESP32 avec firmware **MicroPython v1.20+**
- Bibliothèque `umqtt.simple`
- Certificat `ca.crt` copié sur l'ESP32

### Côté poste Windows (test)
- [Mosquitto for Windows](https://mosquitto.org/download/)
- [Thonny IDE](https://thonny.org/)

---

## Déploiement rapide

### 1. Configurer la VM Ubuntu

```bash
# Mise à jour système
apt update && apt upgrade -y

# Installation SSH
apt install -y openssh-server

# Installation Firewall
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 8883/tcp
ufw enable

# Installation Mosquitto
apt install -y mosquitto mosquitto-clients
systemctl enable mosquitto
```

### 2. Générer les certificats TLS

```bash
mkdir -p /etc/mosquitto/certs && cd /etc/mosquitto/certs

# CA (Autorité de Certification)
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt

# Certificat serveur
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out server.crt -days 3650 -sha256

# Permissions
chown -R mosquitto:mosquitto /etc/mosquitto/certs
chmod 600 /etc/mosquitto/certs/server.key
chmod 644 /etc/mosquitto/certs/*.crt
```

### 3. Configurer Mosquitto TLS

Fichier `/etc/mosquitto/conf.d/secure.conf` :

```conf
# Désactiver MQTT en clair
port 0

# MQTT sécurisé (TLS)
listener 8883
cafile /etc/mosquitto/certs/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile /etc/mosquitto/certs/server.key

allow_anonymous false
password_file /etc/mosquitto/passwd
acl_file /etc/mosquitto/acl
```

### 4. Créer un utilisateur et ses droits (ACL)

```bash
# Créer l'utilisateur
mosquitto_passwd -c /etc/mosquitto/passwd esp32

# Fichier ACL
echo "user esp32" > /etc/mosquitto/acl
echo "topic readwrite esp32/123/#" >> /etc/mosquitto/acl

# Permissions
chown mosquitto:mosquitto /etc/mosquitto/passwd /etc/mosquitto/acl
chmod 600 /etc/mosquitto/passwd /etc/mosquitto/acl

systemctl restart mosquitto
```

### 5. Déployer sur l'ESP32

Copier `ca.crt` sur l'ESP32 via Thonny (Affichage → Fichiers), puis flasher `src/main.py`.

---

## Tests

### Depuis Windows (avant ESP32)

**Terminal 1 — Subscribe (récepteur) :**
```powershell
.\mosquitto_sub.exe -h 192.168.1.183 -p 8883 --cafile C:\Users\<user>\ca.crt `
    -u esp32 -P "esp32" -t esp32/123/status -v
```

**Terminal 2 — Publish (émetteur) :**
```powershell
.\mosquitto_pub.exe -h 192.168.1.183 -p 8883 --cafile C:\Users\<user>\ca.crt `
    -u esp32 -P "esp32" -t esp32/123/status -m "Test MQTTs OK"
```

### Vérifier les logs Mosquitto

```bash
sudo tail -f /var/log/mosquitto/mosquitto.log
```

---

## Sécurité — Points clés

| Fichier | Rôle | Confidentialité |
|---------|------|----------------|
| `ca.key` | Clé privée de la CA | Ne quitte JAMAIS le serveur |
| `ca.crt` | Certificat public de la CA | Copié sur les clients (ESP32, PC) |
| `server.key` | Clé privée du broker | Reste sur le serveur |
| `server.crt` | Certificat signé du broker | Utilisé par Mosquitto |

---

## 📋 Code MicroPython (ESP32)

```python
import network, time, ntptime
from umqtt.simple import MQTTClient

SSID      = "votre_reseau"
WIFI_PWD  = "votre_mdp"
BROKER    = "192.168.1.183"
PORT      = 8883
CLIENT_ID = "esp32_123"
TOPIC     = b"esp32/123/status"

# Connexion Wi-Fi
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(SSID, WIFI_PWD)
while not wlan.isconnected():
    time.sleep(1)

# Synchronisation NTP (obligatoire pour TLS)
ntptime.settime()
time.sleep(2)

# Connexion MQTTs (TLS)
client = MQTTClient(
    client_id=CLIENT_ID,
    server=BROKER,
    port=PORT,
    user="esp32",
    password="esp32",
    ssl=True,
    keepalive=60
)
client.connect()
client.publish(TOPIC, b"ESP32 CONNECTED TLS OK")
```

>  La synchronisation NTP est **obligatoire** avant la connexion TLS — la validation des certificats dépend de l'horloge système.

---

## Résultats obtenus

- [x] VM Ubuntu accessible en SSH depuis Windows
- [x] Firewall UFW actif (ports 22 et 8883 uniquement)
- [x] Broker Mosquitto opérationnel sur port 8883 (TLS)
- [x] Certificats CA et serveur générés et vérifiés (valides 10 ans)
- [x] Authentification par username/password fonctionnelle
- [x] ACL restreignant l'accès aux topics esp32/123/#
- [x] Communication chiffrée validée depuis Windows (publish/subscribe)
- [x] ESP32 connecté au broker en MQTTs depuis MicroPython
- [x] Message `ESP32 CONNECTED TLS OK` reçu côté client Windows

---

## Ressources

- [Mosquitto Documentation](https://mosquitto.org/documentation/)
- [MicroPython umqtt](https://github.com/micropython/micropython-lib/tree/master/micropython/umqtt.simple)
- [OpenSSL Cookbook](https://www.feistyduck.com/library/openssl-cookbook/)
- [Thonny IDE](https://thonny.org/)

---

*TP réalisé dans le cadre d'un projet IoT sécurisé — Nissa Karadag, 2025*
