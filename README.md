# Routage statique avec Cisco Packet Tracer

Exercices d'entraînement sur le **routage statique** : deux routeurs Cisco 1941 reliés entre eux, chacun connecté à un PC. Le même scénario est réalisé deux fois, avec deux types de liaison entre les routeurs :

1. **Maquette 1** : liaison Ethernet (GigabitEthernet)
2. **Maquette 2** : liaison Serial (DCE / DTE)

## Contenu du dépôt

```
MaquetteRoutageStatique/
├── static1.pkt          # Maquette 1 : liaison GigabitEthernet
├── static2serial.pkt    # Maquette 2 : liaison Serial
└── README.md
```

Les fichiers `.pkt` s'ouvrent avec **Cisco Packet Tracer**.

## Objectifs

- Configurer les interfaces d'un routeur (adresse IP, `no shutdown`)
- Créer des routes statiques avec `ip route`
- Comprendre le rôle du **DCE** (`clock rate`) et du **DTE** sur une liaison Serial
- Vérifier la connectivité avec `show ip route`, `show ip interface brief` et `ping`

## Plan d'adressage

| Équipement | Interface | Adresse IP | Masque | Passerelle |
|---|---|---|---|---|
| PC1 | FastEthernet0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.1 |
| PC2 | FastEthernet0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.1 |
| R1 | Gi0/0 | 192.168.1.1 | 255.255.255.0 | - |
| R2 | Gi0/0 | 192.168.2.1 | 255.255.255.0 | - |
| R1 ↔ R2 (maquette 1) | R1 Gi0/1 / R2 Gi0/1 | 10.0.0.1 / 10.0.0.2 | 255.255.255.0 | - |
| R1 ↔ R2 (maquette 2) | R1 Se0/1/0 / R2 Se0/1/0 | 10.0.0.1 / 10.0.0.2 | 255.255.255.0 | - |

Réseaux : `192.168.1.0/24` (côté PC1), `192.168.2.0/24` (côté PC2) et `10.0.0.0/24` (lien entre les routeurs).

---

# Maquette 1 : liaison Ethernet (`static1.pkt`)

## Topologie

```
PC1 ---- Gi0/0 [R1] Gi0/1 ---------- Gi0/1 [R2] Gi0/0 ---- PC2
192.168.1.2   192.168.1.1  10.0.0.1  10.0.0.2  192.168.2.1   192.168.2.2
```

Câblage : PC1 ↔ R1 `Gi0/0`, PC2 ↔ R2 `Gi0/0`, R1 `Gi0/1` ↔ R2 `Gi0/1` (câbles croisés).

## Configuration des PC

| | PC1 | PC2 |
|---|---|---|
| Adresse IPv4 | 192.168.1.2 | 192.168.2.2 |
| Masque | 255.255.255.0 | 255.255.255.0 |
| Passerelle | 192.168.1.1 | 192.168.2.1 |

## Configuration des routeurs

**R1**

```
enable
configure terminal
hostname R1
interface gigabitEthernet 0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
interface gigabitEthernet 0/1
ip address 10.0.0.1 255.255.255.0
no shutdown
exit
ip route 192.168.2.0 255.255.255.0 10.0.0.2
end
```

**R2**

```
enable
configure terminal
hostname R2
interface gigabitEthernet 0/0
ip address 192.168.2.1 255.255.255.0
no shutdown
exit
interface gigabitEthernet 0/1
ip address 10.0.0.2 255.255.255.0
no shutdown
exit
ip route 192.168.1.0 255.255.255.0 10.0.0.1
end
```

## Tests et résultats

### 1. Table de routage de R1 (`show ip route`)

```
C    10.0.0.0/24 is directly connected, GigabitEthernet0/1
L    10.0.0.1/32 is directly connected, GigabitEthernet0/1
C    192.168.1.0/24 is directly connected, GigabitEthernet0/0
L    192.168.1.1/32 is directly connected, GigabitEthernet0/0
S    192.168.2.0/24 [1/0] via 10.0.0.2
```

Résultat : la route statique `S` vers `192.168.2.0/24` via `10.0.0.2` est présente.

### 2. Table de routage de R2 (`show ip route`)

```
C    10.0.0.0/24 is directly connected, GigabitEthernet0/1
L    10.0.0.2/32 is directly connected, GigabitEthernet0/1
S    192.168.1.0/24 [1/0] via 10.0.0.1
C    192.168.2.0/24 is directly connected, GigabitEthernet0/0
L    192.168.2.1/32 is directly connected, GigabitEthernet0/0
```

Résultat : la route statique `S` vers `192.168.1.0/24` via `10.0.0.1` est présente.

### 3. Ping de PC1 vers PC2

