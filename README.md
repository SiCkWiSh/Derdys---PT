# Derdys---PT
# PROJET DERDYS - Architecture Réseau d'Entreprise Redondante et Sécurisée

![Topologie du Projet Derdys](image_0.png)

## 1. Introduction

Le **Projet Derdys** est une simulation d'architecture réseau d'entreprise conçue dans Cisco Packet Tracer. L'objectif est de mettre en place un réseau local (LAN) d'entreprise moderne, performant et hautement disponible, capable de se connecter de manière sécurisée à une ressource distante (Serveur) via un réseau de bordure (WAN).

Cette architecture intègre des concepts clés du réseautage informatique tels que la segmentation VLAN, la redondance de passerelle (HSRP), le routage inter-VLAN, la sécurité des ports (Port-Security) et la translation d'adresses (NAT).

## 2. Architecture et Design

La topologie suit une structure hiérarchique :

### A. Zone Accès (Layer 2)
Cette couche connecte les hôtes finaux (PC Victor, PC Emile).
* **ETG1 (RH - VLAN 10) :** Commutateur d'accès gérant les connexions du service RH.
* **ETG2 (IT - VLAN 20) :** Commutateur d'accès gérant les connexions du service IT.
* **Sécurité :** Le Port-Security est activé sur les ports Fa0/1 de ces switches (mode Sticky) pour limiter l'accès réseau aux adresses MAC connues.
* **Trunking :** Les liens vers la couche distribution (MLS/MLSJean) sont en mode `trunk` pour transporter les tags VLAN.

### B. Zone Distribution/Cœur (Layer 3)
Cette couche assure le routage inter-VLAN et la haute disponibilité.
* **MLS & MLSJean (Multilayer Switches) :** Ces deux commutateurs de niveau 3 gèrent le routage des paquets entre les VLANs (SVI interfaces).
* **Haute Disponibilité (HSRP) :** Le protocole HSRP (Hot Standby Router Protocol) est configuré entre MLS et MLSJean pour fournir une passerelle virtuelle (`.254`) hautement disponible aux utilisateurs. Si un switch tombe en panne, l'autre prend le relais en quelques secondes sans interruption pour l'utilisateur.
* **VLAN de Transit 200 :** Un VLAN spécifique est dédié au routage entre les commutateurs de cœur (MLSJean) et le routeur de bordure (PE1).

### C. Zone Bordure et WAN (Routing & NAT)
* **PE1 (Routeur de Bordure) :** Ce routeur assure la sortie vers le monde extérieur (le serveur).
* **NAT/PAT (Network Address Translation) :** Pour des raisons de sécurité et d'économie d'adresses IPv4, PE1 effectue une translation d'adresses (PAT - Overload) sur l'interface Gig0/3/0. Toutes les adresses privées (192.168.x.x) sont masquées derrière l'adresse publique du routeur vers le WAN.
* **Routage Statique de Retour :** PE1 est configuré avec des routes statiques pour renvoyer le trafic de retour vers les sous-réseaux internes (VLAN 10, 20) via l'IP de MLSJean sur le VLAN de Transit 200.

### D. Zone Distante (Serveur)
* **RE1 (Routeur Distant) :** Routeur simple gérant la connectivité directe du serveur.
* **Aucune Route de Retour :** En accord avec le design PAT sur PE1, RE1 ne possède *aucune* route statique vers les réseaux privés (192.168.x.x). Il sait seulement répondre à PE1 (adresse publique WAN) et au Serveur (réseau local).
* **Serveur :** Configuré avec une IP fixe et une passerelle par défaut vers RE1.

## 3. Plan d'Adressage IP

| Zone / VLAN | Sous-réseau | Passerelle (Virtual IP/Gateway) | IP Physique (HSRP Node 1) | IP Physique (HSRP Node 2) | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | `192.168.10.0/24` | `192.168.10.254` (HSRP) | `192.168.10.253` (MLS) | `192.168.10.252` (MLSJean) | Service RH (Victor) |
| **VLAN 20** | `192.168.20.0/24` | `192.168.20.254` (HSRP) | `192.168.20.253` (MLS) | `192.168.20.252` (MLSJean) | Service IT (Emile) |
| **Transit VLAN 200** | `192.168.200.0/24` | N/A | `192.168.200.253` (MLS) | `192.168.200.252` (MLSJean) | Inter-MLS |
| **Lien Transit vers PE1** | `192.168.200.0/24` | N/A | `192.168.200.1` (PE1-Gig0/0) | `192.168.200.252` (MLSJean-Gig1/0/4) | Routage vers WAN |
| **WAN (PE1-RE1)** | `203.0.113.0/30` | N/A | `203.0.113.1` (PE1-Gig0/3/0) | `203.0.113.2` (RE1-Gig0/3/0) | Lien Inter-Routeur |
| **Serveur Zone** | `172.16.0.0/24` | `172.16.0.1` (RE1-Gig0/0) | N/A | N/A | Zone du Serveur (Server0 @ .200) |

