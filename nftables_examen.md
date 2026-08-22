## Avant tout : vérifier que les règles sont chargées

Sur le **firewall** :

```
sudo nft -f /etc/nftables.conf     # si aucune erreur, rien ne s'affiche
sudo nft list ruleset               # visualiser toutes les règles actives
```

Si `nft -f` renvoie un message, corrige la syntaxe avant d'aller plus loin.

---

## Critère 1 - Politique par défaut : tout bloquer (/2)

```
sudo nft list ruleset | grep policy
```

Tu dois voir `policy drop` sur les trois chaînes de filtrage (input, forward, output). C'est la démonstration directe. Explique au prof : « tout est jeté par défaut, j'ouvre ensuite au cas par cas ».

## Critère 2 — LAN surfe sur Internet, chiffré uniquement (/4)

Depuis le **client LAN** :

```
wget -qO- https://www.google.com | head    # HTTPS -> DOIT marcher
wget http://neverssl.com                    # HTTP -> DOIT échouer
```

Le premier renvoie du HTML (accès chiffré OK), le second échoue (non chiffré bloqué). Pour montrer que le DNS marche aussi :

```
nslookup google.com
```

## Critère 3 — LAN accède au site web DMZ en HTTP, **et seulement** au site (/2)

Depuis le **client LAN** :

```
wget -qO- http://172.16.1.10 | head    # DOIT marcher (page Apache)
ssh 172.16.1.10                          # DOIT échouer (seulement le web !)
ping 172.16.1.10                         # DOIT échouer (seulement le web !)
```

Les deux échecs sont **la preuve** que tu n'es pas trop permissif — souligne-le au prof, c'est ce qu'il vérifie.

## Critère 4 — Sessions déjà établies autorisées (/1)

C'est démontré implicitement par tout ce qui précède : ton `wget https` fonctionne **parce que** la réponse revient via `ct state established,related`. Montre la règle :

```
sudo nft list ruleset | grep "ct state"
```

Explique : « sans cette règle, j'aurais dû écrire une règle de retour pour chaque flux ».

## Critère 5 — Loopback (/1)

Sur le **firewall** :

```
ping -c2 127.0.0.1        # doit répondre
```

Et montre la règle `iif "lo" accept` dans le ruleset.

## Critère 6 — Client WAN → SSH serveur Web via 61337 (/4)

Depuis ton **PC hôte** (le client externe) :

```
ssh -p 61337 <user_du_serveur_DMZ>@<IP_WAN_du_firewall>
```

Une fois connecté, **prouve que tu es sur le serveur DMZ** :

```
hostname
ip -br a          # doit montrer 172.16.1.10
```

C'est le point clé : tu n'es pas sur le firewall, tu as bien traversé via le DNAT.

## Critère 7 — Seul le serveur Web-SSH peut pinger le pare-feu (/2)

Il faut montrer **les deux sens** :

Depuis le **serveur DMZ** (autorisé) :

```
ping -c3 172.16.1.254        # DOIT marcher
```

Depuis le **client LAN** (interdit) :

```
ping -c3 <IP_LAN_du_firewall>    # DOIT échouer
```

Depuis ton **hôte** (interdit) :

```
ping <IP_WAN_du_firewall>        # DOIT échouer
```

Les deux échecs prouvent le « **seul** » du critère. Précise que la règle n'a pas de `iifname`, donc le serveur DMZ peut pinger n'importe quelle interface du firewall — teste-le d'ailleurs :

```
# depuis le serveur DMZ
ping -c2 <IP_WAN_du_firewall>    # doit marcher aussi (quelle que soit l'interface)
```

## Critère 8 — Set `blocked_hosts` (/2)

Montre d'abord le set :

```
sudo nft list set inet filter blocked_hosts
```

Puis démontre-le. Sur le **client LAN**, ajoute temporairement une IP bloquée :

```
sudo ip addr add 192.168.1.50/24 dev enp0s3
wget -qO- --bind-address=192.168.1.50 https://www.google.com    # DOIT échouer
wget -qO- --bind-address=192.168.1.10 https://www.google.com    # DOIT marcher
sudo ip addr del 192.168.1.50/24 dev enp0s3
```

Le `--bind-address` force wget à utiliser l'IP source voulue. Le contraste entre les deux prouve que .50 est bloqué alors que le reste du LAN garde l'accès.

## Critère 9 — Logging des connexions SSH WAN → DMZ (/2)

Depuis ton **hôte**, lance une tentative SSH :

```
ssh -p 61337 <user>@<IP_WAN>
```

Puis sur le **firewall**, montre les logs :

```
sudo journalctl -k --grep "SSH-DMZ"
```

Pour une démo en direct plus impressionnante, lance d'abord sur le firewall :

```
sudo journalctl -kf | grep "SSH-DMZ"
```

puis fais la tentative SSH depuis l'hôte — le prof voit la ligne apparaître en temps réel.