```
C:\> ping 192.168.2.2

Pinging 192.168.2.2 with 32 bytes of data:

Request timed out.
Request timed out.
Reply from 192.168.2.2: bytes=32 time<1ms TTL=126
Reply from 192.168.2.2: bytes=32 time=3ms TTL=126

Ping statistics for 192.168.2.2:
    Packets: Sent = 4, Received = 2, Lost = 2 (50% loss)
```

Résultat : **réussi**. Les premiers paquets sont perdus à cause de la résolution **ARP** (découverte des adresses MAC), ce qui est normal. Un second ping ne perd plus de paquets. `TTL=126` (128 - 2) confirme que les paquets traversent bien **2 routeurs**.

---

# Maquette 2 : liaison Serial (`static2serial.pkt`)

## Topologie

```
PC1 ---- Gi0/0 [R1] Se0/1/0 ====(Serial DCE)==== Se0/1/0 [R2] Gi0/0 ---- PC2
192.168.1.2   192.168.1.1  10.0.0.1       10.0.0.2  192.168.2.1   192.168.2.2
                   DCE                       DTE
```

## Préparation dans Packet Tracer

1. Éteindre chaque routeur, ajouter un module **HWIC-2T** dans le slot de gauche (eHWIC 1), puis rallumer.
2. Relier R1 et R2 avec un **câble Serial DCE** en cliquant **d'abord sur R1** : R1 est le côté **DCE**, R2 le côté **DTE**.
3. Les interfaces apparaissent sous le nom `Serial0/1/0` (à vérifier avec `show ip interface brief`).

Le DCE (R1) fournit l'horloge à la liaison : la commande `clock rate` est donc nécessaire **uniquement sur R1**.

## Configuration des PC

Identique à la maquette 1 (PC1 : `192.168.1.2`, PC2 : `192.168.2.2`).

## Configuration des routeurs

**R1** (DCE)

```
enable
configure terminal
hostname R1
interface gigabitEthernet 0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
interface gigabitEthernet 0/1
no ip address
shutdown
exit
interface serial 0/1/0
ip address 10.0.0.1 255.255.255.0
clock rate 64000
no shutdown
exit
ip route 192.168.2.0 255.255.255.0 10.0.0.2
end
```

**R2** (DTE, sans `clock rate`)

```
enable
configure terminal
hostname R2
interface gigabitEthernet 0/0
ip address 192.168.2.1 255.255.255.0
no shutdown
exit
interface gigabitEthernet 0/1
no ip address
shutdown
exit
interface serial 0/1/0
ip address 10.0.0.2 255.255.255.0
no shutdown
exit
ip route 192.168.1.0 255.255.255.0 10.0.0.1
end
```

Remarque : `Gi0/1` est désactivée (sans adresse IP) pour éviter un conflit avec l'adresse `10.0.0.x` utilisée par l'interface Serial.

## Tests et résultats

### 1. Interfaces de R1 (`show ip interface brief`)

```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.1.1     YES manual up                    up
GigabitEthernet0/1     unassigned      YES manual administratively down down
Serial0/1/0            10.0.0.1        YES manual up                    up
Serial0/1/1            unassigned      YES unset  administratively down down
Vlan1                  unassigned      YES unset  administratively down down
```

### 2. Interfaces de R2 (`show ip interface brief`)

```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.2.1     YES manual up                    up
GigabitEthernet0/1     unassigned      YES manual administratively down down
Serial0/1/0            10.0.0.2        YES manual up                    up
Serial0/1/1            unassigned      YES unset  administratively down down
Vlan1                  unassigned      YES unset  administratively down down
```

Résultat : `Serial0/1/0` est **up/up** sur les deux routeurs, le `clock rate` de R1 fonctionne.

### 3. Ping de PC1 vers PC2

```
C:\> ping 192.168.2.2

Pinging 192.168.2.2 with 32 bytes of data:

Request timed out.
Reply from 192.168.2.2: bytes=32 time=1ms TTL=126
Reply from 192.168.2.2: bytes=32 time=11ms TTL=126
Reply from 192.168.2.2: bytes=32 time=1ms TTL=126

Ping statistics for 192.168.2.2:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss)
```

Résultat : **réussi**, les paquets passent par le lien Serial (`TTL=126`). Le premier paquet perdu correspond à la résolution ARP.

---

## Points à retenir

- Une **route statique** se déclare avec `ip route <réseau> <masque> <adresse du saut suivant>`.
- Les routes statiques doivent être créées **dans les deux sens** : sans route retour sur R2, le ping de PC1 échouerait.
- Sur une liaison Serial, seul le côté **DCE** configure `clock rate`.
- Dans Packet Tracer, un câble Serial DCE est dessiné en **rouge avec un zigzag** : c'est normal. L'état du lien se lit sur les **triangles verts** aux extrémités.
- Le premier ping échoue souvent à cause de l'**ARP**, ce qui est sans gravité.

## Commandes utiles

```
show ip route
show ip interface brief
show running-config
ping <adresse>
copy running-config startup-config
```