## 4. Fonctionnalités et Technologies Implémentées

| Technologie | Implémentation | Bénéfice |
| :--- | :--- | :--- |
| **VLAN Segmentation** | VLANs 10 (RH) et 20 (IT) | Sécurité par isolation du trafic et meilleure gestion des domaines de diffusion. |
| **Port-Security** | Mode Sticky sur ports Fa0/1 (ETG1/2) | Limitation de l'accès physique à des adresses MAC connues uniquement. |
| **HSRP Redundancy** | Priorité HSRP (110) sur MLS (VLAN 10) et MLSJean (VLAN 20). Préemption activée. | Haute disponibilité de la passerelle. Pas de coupure si un switch de distribution tombe. |
| **Inter-VLAN Routing** | SVIs configurées sur les commutateurs MLS/MLSJean. `ip routing` activé. | Communication fluide entre les départements RH et IT. |
| **NAT Overload / PAT** | ACL sur PE1 permit 192.168.x.x. `ip nat inside source list 1 interface Gig0/3/0 overload` | Connexion WAN/Serveur sécurisée. Masquage des adresses IP privées internes. Simplification du routage sur RE1. |
| **Static Routing** | Routes par défaut (0.0.0.0) sur MLS -> PE1. Routes de retour statiques sur PE1 -> MLSJean. | Contrôle total du flux de trafic entrant et sortant. |

## 5. Guide de Configuration (Synthèse)

Ce dépôt contient le fichier `.pkt` complet. Voici un résumé des configurations critiques appliquées par équipement.

### Commutateurs d'Accès (ETG1/2)
```bash
# Activation Port-Security
interface FastEthernet0/1
 switchport port-security sticky

# Configuration Trunk vers Cœur
interface GigabitEthernet0/1
 switchport mode trunk
```

### Commutateurs Multilayer (MLSJean - Exemple de Gateway VLAN 10 & 20)
```bash
# Activation Routage L3
ip routing

# Configuration Gateway HSRP VLAN 10 (IP virtuelle .254, node .252)
interface Vlan10
 ip address 192.168.10.252 255.255.255.0
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt

# Configuration Port vers Transit VLAN 200 (PE1)
interface GigabitEthernet1/0/4
 no switchport
 ip address 192.168.200.252 255.255.255.0
```

### Routeur de Bordure (PE1 - Configuration NAT)
```bash
# Définition des interfaces
interface GigabitEthernet0/0 (Interne/Cœur)
 ip nat inside
interface GigabitEthernet0/3/0 (Externe/WAN)
 ip nat outside

# Définition de l'ACL pour le NAT (masquage de tout le 192.168.0.0/16)
access-list 1 permit 192.168.0.0 0.0.255.255

# Activation NAT Overload
ip nat inside source list 1 interface Gig0/3/0 overload

# Route de Retour vers les PCs (via MLSJean)
ip route 192.168.0.0 255.255.0.0 192.168.200.252
```

## 6. Vérification et Tests

Les étapes suivantes confirment le bon fonctionnement de l'architecture :

1.  **Vérification Port-Security :** Brancher un nouvel appareil sur ETG1 port Fa0/1. Le port doit passer en "shut" (triangle rouge) automatiquement.
2.  **Routage Inter-VLAN :** Depuis le PC Victor (192.168.10.1), effectuer un `ping` vers Emile (192.168.20.1). Le ping doit réussir.
3.  **HSRP Redundancy :** Depuis Emile, lancer un `ping -t 172.16.0.200` vers le serveur. Éteindre le commutateur MLSJean dans Packet Tracer. Le ping doit reprendre après une courte interruption de quelques secondes via MLS.
4.  **Connectivité Serveur & NAT :** Depuis Emile, `ping 172.16.0.200`. Le ping réussit. Sur PE1, la commande `show ip nat translations` montre la traduction active :
    `tcp 203.0.113.1:1024 192.168.20.1:1024 172.16.0.200:80 172.16.0.200:80` (Ceci prouve que le serveur voit PE1 et non Emile directement).

## 7. Prérequis et Comment l'Utiliser

1.  Avoir Cisco Packet Tracer installé (Version récente recommandée).
2.  Télécharger le fichier `derdys.pkt` depuis ce dépôt.
3.  Ouvrir le fichier dans Packet Tracer.
4.  Les configurations sont pré-chargées. Vous pouvez immédiatement effectuer les tests de vérification.

## 8. Auteur

I  D  R  I  S  S     Q .
* Projet conçu et réalisé dans le cadre d'un laboratoire pratique de réseaux.
* Retrouvez la discussion LinkedIn de ce projet ici .
