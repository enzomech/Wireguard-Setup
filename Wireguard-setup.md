
# Setup de wireguard


---

## Table des matières

1. [Objectif](#objectif)
2. [Étapes principales du projet](#etapes-principales-du-projet)
   - [Installation de wireguard](#installation-de-wireguard)
   - [Configuration](#configuration)
   - [Configuration persistante](#configuration-persistante)
   - [Test et debuggage](#test-et-debuggage)

---

## Objectif

L'objectif est de réussir à communiquer entre les deux serveurs de notre infrastructure qui sont sur des réseaux internes distincts.

Rappel de l'infrastucture :

![Diagramme infra TP Wireguard](https://github.com/user-attachments/assets/96bda795-1e45-450c-8e89-4fd392b600dd)


## Étapes principales du projet

Les commandes sont à éxécuter sur chacun des deux pairs wiregard ciblés, hors spécification.

### Installation de wireguard

On met à jour le système et les paquets avant toute chose
```
apt update && apt upgrade
```

On installe ensuite wireguard
```
apt install wireguard
```


### Configuration

On va commencer par générer une clé privée puis une clé publique
```
wg genkey | tee privatekey | wg pubkey > publickey
```

On rajoute ensuite une interface réseau dédié à wireguard :

Pour wgsrv1:
```
ip link add 10.0.0.1/24 dev wg0
```

Pour wgsrv2:
```
ip link add 10.0.0.2/24 dev wg0
```

On attribue ensuite notre clé à notre interface wireguard
```
wg set wg0 private-key ./private
```

On active finalement l'interface réseau de wireguard
```
ip link set up dev wg0
```

On utilise finalement cette courte commande correspondant à notre interface wireguard pour avoir les détails :
```
wg0
```

Cela nous donne ce résultat chez wgsrv1
```
wgsrv1

interface: wg0
  public key: tLTyubgaQU3hjNtxqLKUmAwsiwMX64h94y2knWLiLVQ=
  private key: (hidden)
  listening port: 40091

peer: gXNENQKPUwdVpG40oVGSyHqYkyjXpA7YBSMXs/D/VQc=
  endpoint: 192.168.160.10:33915
  allowed ips: 10.0.0.2/32
```

Cela nous donne ce résultat chez wgsrv2
```
wgsrv2

interface: wg0
  public key: gXNENQKPUwdVpG40oVGSyHqYkyjXpA7YBSMXs/D/VQc=
  private key: (hidden)
  listening port: 33915

peer: tLTyubgaQU3hjNtxqLKUmAwsiwMX64h94y2knWLiLVQ=
  endpoint: 192.168.150.10:40091
  allowed ips: 10.0.0.1/32
```

On rajoute finalement la connexion autorisé pour le pair via cette commande qui spécifie la clé publique du pair visé, ainsi que l'adresse utilisé pour joindre notre pair puis finalement l'adresse du pair visé, suivis du port qui est en écoute spécifié dans les détails plus hauts:

wgsrv1 :
```
wg set wg0 peer gXNENQKPUwdVpG40oVGSyHqYkyjXpA7YBSMXs/D/VQc= allowed-ips 10.0.0.2/32 endpoint 192.168.160.10:33915
```

wgsrv2 :
```
wg set wg0 peer tLTyubgaQU3hjNtxqLKUmAwsiwMX64h94y2knWLiLVQ= allowed-ips 10.0.0.1/32 endpoint 192.168.150.10:40091
```

En revanche cette méthode d'ajout de pair n'est pas persistante et cette autorisation sera effacée au redémarage.

### Configuration persistante

Voici un exemple de configuration persistante à renseigner dans '/etc/wireguard/wg0.conf'
```
[Interface]
PrivateKey = clé_privée_du_peer
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = Clé_publique_du_serveur
Endpoint = IP_DU_SERVEUR:PORT_DU_SERVEUR
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

On active WireGuard sur le client :
```
sudo systemctl start wg-quick@wg0
```

Et pour l'activer au démarrage :
```
sudo systemctl enable wg-quick@wg0
```

On peut verifier la configuration avec cette commande :
```
wg show
```

### Test et debuggage

On peut tester la connection en utilisant un ping sur l'adresse autorisé qui traduit l'IP ciblé :

Donc par exemple sur wgsrv1 :
```
ping 10.0.0.2
```

####Problèmes :

On peut rencontrer les problèmes suivant qui sont spécifiés par wireguard :

```
root@wgsrv1:~# ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
ping: sendmsg: Required key not available
```

Cela correspond à un problème de clé publique cible mal spécifiée 


```
root@wgsrv1:~# ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
ping: sendmsg: Destination address required
```

Cela correspond à un problème d'IP cible, celle qui doit être spécifiée en endpoint lors de la configuration du pair, wireguard trouve donc bien l'adresse enregistrée, mais pas l'adresse ip de la cible.
