nft
#!/usr/sbin/nft -f

flush ruleset

define WAN = "enp0s3"
define LAN = "enp0s8"
define DMZ = "enp0s9"



table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                iif "lo" accept
                ct state established,related accept
                ct state invalid drop
                iifname $WAN ip saddr @blocked\_IPs drop
                icmp type echo-request accept
                iifname $LAN ip saddr { 192.168.1.10, 192.168.1.20 } tcp dport 22 \\
                ct state new limit rate 3/minute accept
                iifname $WAN tcp dport 22 ct state new log prefix "SSH-WAN : " drop
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state established,related accept
                ct state invalid drop
                iifname $WAN ip saddr @blocked\_IPs drop
                # Adresse critiques : aucun acces WAN ni DMZ (une seule regle), placee EN PREMIER
                ip saddr {192.168.1.50-192.168.1.60, 192.168.1.100, 192.168.1.200} \\
                        oifname {$WAN, $DMZ} drop

                # DNS autorise uniquement vers OpenDNS
                iifname $LAN oifname $WAN ip daddr 208.67.222.123 \\
                        meta l4proto {tcp, udp} th dport 53 accept

                # HTTPS sortant
                iifname $LAN oifname $WAN tcp dport 443 accept

                # HTTP refuse immediatement .reject. au lieu de laisser ramer
                iifname $LAN oifname $WAN tcp dport 80 reject with tcp reset

                #Ping autorise LAN to DMZ
                iifname $LAN oifname $DMZ icmp type echo-request accept

                #Acces HTTP au serveur web LAN to DMZ
                iifname $LAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 80 accept

                #destination DEJA traduite par le DNAT : on filtre sur l'IP interne et le port 22
                iifname $WAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 80 accept
                iifname $WAN oifname $DMZ ip daddr 172.16.0.10 tcp dport 22 accept
        }
        chain output {
                type filter hook output priority filter; policy drop;
                oif "lo" accept
                ct state established,related accept
        }
        set blocked\_IPs {
                type ipv4\_addr
                flags interval
                elements = {
                        10.0.0.0/8,
                        172.16.0.0/12,
                        127.0.0.0/8,
                        169.254.0.0/16
                }
        }
}

table ip nat {
        chain prerouting {
                type nat hook prerouting priority dstnat;
                iifname $WAN tcp dport 80 dnat to 172.16.0.10:80
                iifname $WAN tcp dport 61337 dnat to 172.16.0.10:22
        }
        chain postrouting {
                type nat hook postrouting priority srcnat;
                oifname $WAN ip saddr 192.168.1.0/24 masquerade
        }
}

