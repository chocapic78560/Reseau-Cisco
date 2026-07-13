# 🌐 Infrastructure Réseau IPv6 — Cisco Packet Tracer

Projet réseau réalisé en solo sous **Cisco Packet Tracer**, mettant en œuvre une infrastructure d'entreprise multi-sites avec routage dynamique IPv6, segmentation VLAN, commutation redondante et services applicatifs.

## Présentation

Ce projet simule un réseau d'entreprise entièrement opérationnel réunissant **6 routeurs**, **4 switches multicouches**, **6 PCs** et **6 serveurs**, tous communicant exclusivement en **IPv6** (aucun routage IPv4). Le réseau est segmenté en plusieurs VLANs, route dynamiquement via **OSPFv3**, et assure la redondance de commutation via le protocole **PVST+ Spanning Tree** avec élection contrôlée du Root Bridge et blocage de ports maîtrisé.

![Topologie réseau](topologie.svg)

---

## Topologie

La topologie physique repose sur un **anneau carré** : 4 switches multicouches interconnectés par des doubles liens redondants, formant le cœur du réseau de commutation. Chaque routeur se connecte à ce cœur via des sous-interfaces trunk (Router-on-a-Stick), et chaque switch dessert un groupe d'équipements terminaux.

```
        R1      R2      R3
         \      |      /
          \     |     /
        [Switch1]---[Switch2]
              |   X   |
        [Switch3]---[Switch4]
          /     |     \
         /      |      \
        R4      R5      R6
```

- **6 Routeurs** (Cisco 2811) : R1 à R6, chacun avec plusieurs sous-interfaces trunk dot1Q
- **4 Switches multicouches** (Cisco 3560-24PS) : Switch1 à Switch4
- **6 PCs** : PC1 à PC6, un par segment routeur
- **6 Serveurs** : S1 à S6, un par segment routeur

> 📸 *[Insérer ici le schéma détaillé de la topologie Packet Tracer]*

---

## Technologies et protocoles

| Couche | Technologie |
|---|---|
| Couche 3 — Routage | OSPFv3 (IPv6), Area 0 |
| Couche 3 — Adressage | IPv6 statique (PCs & Serveurs), sous-interfaces IPv6 (Routeurs) |
| Couche 2 — Commutation | Trunk VLAN 802.1Q |
| Couche 2 — Redondance | PVST+ Spanning Tree Protocol |
| Couche 2 — Encapsulation | dot1Q sur sous-interfaces (Router-on-a-Stick) |
| Couche Application | SMTP/POP3, FTP, HTTP/HTTPS, DNS |

---

## Plan d'adressage VLAN

Le réseau utilise **11 VLANs** au total :

| ID VLAN | Nom | Rôle |
|---|---|---|
| VLAN 1 | LAN_VLAN1 | Segment local R1 (PC1, S1) |
| VLAN 2 | LAN_VLAN2 | Segment local R2 (PC2, S2) |
| VLAN 3 | LAN_VLAN3 | Segment local R3 (PC3, S3) |
| VLAN 4 | LAN_VLAN4 | Segment local R4 (PC4, S4) |
| VLAN 5 | LAN_VLAN5 | Segment local R5 (PC5, S5) |
| VLAN 6 | LAN_VLAN6 | Segment local R6 (PC6, S6) |
| VLAN 12 | LIEN_R1_R2 | Lien inter-routeurs R1 ↔ R2 |
| VLAN 23 | LIEN_R2_R3 | Lien inter-routeurs R2 ↔ R3 |
| VLAN 34 | LIEN_R3_R4 | Lien inter-routeurs R3 ↔ R4 |
| VLAN 45 | LIEN_R4_R5 | Lien inter-routeurs R4 ↔ R5 |
| VLAN 56 | LIEN_R5_R6 | Lien inter-routeurs R5 ↔ R6 |

Chaque routeur se connecte au réseau de commutation via **un seul port trunk (FastEthernet0/0)** avec plusieurs sous-interfaces dot1Q — une pour son VLAN LAN local, une ou deux pour les VLANs de liens inter-routeurs.

> 📸 *[Insérer ici une capture de `show vlan brief` sur l'un des switches]*

---

## Routage — OSPFv3

Les 6 routeurs exécutent **OSPFv3 en Area 0**, annonçant leurs préfixes LAN locaux et leurs préfixes de liens inter-routeurs. Chaque routeur est configuré avec :

- Un **Router ID** unique au format IPv4 (ex : `1.1.1.1` pour R1)
- **OSPFv3 activé par sous-interface** (`ipv6 ospf 1 area 0`)
- **Aucune adresse IPv4** — environnement 100% IPv6

Exemple de convention d'adressage sur les sous-interfaces (R2) :
```
interface FastEthernet0/0.2   → VLAN 2  → 2000:2001:112::2/64  (LAN local)
interface FastEthernet0/0.12  → VLAN 12 → 2000:12XX:112::2/64  (lien R1-R2)
interface FastEthernet0/0.23  → VLAN 23 → 2000:23XX:112::2/64  (lien R2-R3)
```

> 📸 *[Insérer ici une capture de `show ipv6 ospf neighbor` sur un routeur, montrant toutes les adjacences]*

> 📸 *[Insérer ici une capture d'un ping réussi entre PC1 et PC6 (hôtes les plus éloignés)]*

> 📸 *[Insérer ici une capture de `tracert` de PC1 vers PC6 montrant le chemin complet hop par hop]*

---

## Commutation — Spanning Tree

Les 4 switches sont interconnectés en **anneau carré** avec **double lien par segment** (8 câbles inter-switch au total), créant plusieurs boucles redondantes. PVST+ gère la prévention des boucles par VLAN.

### Configuration STP

| Switch | Priorité STP | Rôle |
|---|---|---|
| Switch4 | 4096 | ✅ Root Bridge (tous VLANs) |
| Switch1 | 28672 | Non-root |
| Switch2 | 61440 | Non-root (3 ports bloqués) |
| Switch3 | 32768 (défaut) | Non-root |

- **Root Bridge** : Switch4 (`spanning-tree vlan 1-4094 priority 4096`)
- **Switch avec 3 ports bloqués** : Switch2 — ports en état `Altn BLK`
- Tous les ports bloqués sont des ports **GigabitEthernet**
- Configuration réalisée **sans utiliser la commande `spanning-tree cost`**

> 📸 *[Insérer ici une capture de `show spanning-tree` sur Switch4 montrant "This bridge is the root"]*

> 📸 *[Insérer ici une capture de `show spanning-tree vlan 1` sur Switch2 montrant 3 ports en `Altn BLK`]*

---

## Services applicatifs

| Serveur | Service | Détails |
|---|---|---|
| S1 | DNS | Résolution de noms pour le réseau |
| S4 | SMTP / POP3 | Serveur mail — domaine `[tonNom].com` |
| S5 | FTP | Serveur de transfert de fichiers |
| S6 | HTTP / HTTPS | Serveur web |

### Test messagerie
Deux comptes utilisateurs sont configurés sur le serveur SMTP (S4) :
- **User1** / `pass1` → configuré sur le client mail d'un PC (Desktop → E Mail)
- **User2** / `pass2` → configuré sur un second PC client

> 📸 *[Insérer ici une capture d'un envoi/réception de mail réussi entre User1 et User2]*

> 📸 *[Insérer ici une capture d'un accès HTTP réussi depuis un PC vers le serveur web S6]*

---

*Réalisé avec Cisco Packet Tracer · IPv6 · OSPFv3 · PVST+ · 802.1Q*
