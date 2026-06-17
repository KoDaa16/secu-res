# Sécurité des réseaux - Pfsense GitHub

# Topologie :

```bash
   ----------Tunnel OpenVPN - WAN en DHCP----------
   |                 10.0.8.0/30                  |   
Site 1                                        Site 2
-----------                                   -----------
Pfsense1 :                                    Pfsense 2 :
WAN : DHCP                                    WAN : DHCP
LAN : 172.16.0.1/25                           LAN : 172.18.0.1/25
    |                                             |
    | 172.16.0.0/25                               |
    |                                             | 172.18.0.0/25
Routeur Linux                                     |
Vers PF1 : 172.16.0.126                           |
vers client : 172.16.0.254                    Client Linux 2
    |                                         IP : 172.18.0.10
    |172.16.0.128/25                          GW : 172.18.0.1
    |
Client Linux
IP : 172.16.0.129
GW : 172.16.0.254
```

# 1. Ce qu'on te demande (et le piège à comprendre)

L'examen est noté sur 20, en 2h. La règle d'or : un critère ne compte que si tu peux le **démontrer ET l'expliquer**, et il faut valider les critères de base avant de toucher aux dépassements.

Les deux critères de base (le cœur, 10 points) : d'abord que le tunnel VPN soit monté et que le Routeur Linux arrive à pinguer le Client Linux 2 (172.18.0.10) ; ensuite que les deux clients Linux puissent se pinguer mutuellement.

