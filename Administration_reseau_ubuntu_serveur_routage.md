# Administration réseau Ubuntu serveur 

Pour cet exercice, on va simuler un réseau avec un serveur Ubuntu qui fera office de serveur DHCP. On aura aussi un réseau interne avec des machines clientes qui se connecteront au serveur DHCP pour obtenir une adresse IP et qui utiliseront l'adresse IP DNS du réseau externe pour résoudre les noms de domaine.

PS: Les machines clientes doivent avoir une carte réseau "interne" et le nom du réseau doit être le même que celui du serveur. (Exemple: "lab_diaz" pour notre cas)
Les machines clientes doivent avoir un nom de réseau pour être sur un même switch

## 1. Paramétrage des cartes réseaux

- Deux cartes réseaux :
    -Une en bridge (connecté au réseau local donnant accès à Internet)
    -Une en interne (connecté au réseau interne, c'est celle-ci qui servira de passerelle pour le réseau interne)

## 2. Configuration des interfaces réseau

L'interface en bridge doit avoir une adresse IP réservée ou fixe (pour ne pas à avoir à changer de configuration à chaque redémarrage du serveur) et configurer l'interface en interne avec une adresse IP fixe

## 3. Activation du routage

- Activer le routage(On va proposer deux méthodes pour le faire)
    - sysctl -w net.ipv4.ip_forward=1

    - sudo nano /proc/sys/net/ipv4/ip_forward :
    Ce fichier a de base la valeur 0 et on va devoir l'attribuer la valeur 1 pour activer le routage.

- Vérification du routage

    - sysctl net.ipv4.ip_forward

    - cat /proc/sys/net/ipv4/ip_forward


## 4. Configuration du serveur DHCP

- Installation du serveur DHCP

    - sudo apt update
    - sudo apt install isc-dhcp-server

- Configuration du serveur DHCP

    - sudo nano /etc/default/isc-dhcp-server
        INTERFACESv4="enp0s8" (interface du réseau interne, celle qui recevra les requêtes DHCP)
        INTERFACESv6=""

    - sudo nano /etc/dhcp/dhcpd.conf
        subnet (adresse du réseau) netmask (masque du réseau) {
            range (adresse IP de début) (adresse IP de fin);     # Plage d'IP distribuées
            option routers (adresse IP de la passerelle);           # Passerelle par défaut (IP de l'interface en interne)
            option domain-name-servers (adresse IP du serveur DNS);   # DNS
            default-lease-time 600;  # Durée de vie du bail en secondes
            max-lease-time 7200;     # Durée de vie maximale du bail en secondes
            }
        ![alt text](image-7.png)

     Netplan est un outil de configuration réseau pour Ubuntu
    - Il est situé dans le dossier /etc/netplan/
    - Il est écrit en YAML
    - Il est composé de deux parties :
        - Une partie pour la carte réseau en bridge
        - Une partie pour la carte réseau en interne

    Créer un nouveau fichier yaml avec un nombre élevé pour qu'il soit le dernier à être lu par le serveur et qu'il remplace les configurations précedentes:

    - sudo nano /etc/netplan/99-config-statique.yaml

          network:
            version: 2
            renderer: networkd
            ethernets:
                enp0s8:
                dhcp4: no
                addresses:
                    - x.x.x.x/az    # L'IP fixe de ton serveur
                nameservers:
                    addresses: [x.x.x.x, y.y.y.y] # serveurs DNS
                routes:
                    - to: default
                    via: x.x.x.x  # L'IP de ta box ou de ton routeur
        ![alt text](image-4.png)
    - sudo netplan try (Pour tester les configurations)
    - sudo netplan apply (Pour appliquer définitivement les configurations)

 


- Vérification du serveur DHCP
    - sudo systemctl (re)start isc-dhcp-server
    - sudo systemctl status isc-dhcp-server
    ![alt text](image-3.png)

    En cas d'erreur, taper la commande "sudo journalctl -xeu isc-dhcp-server" pour voir précisément quelle ligne du fichier de config pose problème.
    Pour consulter les logs, on peut taper la commande "sudo tail -f /var/log/syslog | grep dhcpd"
    
    VM Ubuntu (Client):
    - Libérer l'ancienne IP
    sudo dhclient -r 
    - Demander une nouvelle IP (mode verbeux pour voir les échanges)
    sudo dhclient -v

    VM Windows (Client):
    - ipconfig /release
    - ipconfig /renew

    ![alt text](image-6.png) 
    On voit là que les machines Ubuntu et Windows se voient attribuer des paramètres IP par le serveur DHCP.
    

  
## 5. Configuration du NAT (Masquerading)

Pour que les clients du réseau interne puissent accéder à Internet via l'interface bridge du serveur, il faut configurer le NAT avec `iptables`.

- Configuration de la règle NAT :
    - `sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE` (Remplacez `enp0s3` par le nom de votre interface bridge)

- Rendre la règle persistante :
    - `sudo apt install iptables-persistent`
    - `sudo netfilter-persistent save`