Le **piège central** de ce sujet, c'est le Routeur Linux. Regarde le schéma : derrière pfSense 1, il y a un routeur qui crée un *deuxième* sous-réseau (172.16.0.128/25) où vit le Client 1. Or pfSense 1 ne connaît directement que son LAN (172.16.0.0/25) — il ignore l'existence du 172.16.0.128/25. Tu devras donc faire deux choses sur pfSense 1 : lui ajouter une **route statique** vers 172.16.0.128/25 via le Routeur Linux (172.16.0.126), et **annoncer tout le 172.16.0.0/24** dans le VPN (le /24 englobe les deux /25 d'un coup). C'est là que se gagnent ou se perdent la majorité des points.

---

# 2. Plan d'adressage (à recopier tel quel)

| Machine | Interface | IP / masque | Passerelle (GW) | Réseau VirtualBox |
| --- | --- | --- | --- | --- |
| pfSense 1 | WAN | DHCP | auto | Réseau NAT (commun) |
| pfSense 1 | LAN | 172.16.0.1/25 | — | Interne « site1-lan » |
| Routeur Linux | eth0 (côté PF1) | 172.16.0.126/25 | 172.16.0.1 | Interne « site1-lan » |
| Routeur Linux | eth1 (côté client) | 172.16.0.254/25 | aucune | Interne « site1-cli » |
| Client Linux 1 | eth0 | 172.16.0.129/25 | 172.16.0.254 | Interne « site1-cli » |
| pfSense 2 | WAN | DHCP | auto | Réseau NAT (commun) |
| pfSense 2 | LAN | 172.18.0.1/25 | — | Interne « site2-lan » |
| Client Linux 2 | eth0 | 172.18.0.10/25 | 172.18.0.1 | Interne « site2-lan » |

Côté VPN (on s'en servira plus tard) : réseau du tunnel `10.0.8.0/30`, réseau local annoncé par PF1 = `172.16.0.0/24`, réseau distant annoncé par PF2 = `172.18.0.0/25`.

---

# 3. Les VMs à créer et leur réseau dans VirtualBox

Cinq VMs au total (pas besoin du Windows Server : il ne sert que pour le VPN *remote access* du cours, pas pour cet examen) :

- pfSense 1 - 2 cartes : carte 1 = WAN (Réseau NAT), carte 2 = LAN (Réseau interne « site1-lan »).
- Routeur Linux (Debian) - 2 cartes : carte 1 sur « site1-lan », carte 2 sur « site1-cli ».
- Client Linux 1 (Debian) - 1 carte : sur « site1-cli ».
- pfSense 2 - 2 cartes : carte 1 = WAN (Réseau NAT), carte 2 = LAN (Réseau interne « site2-lan »).
- Client Linux 2 (Debian) - 1 carte : sur « site2-lan ».

---

# 4. Config Pfsense - Assigner interfaces

## Sur PF1 :

Option 1 :

```bash
n
WAN : em0
LAN : em1
y
```

Option 2 :

```bash
2 --> choisir l'interface LAN
n
172.16.0.1 #PF1   Ou 172.18.0.1 #PF2
25
Enter
n
Enter
n
n
ENTER
```

Options 7 : **Ping host** sert à valider que ton WAN fonctionne.

```bash
ping 8.8.8.8
```

Teste `8.8.8.8` : si le ping répond, ton pfSense 1 sort bien sur Internet via le Réseau NAT. C'est exactement le test demandé dans l'énoncé.

## Desactiver le blocage sur les interface internet de PFsense

Dans Configure WAN → Tout en bas, desactiver les 2 derniers 

→ Block RFC1918 Private Networks   ET  →  Block bogon networks 

> ou dans ***interfaces → WAN (tout en bas)***
> 

**Elles peuvent maintenant s’entre ping (PF1 et PF2)**

---

# 5. Config des Clients et Routeur

**Routeur :**

```bash
# Carte 1 — vers pfSense 1 (réseau 172.16.0.0/25)
auto enp0s3
iface enp0s3 inet static
    address 172.16.0.126/25
    gateway 172.16.0.1

# Carte 2 — vers Client Linux 1 (réseau 172.16.0.128/25)
auto enp0s8
iface enp0s8 inet static
    address 172.16.0.254/25
```

`sudo nano /etc/sysctl.conf` ajoute `net.ipv4.ip_forward=1` Puis applique avec `sudo sysctl -p`.

**Client Linux 1 :**

```bash
auto enp0s3
iface enp0s3 inet static
address 172.16.0.129/25
gateway 172.16.0.254
```

**Client Linux 2 :**

```bash
auto enp0s3
iface enp0s3 inet static
    address 172.18.0.10/25
    gateway 172.18.0.1
```

**TEST 3 pings OK et 1 ping echouer :**

```bash
ping 172.16.0.254 #ping OK -> depuis Client1 (C1 -> R)
ping 172.16.0.1 #ping OK -> depuis Routeur (R -> PF1)
ping 172.18.0.1 #ping OK -> depuis Client2  (C2 -> PF1)
ping 172.16.0.1 #ping NO OK -> depuis Client1 (C1 -> PF1) PF1 ne connait que son reseau

```

---

# 6. Créer la passerelle sur PF1 (RL vu par PF1)

PF1 → System → Routing → Gateway → Add

```bash
Interface → LAN
Address Family → IPv4
Name → ROUTEUR_LINUX
Gateway → 172.16.0.126 (l'adresse du Routeur côté PF1)
Save. Apply Changes
```

---

# 7. Créer la route statique sur PF1

PF1 → System → Routing → Static Routes → Add

```bash
Destination network → 172.16.0.128 masque /25
Gateway → ROUTEUR_LINUX 
Description → libre, ex. « Sous-réseau Client 1 »
Save, puis Apply Changes.
```

---

# 8. Autoriser la ping sur firewall

Firewall → Rules → LAN → Add (haut)

```bash
Action → Pass
Interface → LAN
Address Family → IPv4
Protocol → Any
Source → choisis Network et saisis 172.16.0.128 /25
Destination → Any
Description → Autoriser sous-réseau Client 1
Save, puis Apply Changes.
```

---

# 9. Test ping Client1 → PF1

```bash
ping 172.16.0.1 #ping OK 
```

---

# 10. Créer le tunnel provisoire

**PF1 → VPN → OpenVPN → Servers → Add**

```bash
Server mode → Peer to Peer ( Shared Key )
Device mode → tun
Protocol → UDP on IPv4 only
Interface → WAN
Local port → laisse 1194
Shared Key → coche Automatically generate a shared key (tu copie apres le certif)
Encryption Algorithm → laisse la valeur par défaut
IPv4 Tunnel Network -> 10.0.8.0/30
IPv4 Remote network(s) -> 172.18.0.0/24
Save
```

copier la sharing key -----BEGIN OpenVPN Static key-----

**PF2 → VPN → OpenVPN → Clients → Add**

```
Server mode → Peer to Peer ( Shared Key )
Protocol → UDP on IPv4 only
Device mode → tun
Interface → WAN
Server host or address → l'adresse WAN de PF1 (le 10.1.31.x de PF1)
Server port → 1194
Shared Key → décoche « Automatically generate », et colle la key
Encryption Algorithm → la même que sur PF1 (change rien)
Tunnel Network → 10.0.8.0/30 
Remote Network → 172.16.0.0/24 
Save.
```

**PF1 → FIrewall → Rules → WAN → Add**

```
Action → Pass
Interface → WAN
Address Family → IPv4
Protocol → UDP
Source → Any 
Destination → WAN address
Destination Port Range → From 1194 To 1194 
Description → « OpenVPN entrant »
Save, puis Apply Changes.
```

Verifie le tunnel fonctionne → **Status → OpenVPN → Voir Virtual Adress : 10.0.8.x et Stauts : Connected (Success)**

---

# 11 Ouvrir le firewall sur OpenVPN

**PF1 et PF2 → Firewall → Rules → OpenVPN → Add**

```
Action → Pass
Interface → OpenVPN
Address Family → IPv4
Protocol → AnySource → Any
Destination → Any
Description → ex. « Autoriser trafic VPN »
Save, puis Apply Changes.
Refais exactement la même règle sur PF2 (onglet OpenVPN → Add → Pass / Any / Any).
```

Refais exactement la même règle sur PF2 

**TEST : /6 Sur Routeur ping client 2 (ping 172.18.0.10). 
Sur Client 1 ping Client 2 (ping 172.18.0.10) et depuis client 2 ping client 1 (ping 172.16.0.129)**

# ICI TU AS FINI LA PREMIÈRE PARTIE /10

---

# 12. Bloquer SSH de Client 2 → Client 1 (juste ssh)

**PF1 → Firewall → Rules → OpenVPN → Add (en haut)**

```
Action → Block (ou Reject ; voir ma remarque plus bas)
Interface → OpenVPN
Address Family → IPv4
Protocol → TCP (SSH)
Source → Network 172.18.0.0/25 
Destination → Single host 172.16.0.129
Destination Port Range → From SSH (22) To SSH (22)
Save, puis Apply Changes.
```

sur client 1 sudo apt install openssh-server

sur client 2 : ssh 172.16.0.129 #bloquer         ET ping 172.16.0.129 #passe

---

# 13. Créer la Certificate authorities

**PF1 → Certificates → Authorities → Add**

```
Descriptive Name -> VPN-CA
Method -> Import an existing Certificate Authority
Key type → RSA, longueur 2048
Digest Algorithm → SHA256
Lifetime (days) → laisse la valeur par défaut (souvent 3650)
Save.

```

# 14. Créer le certificat Serveur

**PF1 → Certificates → Certificates → Add**

```
Method → Create an internal Certificate
Descriptive name → serveur-vpn Dans la section Internal Certificate :
Certificate authority → sélectionne VPN-CA 
Key type → RSA, taille 2048
Digest Algorithm → SHA256
Common Name → serveur-vpn
Certificate Type → Server Certificate
Save
```

# 15. Créer le certificat Client

**PF1 → Certificates → Certificates → Add**

```
Method → Create an internal Certificate
Descriptive name → client-vpn
Certificate authority → VPN-CA 
Key type → RSA 2048, Digest → SHA256 (comme avant)
Common Name → client-vpn
Certificate Type → User Certificate 
Save.
```

---

# 16. Créer le Tunnel SSL/TLS sur PF1

**PF1 → VPN → OpenVPN → Serveur → Add**

```
Server mode → Peer to Peer ( SSL/TLS )
Device mode → tun 
Protocol → UDP on IPv4 only
Interface → WAN
Local port → 1195 
TLS Configuration → coche Use a TLS Key, laisse Automatically generate a TLS Key
Peer Certificate Authority → sélectionne VPN-CA
Server certificate → sélectionne serveur-vpn (ton certificat de type serveur)
DH Parameter Length → laisse la valeur par défaut (2048)
Encryption Algorithm et Auth digest → laisse les valeurs par défaut
IPv4 Tunnel Network → 10.0.9.0/30
IPv4 Local network(s) → 172.16.0.0/24
IPv4 Remote network(s) → 172.18.0.0/25
Save
```

---

# 17. Exporter tout (CA, Certif client + key, key TLS)

1. Exporter le CA :

**PF1 → System → Certificates → Authorities → cliquer sur engrenage du CA (VPN-CA)**

1. Exporter le Certif client et sa key :

**PF1 → System → Certificates → Certificates → cliquer sur engrenage et key du Certif client (client vpn)**

1. Exporter la clé TLS :

**PF1 → VPN → OpenVPN → Servers → Ouvrir le Tunel SSL/TLS et copier coller le TLS KEY**

---

# 18. Importer tout sur PF2

1. Importer le CA :

**PF1 → System → Certificates → Authorities → Add**

```
Method → Import an existing Certificate Authority 
Descriptive name → VPN-CA 
Certificate data → copier le contenu du fichier CA .crt 
Certificate Private Key → laisse vide.
Save
```

1. Importer le Certif client et sa key :

**PF1 → System → Certificates → Certificates → Add**

```
Method → Import an existing Certificate
Descriptive name → client-vpn
Certificate data → colle le contenu du fichier client-vpn.crt 
Private key data → colle le contenu du fichier client-vpn.key #prouve l'idtité de PF2
Save
```

---

# 19. Créer le Tunnel SSL/TLS sur PF2

**PF1 → VPN → OpenVPN → Clients → Add**

```
Server mode → Peer to Peer ( SSL/TLS )
Device mode → tun 
Protocol → UDP on IPv4 only
Interface → WAN
Local port → 1195 
TLS Configuration → coche Use a TLS Key
décoche Automatically generate a TLS Key et colle la key TLS 
Peer Certificate Authority → sélectionne VPN-CA
Server certificate → sélectionne client-vpn
Encryption Algorithm et Auth digest → laisse les valeurs par défaut
IPv4 Tunnel Network → 10.0.9.0/30
IPv4 Remote network(s) → 172.16.0.0/24
Save. Apply Changes.
```

---

# Autoriser l’UDP 1195 sur le WAN de PF1 et Test

**PF1 → Firewall → Rules → WAN → Copier celui 1194 et modifier juste le port avec 1195
Save. Apply Changes.**

Pour tester les pings → Disable l’ancien Tunnel pour montrer que le nouveau (ssl/tls) fonctionne 

# Ce que tu dois savoir expliquer à l'oral

Pour ce critère, on te demandera sûrement la différence avec la clé partagée. L'idée à formuler : en clé partagée, les deux pairs partagent un même secret symétrique, sans vérification d'identité. En SSL/TLS, une autorité de certification signe un certificat distinct pour chaque pair ; à la connexion, chacun prouve son identité avec son certificat, l'autre le valide grâce à la CA commune, et une clé de session est négociée dynamiquement. C'est plus sûr (authentification mutuelle, secret renouvelé) et ça passe à l'échelle (révoquer un client = révoquer son certificat, sans toucher aux autres).

# Examen

Manipulation :
Vous disposez de 2h pour réaliser la manipulation.
Un critère n'est validé que si l'étape est entièrement fonctionnelle, que vous êtes capable d'expliquer et de démontrer ce qui a été réalisé.
Les critères de base devront être validés avant de pouvoir passer aux critères de dépassement.

Vous disposez de 2 ordinateurs représentant chacun un des deux sites.
Vous disposez également d'une VM Debian prête à l'emploi sur le NAS pour les différentes machines Linux.
Vous devez connecter les deux sites par un VPN.
Les IP pour le WAN sont à configurer en DHCP.
Dans un premier temps, le tunnel VPN peut être établi sur base d'un secret partagé.

Critère de base :

1. Le VPN est établi entre les deux sites et le routeur Linux peut pinger le client Linux 2  /6
2. Les deux clients Linux peuvent se pinger.  /4

Critères de dépassement

1. Le tunnel VPN est établi en SSL/TLS.  /3
2. Le client Linux 2 peut pinger le client Linux 1 mais ne peut pas se connecter en SSH sur
cette dernière.  /3
3. Le client Linux 1 se connecte à Internet en passant par le site 2.  /4