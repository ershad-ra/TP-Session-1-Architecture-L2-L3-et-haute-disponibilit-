> **Format :** TP guidé sur 2h. Vous lisez, vous configurez, vous testez, vous prenez les captures demandées. Les questions intercalées ne sont pas des questions piège : elles sont là pour vous pousser à réfléchir et chercher si besoin. Pas de copier-coller aveugle.

> **Livrable :** ce document complété (vos réponses aux questions + vos captures d'écran), déposé sur GitHub Classroom à la fin de la séance.

---

## Le scénario

L'école Ynov Toulouse renouvelle complètement son infrastructure réseau. Le campus est composé de deux bâtiments, avec une salle serveur (le **datacenter**) au premier étage du bâtiment A. Le câblage fibre (mono et multi 10G en paire) entre bâtiments et entre étages a été refait, et le nouveau matériel est posé.

Vous venez d'être embauché comme ingénieur sécurité réseau. Le DSI vous demande un **POC** illustrant la mise en place d'une infrastructure répondant aux standards actuels : haute disponibilité, segmentation, et politique Zero Trust.

Le pare-feu de bordure viendra dans un second temps. Cette première étape se concentre sur la **sécurisation du LAN**. Le budget étant limité, vous avez proposé une architecture **2 tiers (collapsed core)** — niveau core et niveau distribution fusionnés. C'est adapté pour un campus de cette taille : une paire de distribution, un bloc datacenter, et l'accès Internet.

Pour rester abordable, l'accès et le datacenter restent en **L2**, et la distribution est en **L3**.

> **Question 1 :** pourquoi avoir choisi des switches **L3** pour la distribution plutôt que des switches L2 + un routeur ? Donnez au moins deux raisons (performance, coût, complexité…).

> **Question 2 :** pourquoi avoir demandé que les switches de distribution et les switches d'accès du datacenter soient **en paire** ? Qu'est-ce qu'on perd si on n'en met qu'un ?

### Plan VLAN

Le plan complet prévoit aussi voix, invités, transit, etc., mais pour le POC on garde l'essentiel :

| VLAN | Nom | Rôle |
| --- | --- | --- |
| 10 | MGMT | Gestion des switches (SSH, SNMP, syslog) |
| 20 | SERVERS | Serveurs du datacenter |
| 30 | COLLABORATEUR | Postes des collaborateurs |
| 40 | ETUDIANT | Postes étudiants |
| 99 | NATIVE | VLAN natif des trunks (jamais utilisé pour du trafic) |
| 900 | INTERNAL_TRANSIT | Transit L3 entre DSW-01 et DSW-02 (voir bonus 2) |
| 999 | BLACKHOLE | VLAN poubelle pour les interfaces inutilisées |

### Plan d'adressage IP

Pas trop large (un /16 sur un LAN utilisateur = risque de broadcast storm, surface de scan énorme, gaspillage), pas trop petit (pas d'anticipation = repartir à zéro dans 2 ans). On fait du **VLSM** :

| VLAN | Subnet | Masque | Hosts utilisables |
| --- | --- | --- | --- |
| 10 MGMT | 10.31.10.0/27 | 255.255.255.224 | 30 |
| 20 SERVERS | 10.31.20.0/27 | 255.255.255.224 | 30 |
| 30 COLLAB | 10.31.30.0/25 | 255.255.255.128 | 126 |
| 40 ETUDIANT | 10.31.40.0/22 | 255.255.252.0 | 1022 |

> **Question 3 :** pourquoi un plan IP trop large est-il un problème de sécurité ? Pensez à ce que voit un attaquant qui scanne, et à ce que voit un broadcast.

> **Question 4 :** la marge entre les blocs (10.31.10, .20, .30, .40) est volontairement non collée. Quel est l'intérêt opérationnel ?

### Haute disponibilité

Redondance à deux niveaux :
- **Liens** : agrégation via **LACP** (standard IEEE 802.3ad). L'équivalent propriétaire Cisco est **PAgP** — on évite, on reste sur du standard.
- **Matériel + passerelle** : **HSRP** (propriétaire Cisco). L'équivalent standard est **VRRP** (RFC 5798), fonctionnellement très proche. HSRP utilise un **virtual IP** + **virtual MAC** partagés entre les deux switches, avec un actif et un standby.

### Topologie

Voir le diagramme fourni. Récapitulatif des matériels et liens :

- **DSW-01 / DSW-02** : paire distribution L3
  - Entre eux : 2 liens (e0/3 + e1/0) → futur Port-channel 2 (LACP)
  - Vers Internet : chacun e1/1 vers le switch "Internet" (cloud NAT VMware)
- **ASW-1** (Bâtiment A, L2) : uplinks e0/0→DSW-01 et e0/1→DSW-02 (futur Po1), e0/2 Étudiant, e0/3 Collab, e1/0 vers cloud "mgmt-sur-vlan10-pour-ssh"
- **ASW-2** (Bâtiment B, L2) : uplinks e0/0→DSW-01 et e0/1→DSW-02 (futur Po1), e0/2 Collab, e0/3 Adminsys
- **ASW-DC** (datacenter, L2) : uplinks e0/0→DSW-01 et e0/1→DSW-02 (futur Po1), e0/2 vers le serveur Linux

> **Note importante de nommage :** le switch du datacenter est nommé **ASW-DC** (Access Switch Datacenter). Ne le confondez pas avec DSW-01/DSW-02. Tous les switches d'accès commencent par `ASW-`, ceux de distribution par `DSW-`.

Les PCs TinyCore (Étudiant, Collab, Adminsys) démarrent en DHCP. Le serveur Linux dans le datacenter porte le DHCP (et plus tard DNS, FreeRADIUS, OpenLDAP — vu en session 3/4).

---

## Étape 0 — Préparation du lab

Ouvrez votre instance EVE-NG, importez le fichier topologie fourni, et démarrez tous les nœuds. Vérifiez que chaque switch est accessible via le port console.

> **Rappel utile :** pour vous connecter à un nouveau switch jamais configuré, la méthode #1 reste **le port console**. SSH n'est pas encore configuré, IP n'est pas encore définie. Une fois le L2 en place et une IP de management posée, on basculera sur SSH.

Niveaux de prompt :
- `>` : mode utilisateur, on peut faire des `show`, c'est tout.
- `#` : mode privilégié, accès complet à la lecture et aux commandes opérationnelles.
- `(config)#` : mode configuration global, accès aux modifications.

```text
enable
configure terminal
```

Et pour sauvegarder, deux écritures équivalentes :

```text
write memory
! ou
copy running-config startup-config
```

> **À retenir :** vos commandes ne sont **pas** persistantes tant que vous n'avez pas sauvegardé. Un `reload` perd tout. Prenez le réflexe de `write` après chaque étape validée.

---

## Étape 1 — Hostnames et premier durcissement

Avant toute autre chose, donnez à chaque équipement son hostname, et appliquez quelques bases de sécurité device qui ne coûtent rien et qui font une vraie différence si quelqu'un branche un câble console sur un switch oublié dans un placard.

Sur **chaque switch** (adapter le hostname) :

```text
enable
configure terminal

hostname DSW-01

! Mot de passe enable - chiffré (pas le vieux 'enable password' en clair)
enable secret Ynov-2026!

! Chiffrer aussi les mots de passe restants dans la config
service password-encryption

! Bannière légale - obligation dans pas mal de juridictions pour
! poursuivre un intrus
banner motd ^
*****************************************************
* ACCES RESERVE - Ynov Toulouse                     *
* Toute connexion non autorisee fera l objet de     *
* poursuites. Activite tracee et journalisee.       *
*****************************************************
^

! Console : mot de passe obligatoire, déconnexion auto si inactif
line console 0
 password Cons0le-Ynov!
 login
 exec-timeout 10 0
 logging synchronous
 exit

! VTY (telnet/SSH entrant) : on ferme tout pour l'instant.
! SSH sera ouvert plus loin une fois la crypto en place.
line vty 0 15
 transport input none
 exec-timeout 10 0
 exit

end
write memory
```

> **Question 5 :** pourquoi `enable secret` plutôt que `enable password` ? Cherchez la différence d'algo de stockage.

> **Question 6 :** `transport input none` sur les VTY — qu'est-ce que ça bloque concrètement, et pourquoi c'est une bonne valeur par défaut tant que le SSH n'est pas configuré ?

> **Question 7 :** pourquoi `logging synchronous` sur la console rend la vie plus agréable quand vous tapez ?

📸 **Capture demandée** : `show running-config | section line` sur un des switches.

---

## Étape 2 — Création des VLANs

Sur **chaque switch**, créez les VLANs **nécessaires à ce switch**. Pas plus.

Exemple sur DSW-01 (porte tous les VLANs car il fait du routage inter-VLAN) :

```text
enable
configure terminal

vlan 10
 name MGMT
 exit
vlan 20
 name SERVERS
 exit
vlan 30
 name COLLAB
 exit
vlan 40
 name ETUDIANT
 exit
vlan 99
 name NATIVE
 exit
vlan 999
 name BLACKHOLE
 exit

end
write memory
```

Sur **ASW-1** par exemple : VLAN 30, 40, 10 (pour le management du switch lui-même), 99 et 999. **Pas le 20**, il ne traverse pas ce switch.

> **Question 8 :** pourquoi est-ce une mauvaise pratique de créer tous les VLANs sur tous les switches "au cas où" ? Pensez à VTP, à la table CAM, et à la surface d'attaque (un VLAN qui existe peut être joint si quelqu'un trouve comment).

📸 **Capture demandée** : `show vlan brief` sur DSW-01 et sur ASW-1.

---

## Étape 3 — Ports d'accès

Sur chaque switch d'accès, les ports clients sont en mode **access** (équivalent *untagged* — le tag 802.1Q est retiré avant d'envoyer la trame au PC).

Sur **ASW-1** :

```text
enable
configure terminal

! Étudiant - VLAN 40
interface ethernet 0/2
 description Poste Etudiant
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
 exit

! Collab - VLAN 30
interface ethernet 0/3
 description Poste Collaborateur
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
 exit

end
write memory
```

Faites pareil sur **ASW-2** (e0/2 Collab VLAN 30, e0/3 Adminsys VLAN 10) et sur **ASW-DC** (e0/2 serveur Linux VLAN 20).

> **À noter sur `portfast` + `bpduguard` :**
> - `portfast` saute les phases listening/learning de STP — le port passe directement en forwarding. Sans ça, un PC peut attendre ~30 secondes avant d'avoir le lien réseau utilisable, et un client DHCP peut timeout.
> - **`bpduguard`** : si jamais un BPDU arrive sur ce port (typiquement parce que quelqu'un a branché un mini-switch perso ou tente une attaque STP), le port est mis en `err-disabled` immédiatement. Un poste utilisateur n'a aucune raison légitime d'envoyer des BPDU.
> - Les deux ensemble = "ce port est pour un endpoint, point". C'est un standard absolu sur tous les ports d'accès.

> **Question 9 :** quelle attaque STP `bpduguard` protège-t-il contre ? Cherchez "STP root bridge attack".

📸 **Capture demandée** : `show interfaces status` sur ASW-1.

> **À propos de `show ip interface brief` :** les colonnes `Status` et `Protocol` reflètent deux niveaux :
> - **Status** = état L1 (le câble est-il branché, l'interface administrativement up ?)
> - **Protocol** = état L2 (le keepalive / la négociation passe-t-elle ?)
>
> Si Status=up mais Protocol=down, c'est typiquement un souci L2 (mismatch encapsulation, duplex, partenaire qui ne répond pas).

Contrairement aux routeurs, les interfaces des switches sont **allumées par défaut**. Vous pouvez forcer manuellement avec `shutdown` / `no shutdown`.

📸 **Capture demandée** : `show ip interface brief` sur ASW-1.

---

## Étape 4 — Le VLAN BLACKHOLE pour les ports inutilisés

Les ports non utilisés sur un switch sont une porte d'entrée gratuite pour quiconque a un accès physique. Règle : tout port non assigné → access VLAN 999 (BLACKHOLE) + shutdown.

Sur **chaque switch**, identifiez les ports inutilisés et :

```text
configure terminal

! Exemple sur ASW-1, supposons que e1/1, e1/2, e1/3 sont inutilisés
interface range ethernet 1/1 - 3
 description UNUSED - BLACKHOLE
 switchport mode access
 switchport access vlan 999
 shutdown
 exit

end
write memory
```

> **Question 10 :** un attaquant branche un câble sur un port qui est `shutdown` ET dans BLACKHOLE. Que se passe-t-il pour lui ? Et si on avait juste fait `shutdown` sans changer le VLAN ?

📸 **Capture demandée** : `show interfaces status | include 999` sur un des switches.

---

## Étape 5 — Trunks entre ASW et DSW (sans LACP pour l'instant)

Les liens montants des switches d'accès vers les switches de distribution doivent porter plusieurs VLANs : ils sont en mode **trunk** (équivalent *tagged* — chaque trame conserve son tag 802.1Q sauf celles du VLAN natif).

Deux encapsulations possibles historiquement : **dot1q** (standard IEEE 802.1Q) et **ISL** (Inter-Switch Link, propriétaire Cisco, mort). Sur les IOS récents seul dot1q est dispo, mais sur certains modèles plus anciens il faut le préciser explicitement, donc on le fait par habitude.

Avant de bundler en LACP, on va d'abord configurer **chaque lien individuellement** pour vérifier que le trunk monte. On bundlera ensuite à l'étape suivante.

Sur **ASW-1**, interfaces e0/0 (vers DSW-01) et e0/1 (vers DSW-02) :

```text
configure terminal

interface range ethernet 0/0 - 1
 description Uplink vers DSW
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,40
 switchport nonegotiate
 no shutdown
 exit

end
write memory
```

Côté **DSW-01**, interface e0/1 (vers ASW-1) — symétrique :

```text
configure terminal

interface ethernet 0/1
 description Vers ASW-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,40
 switchport nonegotiate
 no shutdown
 exit

end
```

Refaites le même travail pour tous les liens ASW ↔ DSW (consultez le diagramme pour les interfaces). Adaptez les VLANs autorisés à chaque switch :
- **ASW-1** : 10, 30, 40
- **ASW-2** : 10, 30
- **ASW-DC** : 10, 20

> **Question 11 :** pourquoi limiter les VLANs autorisés sur chaque trunk au strict nécessaire ? Quelle attaque facilite-t-on en permettant tous les VLANs partout ?

> **Question 12 :** le VLAN natif passe **non taggé** sur le trunk. Quelle attaque exploite ça si VLAN natif = VLAN 1 (la valeur par défaut) ? Cherchez "VLAN hopping double tagging".

> **Question 13 :** que se passe-t-il si DSW-01 a `native vlan 99` mais que ASW-1 a `native vlan 1` (oubli) ? Quelle alerte voit-on dans `show interfaces trunk` ?

> **À propos de `switchport nonegotiate` :** cette commande désactive DTP (Dynamic Trunking Protocol). DTP négocie automatiquement le mode trunk avec le voisin — pratique mais c'est aussi le vecteur de l'attaque "switch spoofing". Comme on a fixé le mode en dur (`switchport mode trunk`), on n'a plus besoin de DTP, et on coupe par sécurité. (On reverra DTP en détail en session 3 ou 4.)

📸 **Capture demandée** : `show interfaces trunk` sur DSW-01 (doit lister les trois trunks vers ASW-1, ASW-2, ASW-DC avec le bon native VLAN et les bons VLANs allowed).

---

## Étape 6 — Agrégation des uplinks en LACP (Port-channel 1)

Maintenant qu'on a validé que les trunks individuels fonctionnent, on les **bundle** en un canal logique unique avec LACP. LACP est le standard IEEE 802.3ad (équivalent propriétaire Cisco : PAgP, qu'on évite).

> **Limite de notre lab :** LACP suppose que les deux extrémités d'un bundle sont sur **le même switch physique** (ou logique si on est en stack). Or DSW-01 et DSW-02 sont deux boîtes séparées et nos IOS dans EVE-NG ne supportent pas le stacking. Donc on **ne peut pas** faire un Po unique entre ASW-1 et DSW-01+DSW-02. À la place, ASW-1 a deux uplinks séparés (e0/0 vers DSW-01, e0/1 vers DSW-02), et STP s'occupe d'éviter la boucle.
>
> Là où on **peut** faire du LACP : entre DSW-01 et DSW-02 (2 liens), parce que les deux extrémités du Po sont sur le même switch à chaque bout. Et plus tard, entre le serveur Linux et ASW-DC (bonus 1) si on met deux NICs au serveur.

> **Question 14 :** donnez un autre exemple concret en production de cas où LACP s'applique bien (deux extrémités sur le même switch ou paire stackée).

### Po2 — entre DSW-01 et DSW-02

Sur **DSW-01**, e0/3 et e1/0 sont les liens vers DSW-02 :

```text
configure terminal

interface range ethernet 0/3, ethernet 1/0
 description Vers DSW-02 (membre Po2)
 channel-group 2 mode active
 no shutdown
 exit

interface Port-channel 2
 description Lien inter-DSW (vers DSW-02)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40
 switchport nonegotiate
 exit

end
write memory
```

Sur **DSW-02**, configuration miroir (e0/3 et e1/0) :

```text
configure terminal

interface range ethernet 0/3, ethernet 1/0
 description Vers DSW-01 (membre Po2)
 channel-group 2 mode active
 no shutdown
 exit

interface Port-channel 2
 description Lien inter-DSW (vers DSW-01)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40
 switchport nonegotiate
 exit

end
write memory
```

> **À propos de `channel-group X mode active` :** `active` = LACP, négociation symétrique. Autres modes :
> - `passive` = LACP mais n'initie pas (deux extrémités passive = pas de bundle).
> - `on` = bundle forcé sans protocole de négociation. Dangereux : si une extrémité est `on` et l'autre `active`, le bundle ne se forme pas.
> - `desirable` / `auto` = PAgP, à éviter (propriétaire).
>
> **Règle simple :** `active` des deux côtés. Toujours.

📸 **Capture demandée** : `show etherchannel summary` sur DSW-01. Vous devez voir `Po2(SU)` (S = L2, U = in-use) et les deux membres en `(P)` (bundled in port-channel).

> **Question 15 :** que veut dire `(P)` à côté du nom d'interface dans `show etherchannel summary` ? Et `(D)` ? `(s)` ?

---

## Étape 7 — Routage L3 et SVIs sur les DSW

C'est ici qu'on quitte le pur L2. Le routage inter-VLAN se fait sur les switches de distribution via les **SVIs** (Switch Virtual Interfaces) : une interface virtuelle par VLAN, avec une IP, qui sert de passerelle pour les hosts de ce VLAN.

Avant d'activer le routage, regardez la table actuelle :

```text
show ip route
```

📸 **Capture demandée** : `show ip route` sur DSW-01 **avant** activation du routage.

Vous voyez "Default gateway is not set" ou une table quasi vide : c'est normal, le switch n'est pas encore en L3.

### Activation du routage

Sur **DSW-01** et **DSW-02** :

```text
configure terminal

! Active le routage IP - le switch peut maintenant router entre SVIs
ip routing

! Désactive CEF - sur IOL/IOSv en VM dans EVE-NG, CEF casse le routage
! car le hardware FIB n'existe pas. Sur du vrai matériel, on garde CEF activé.
no ip cef

end
```

### Création des SVIs

Sur **DSW-01** (qui sera HSRP **active** pour VLANs 10 et 30, **standby** pour 20 et 40) :

```text
configure terminal

interface vlan 10
 description SVI MGMT
 ip address 10.31.10.1 255.255.255.224
 no shutdown
 exit

interface vlan 20
 description SVI SERVERS
 ip address 10.31.20.1 255.255.255.224
 no shutdown
 exit

interface vlan 30
 description SVI COLLAB
 ip address 10.31.30.1 255.255.255.128
 no shutdown
 exit

interface vlan 40
 description SVI ETUDIANT
 ip address 10.31.40.1 255.255.252.0
 no shutdown
 exit

end
write memory
```

Sur **DSW-02**, les **mêmes VLANs** mais avec l'IP `.2` au lieu de `.1` partout :

```text
configure terminal

interface vlan 10
 description SVI MGMT
 ip address 10.31.10.2 255.255.255.224
 no shutdown
 exit

interface vlan 20
 description SVI SERVERS
 ip address 10.31.20.2 255.255.255.224
 no shutdown
 exit

interface vlan 30
 description SVI COLLAB
 ip address 10.31.30.2 255.255.255.128
 no shutdown
 exit

interface vlan 40
 description SVI ETUDIANT
 ip address 10.31.40.2 255.255.252.0
 no shutdown
 exit

end
write memory
```

📸 **Capture demandée** : `show ip route connected` sur DSW-01 **après** activation. Vous devez voir les 4 subnets en `C` (connected).

> **Question 16 :** pour l'instant les hosts ne savent pas encore quelle IP utiliser comme passerelle. On a `.1` sur DSW-01 et `.2` sur DSW-02. Mettre `.1` sur les PCs = pas de redondance, et inversement. Quelle est la solution ? (Indice : ça commence par H et c'est dans le titre de l'étape suivante.)

---

## Étape 8 — STP root bridges

Avant HSRP, fixons proprement les **root bridges** STP. Sans intervention, STP élit le switch avec le BID le plus bas — souvent un switch d'accès parce qu'il a une MAC plus ancienne, ce qui donne des chemins absurdes.

**Convention :** sur chaque paire VLAN, le DSW HSRP **active** est aussi le **root primary** STP. C'est cohérent : le trafic L2 et le trafic routé prennent le même switch en mode normal.

Sur **DSW-01** :

```text
configure terminal

! DSW-01 = root primary pour VLANs où il sera HSRP active
spanning-tree vlan 10,30 root primary

! DSW-01 = root secondary pour VLANs où DSW-02 sera HSRP active
spanning-tree vlan 20,40 root secondary

end
write memory
```

Sur **DSW-02** :

```text
configure terminal

spanning-tree vlan 20,40 root primary
spanning-tree vlan 10,30 root secondary

end
write memory
```

📸 **Capture demandée** : `show spanning-tree vlan 30 brief` sur DSW-01 (vous devez être root).

> **Question 17 :** pourquoi aligner STP root et HSRP active sur le même DSW ? Que se passerait-il s'ils étaient désalignés (STP root = DSW-01, HSRP active = DSW-02 pour le même VLAN) ?

---

## Étape 9 — HSRP : passerelle redondante

**Concept :** HSRP (Hot Standby Router Protocol) permet à deux routeurs/switches L3 de partager une **IP virtuelle** (VIP) et une **MAC virtuelle**. Les hosts utilisent la VIP comme passerelle. À un instant T, un seul des deux switches est **active** (forward) ; l'autre est **standby** (silencieux, mais prêt). Si l'active tombe, le standby prend le relais en une à trois secondes (hellos toutes les secondes par défaut).

**Pourquoi un mode active/standby croisé entre les deux DSW ?**
Si DSW-01 est active pour **tous** les VLANs, DSW-02 ne fait rien en mode normal — son CPU et sa bande passante sont gâchés. En croisant (DSW-01 active pour 10+30, DSW-02 active pour 20+40), on **load-balance** : chaque DSW porte la moitié du trafic en temps normal, et l'autre prend le relais complet en cas de panne.

### Plan d'adressage HSRP

Convention : `.1` = DSW-01, `.2` = DSW-02, `.3` = VIP (la passerelle vue par les hosts).

| VLAN | Subnet | DSW-01 | DSW-02 | VIP (passerelle hosts) | Active |
| --- | --- | --- | --- | --- | --- |
| 10 MGMT | 10.31.10.0/27 | .1 | .2 | **10.31.10.3** | DSW-01 |
| 20 SERVERS | 10.31.20.0/27 | .1 | .2 | **10.31.20.3** | DSW-02 |
| 30 COLLAB | 10.31.30.0/25 | .1 | .2 | **10.31.30.3** | DSW-01 |
| 40 ETUDIANT | 10.31.40.0/22 | .1 | .2 | **10.31.40.3** | DSW-02 |

### Configuration sur DSW-01

```text
configure terminal

! VLAN 10 - DSW-01 ACTIVE (priorité 110 > défaut 100)
interface vlan 10
 standby version 2
 standby 10 ip 10.31.10.3
 standby 10 priority 110
 standby 10 preempt
 standby 10 authentication md5 key-string Ynov-HSRP-2026!
 exit

! VLAN 20 - DSW-01 STANDBY (priorité 100)
interface vlan 20
 standby version 2
 standby 20 ip 10.31.20.3
 standby 20 priority 100
 standby 20 preempt
 standby 20 authentication md5 key-string Ynov-HSRP-2026!
 exit

! VLAN 30 - DSW-01 ACTIVE
interface vlan 30
 standby version 2
 standby 30 ip 10.31.30.3
 standby 30 priority 110
 standby 30 preempt
 standby 30 authentication md5 key-string Ynov-HSRP-2026!
 exit

! VLAN 40 - DSW-01 STANDBY
interface vlan 40
 standby version 2
 standby 40 ip 10.31.40.3
 standby 40 priority 100
 standby 40 preempt
 standby 40 authentication md5 key-string Ynov-HSRP-2026!
 exit

end
write memory
```

### Configuration sur DSW-02 (priorités inversées)

```text
configure terminal

interface vlan 10
 standby version 2
 standby 10 ip 10.31.10.3
 standby 10 priority 100
 standby 10 preempt
 standby 10 authentication md5 key-string Ynov-HSRP-2026!
 exit

interface vlan 20
 standby version 2
 standby 20 ip 10.31.20.3
 standby 20 priority 110
 standby 20 preempt
 standby 20 authentication md5 key-string Ynov-HSRP-2026!
 exit

interface vlan 30
 standby version 2
 standby 30 ip 10.31.30.3
 standby 30 priority 100
 standby 30 preempt
 standby 30 authentication md5 key-string Ynov-HSRP-2026!
 exit

interface vlan 40
 standby version 2
 standby 40 ip 10.31.40.3
 standby 40 priority 110
 standby 40 preempt
 standby 40 authentication md5 key-string Ynov-HSRP-2026!
 exit

end
write memory
```

**Explication des commandes nouvelles :**
- `standby version 2` : HSRPv2 supporte plus de groupes (0–4095 vs 0–255 en v1), MAC virtuelle différente, IPv6. À utiliser systématiquement.
- `standby X ip Y` : VIP du groupe HSRP numéro X. Convention : numéro de groupe = numéro de VLAN, pour la lisibilité.
- `standby X priority N` : 100 par défaut, le plus haut gagne.
- `standby X preempt` : sans ça, si l'active tombe puis revient, il ne reprend pas son rôle. Avec preempt, dès qu'il revient et que sa priorité est supérieure, il **récupère** le rôle active. Important pour rester sur le plan de design prévu.
- `standby X authentication md5 key-string ...` : empêche un attaquant qui se branche sur le segment d'envoyer un hello HSRP avec priorité 255 et de détourner le rôle active (= MITM passerelle). **Aucune raison de ne pas l'activer.**

📸 **Capture demandée** : `show standby brief` sur DSW-01 et DSW-02. Vous devez voir le rôle Active/Standby cohérent avec le plan.

### Interface tracking — important

Tel quel, HSRP **ne sait pas** si l'uplink Internet est tombé. DSW-01 reste active pour VLAN 30 même si son lien Internet est mort → les paquets des collaborateurs partent dans le vide.

On va corriger ça en suivant l'état de l'interface uplink Internet (e1/1). Si l'interface tombe, on **décrémente la priorité HSRP** de 20 → la priorité chute en dessous de celle du voisin → preempt déclenche → DSW-02 devient active → le trafic ressort par le bon chemin.

Sur **DSW-01** :

```text
configure terminal

! Définit un objet "track" qui suit l'état L2 de e1/1
track 1 interface ethernet 1/1 line-protocol

! On rattache ce track aux groupes HSRP où DSW-01 est active.
! Décrément 20 fait passer 110 -> 90, en dessous de 100 chez DSW-02.
interface vlan 10
 standby 10 track 1 decrement 20
 exit
interface vlan 30
 standby 30 track 1 decrement 20
 exit

end
write memory
```

Sur **DSW-02** (symétrique, sur les VLANs où il est active) :

```text
configure terminal

track 1 interface ethernet 1/1 line-protocol

interface vlan 20
 standby 20 track 1 decrement 20
 exit
interface vlan 40
 standby 40 track 1 decrement 20
 exit

end
write memory
```

> **Question 18 :** sans interface tracking, décrivez ce qui se passe pour un collaborateur (VLAN 30) si DSW-01 perd son uplink Internet mais que tout le reste fonctionne. Pourquoi ses requêtes vers Google échouent ?

> **Question 19 :** pourquoi décrémenter de 20 et pas de 5 ? Calculez ce que devient la priorité de DSW-01 après décrément, et comparez à celle de DSW-02.

📸 **Capture demandée** : `show standby` (sans le `brief`, version détaillée) sur DSW-01 pour le VLAN 30. Vous devez voir la ligne `Track object 1 state Up decrement 20`.

---

## Étape 10 — Configurer le DHCP relay sur les SVIs utilisateurs

Le serveur DHCP est dans le VLAN 20 (10.31.20.10 sur le serveur Linux). Les PCs en VLAN 30 et 40 sont dans des broadcast domains différents — leurs requêtes DHCP `DISCOVER` (broadcast L2 + L3) ne franchissent pas les frontières L3 sans aide.

**Solution :** `ip helper-address` sur le SVI du VLAN client. Le DSW reçoit le DISCOVER en broadcast, le réencapsule en unicast vers l'IP du serveur DHCP, et relaie la réponse en retour.

Sur **DSW-01 ET DSW-02** (les deux doivent relayer, sinon en cas de failover le DHCP casse) :

```text
configure terminal

interface vlan 30
 ip helper-address 10.31.20.10
 exit

interface vlan 40
 ip helper-address 10.31.20.10
 exit

end
write memory
```

> **Question 20 :** pourquoi le DHCP relay doit être configuré sur **les deux** DSW, et pas seulement sur l'active HSRP ?

📸 **Capture demandée** : `show ip interface vlan 30` sur DSW-01 (vous devez voir la ligne "Helper address is 10.31.20.10").

---

## Étape 11 — Configurer les PCs et tester

Sur les PCs TinyCore, les VLAN 30 et 40 démarrent en DHCP. Adminsys (VLAN 10) est en statique.

**Adminsys (TinyCore en VLAN 10)** :
```sh
sudo ifconfig eth0 10.31.10.20 netmask 255.255.255.224 up
sudo route add default gw 10.31.10.3
```

**Collab (VLAN 30) et Étudiant (VLAN 40)** : laissez le client DHCP faire son travail.
```sh
sudo udhcpc -i eth0
```

Une fois les leases reçus :

```sh
ip addr show eth0
ip route
ping -c 3 10.31.30.3       # passerelle VIP (depuis Collab)
ping -c 3 10.31.40.3       # passerelle VIP (depuis Étudiant)
ping -c 3 10.31.20.10      # serveur Linux
```

> **Question 21 :** depuis Collab (VLAN 30), pingez Étudiant (VLAN 40). Ça doit fonctionner. Sur quel DSW le paquet routé est-il passé ? (Indice : `traceroute` ou regardez le plan HSRP active.)

📸 **Capture demandée** : sortie de `ip addr show eth0` et `ping` réussi vers la passerelle, depuis un PC Collab et un PC Étudiant.

---

## Étape 12 — Lien Internet et HSRP côté outside

Le switch "Internet" sur le diagramme est en réalité un unmanaged switch (présenté sous icône routeur) qui ponte les interfaces e1/1 des deux DSW vers le réseau **NAT VMware Workstation Pro** (typiquement 192.168.29.0/24).

> **Vérification préalable :** sur votre VMware Workstation, **éditez le réseau NAT** ("Virtual Network Editor" → VMnet8 ou équivalent) et notez la plage exacte (parfois 192.168.X.0/24 où X dépend de votre install). La gateway par défaut donnée par VMware est souvent .2. L'adresse .1 est parfois prise par l'hôte. **Vérifiez avant de configurer** — ajustez les exemples ci-dessous si votre plage est différente.

Plan d'adressage côté Internet (en supposant 192.168.29.0/24, GW VMware = .2, et en évitant le bas de plage potentiellement utilisé) :

| Rôle | IP |
| --- | --- |
| Gateway VMware (NAT) | 192.168.29.2 |
| DSW-01 e1/1 | 192.168.29.101 |
| DSW-02 e1/1 | 192.168.29.102 |
| VIP HSRP outside | **192.168.29.100** |

Sur le SVI, on a déjà géré la redondance L3. Pour le lien outside on bascule en **interface routée** (no switchport) car ce n'est pas un VLAN du LAN — c'est un point-à-point vers le NAT.

### Sur DSW-01

```text
configure terminal

interface ethernet 1/1
 description Uplink Internet (vers cloud NAT)
 no switchport
 ip address 192.168.29.101 255.255.255.0
 standby version 2
 standby 100 ip 192.168.29.100
 standby 100 priority 110
 standby 100 preempt
 standby 100 authentication md5 key-string Ynov-HSRP-2026!
 no shutdown
 exit

! Route par défaut vers la VIP HSRP de la gateway VMware
ip route 0.0.0.0 0.0.0.0 192.168.29.2

end
write memory
```

### Sur DSW-02

```text
configure terminal

interface ethernet 1/1
 description Uplink Internet (vers cloud NAT)
 no switchport
 ip address 192.168.29.102 255.255.255.0
 standby version 2
 standby 100 ip 192.168.29.100
 standby 100 priority 100
 standby 100 preempt
 standby 100 authentication md5 key-string Ynov-HSRP-2026!
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 192.168.29.2

end
write memory
```

**Pourquoi `no switchport` ?** Par défaut une interface de switch L3 est en mode L2 (elle attend des trames taggées ou non, elle ne peut pas porter d'IP directement). `no switchport` la convertit en **interface routée** — elle se comporte comme une interface de routeur classique, capable de porter une IP. Pas de VLAN, pas de SVI : c'est du L3 pur.

> **Question 22 :** quelle est la différence pratique entre `interface vlan 10` (SVI) et `interface ethernet 1/1 / no switchport / ip address ...` (routed) ? Donnez un cas d'usage où chacune est préférable.

📸 **Capture demandée** : `show ip route` sur DSW-01 (vous devez voir la route par défaut + les subnets connectés + les SVIs).

> **Question 23 :** dans `show ip route`, les lignes commencent par une lettre : `C` pour connected, `S` pour static, `L` pour local... Pourquoi voit-on à la fois `C` et `L` pour le même subnet ? (Cherchez la différence connected vs local.)

### Test depuis un PC

```sh
ping -c 3 192.168.29.2        # gateway VMware
ping -c 3 8.8.8.8              # internet (si votre NAT VMware route)
```

---

## Étape 13 — IP de management des switches L2

Les switches d'accès (ASW-1, ASW-2, ASW-DC) sont en L2 — ils n'ont pas de SVI routé. On leur donne **une seule IP** dans le VLAN 10 (MGMT) pour SSH/SNMP/syslog, et une default gateway pointant sur la VIP HSRP.

Convention : adresses finissant en `.10X` pour les ASW.

| Switch | IP mgmt |
| --- | --- |
| ASW-1 | 10.31.10.101/27 |
| ASW-2 | 10.31.10.102/27 |
| ASW-DC | 10.31.10.103/27 |

Sur **ASW-1** :

```text
configure terminal

interface vlan 10
 description Mgmt SVI
 ip address 10.31.10.101 255.255.255.224
 no shutdown
 exit

! Sur un switch L2, on n'a pas 'ip route' mais 'ip default-gateway'
ip default-gateway 10.31.10.3

end
write memory
```

Refaire pareil sur **ASW-2** (101 → 102) et **ASW-DC** (101 → 103).

> **Question 24 :** sur un switch en mode L2, pourquoi utilise-t-on `ip default-gateway` au lieu de `ip route 0.0.0.0 0.0.0.0 X.X.X.X` ? Cherchez ce que change `ip routing` activé ou non.

### Test

Depuis le PC Adminsys (10.31.10.20), pingez les trois ASW :

```sh
ping -c 2 10.31.10.101    # ASW-1
ping -c 2 10.31.10.102    # ASW-2
ping -c 2 10.31.10.103    # ASW-DC
ping -c 2 10.31.10.1      # SVI DSW-01
ping -c 2 10.31.10.2      # SVI DSW-02
ping -c 2 10.31.10.3      # VIP
```

📸 **Capture demandée** : pings réussis depuis Adminsys vers les 3 ASW.

---

## Étape 14 — Activer SSH sur les switches

Avant la session ACL de cet après-midi, vous avez besoin de SSH actif sur **tous les switches**. Pas encore d'authentification RADIUS (session 3) — on reste en local users.

Sur **chaque switch** (DSW et ASW) :

```text
configure terminal

! Nom de domaine obligatoire pour générer la clé RSA
ip domain-name ynov-toulouse.lab

! Génération de la clé RSA - 2048 bits minimum
crypto key generate rsa modulus 2048

! Force SSHv2 (SSHv1 a des failles connues, ne jamais l'autoriser)
ip ssh version 2

! Local user (mot de passe chiffré scrypt si dispo, sinon MD5)
username admin privilege 15 secret Ynov-Admin-2026!

! Active l'authentification locale sur les VTY et n'autorise QUE SSH
line vty 0 15
 login local
 transport input ssh
 exec-timeout 10 0
 exit

end
write memory
```

> **Question 25 :** pourquoi `transport input ssh` et pas `transport input ssh telnet` (les deux) ? Quelle erreur classique fait-on en laissant telnet activé "au cas où" ?

> **Question 26 :** la commande `crypto key generate rsa modulus 2048` — pourquoi 2048 et pas 512 (qui est plus rapide à générer) ?

📸 **Capture demandée** : `show ip ssh` sur DSW-01 (doit afficher SSH Enabled - version 2.0).

### Tester SSH depuis votre PC hôte

Pour vérifier que SSH fonctionne vraiment depuis une station extérieure à EVE-NG (et pas seulement depuis un TinyCore interne au lab), voici une astuce **pratique et propre** :

1. **Sur VMware Workstation Pro** → menu **Edit → Virtual Network Editor** → bouton **Add Network**, choisir un **VMnet** libre (ex: VMnet2), type **Host-only**.
2. Configurer ce VMnet en **DHCP désactivé** et lui donner un subnet dans la plage MGMT, par exemple : `10.31.10.0/27`. Donnez à votre **carte hôte sur ce VMnet** une IP libre, par exemple `10.31.10.50/27`.
3. Dans **EVE-NG**, modifiez la config de la VM EVE-NG : ajoutez un nouveau **NIC** rattaché à ce VMnet2.
4. Redémarrez EVE-NG. Dans l'interface web EVE-NG, ce nouveau NIC apparaît comme `Cloud1` (ou Cloud2 selon l'ordre).
5. Sur la topologie de votre lab, ajoutez un nœud **Network → Cloud1**, connectez-le à un port libre d'un switch (par exemple **ASW-1 e1/0** comme prévu sur le diagramme, étiqueté "mgmt-sur-vlan10-pour-ssh").
6. Côté switch, mettez ce port en **access VLAN 10** :
   ```text
   interface ethernet 1/0
    description Cloud1 - Acces SSH depuis machine hote
    switchport mode access
    switchport access vlan 10
    spanning-tree portfast
    spanning-tree bpduguard enable
    no shutdown
    exit
   ```

Maintenant, depuis votre laptop directement (terminal Windows/Mac/Linux) :
```sh
ssh admin@10.31.10.101
```

> **Avantages de cette méthode :**
> - Vous gardez votre console accessible en parallèle pour debug.
> - Vous testez SSH dans des conditions réalistes (client externe au lab).
> - Vous pouvez copier-coller depuis votre laptop sans passer par les sous-fenêtres TinyCore.

📸 **Capture demandée** : session SSH réussie depuis votre laptop vers **un** switch.

> **Note de sécurité :** on est encore en authentification **locale** avec mot de passe unique sur tous les équipements. C'est dur à scaler et auditer. En session 3 on basculera sur RADIUS pour centraliser l'authentification admin avec des comptes nominatifs et de l'accounting.

---

## Test de failover HSRP

C'est le moment de vérité — votre HD est-elle vraiment H ?

1. Depuis un PC Collab (VLAN 30), lancez un ping continu vers Internet ou la VIP outside :
   ```sh
   ping 192.168.29.100
   ```

2. Sur **DSW-01** (HSRP active pour VLAN 30), forcez la bascule :
   ```text
   interface vlan 30
    standby 30 priority 50
    end
   ```

3. Observez : un à deux paquets perdus, puis le ping reprend. DSW-02 a pris le rôle active.

4. Sur **DSW-02** : `show standby brief` → VLAN 30 doit maintenant être en `Active`.

5. Remettez DSW-01 :
   ```text
   interface vlan 30
    no standby 30 priority 50
    end
   ```
   Comme `preempt` est configuré, DSW-01 reprend automatiquement le rôle active.

> **Test bonus :** au lieu de jouer sur la priorité, faites `shutdown` de l'interface e1/1 sur DSW-01 (uplink Internet). Grâce à l'interface tracking, la priorité chute, DSW-02 prend le rôle pour les VLANs concernés. Vérifiez avec `show standby brief`.

📸 **Capture demandée** : `show standby brief` après bascule, montrant DSW-02 en Active pour VLAN 30.

---

## Bonus 1 — LACP entre le serveur Linux et ASW-DC

Les serveurs critiques (et le serveur Ynov qui héberge bientôt DHCP + DNS + FreeRADIUS + OpenLDAP en est un) ne doivent pas tomber si un câble est arraché. Un serveur de production a typiquement **2 NICs** bondées en LACP vers le switch d'accès.

> **Question 27 :** pourquoi est-il particulièrement critique d'avoir de la redondance de lien sur un serveur d'infrastructure (DHCP, DNS, FreeRADIUS) par rapport à un poste utilisateur ? Que se passe-t-il sur le campus entier si ce serveur perd son lien réseau pendant 30 secondes ?

### Préparation côté serveur Linux

Sur le serveur, créez un bond Linux mode 802.3ad (LACP) :

```sh
# Charger le module (la plupart des distros l'ont déjà)
sudo modprobe bonding

# Créer le bond
sudo ip link add bond0 type bond mode 802.3ad miimon 100 lacp_rate fast

# Ajouter les deux NICs (adaptez les noms : eth0/eth1 ou ens33/ens34...)
sudo ip link set eth0 down
sudo ip link set eth1 down
sudo ip link set eth0 master bond0
sudo ip link set eth1 master bond0
sudo ip link set eth0 up
sudo ip link set eth1 up
sudo ip link set bond0 up

# Donner l'IP au bond (l'ancienne sur eth0 doit être retirée)
sudo ip addr add 10.31.20.10/27 dev bond0
sudo ip route add default via 10.31.20.3
```

Vérifiez l'état :
```sh
cat /proc/net/bonding/bond0
```

Vous devez voir `Bonding Mode: IEEE 802.3ad Dynamic link aggregation` et les deux slaves en `link: up`.

### Côté ASW-DC

Supposons que les deux liens vers le serveur sont **e0/2** et **e0/3** :

```text
configure terminal

interface range ethernet 0/2 - 3
 description Vers serveur Linux (membre Po3)
 channel-group 3 mode active
 no shutdown
 exit

interface Port-channel 3
 description Bundle LACP vers serveur Linux
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

end
write memory
```

**Notez bien :** ici Po3 est en **access** (pas trunk) — le serveur est mono-VLAN (VLAN 20). On garde portfast + bpduguard parce que c'est un port d'accès vers un endpoint, même si l'endpoint a deux pattes.

### Test

```sh
# Sur le serveur, ping vers la VIP HSRP
ping -c 3 10.31.20.3

# Arrachez (administrativement) un lien sur ASW-DC :
#   interface eth 0/2
#   shutdown
# Le ping continue. Remontez-le, l'autre tombe, ça continue.
```

📸 **Capture demandée** : `show etherchannel summary` sur ASW-DC (Po3 doit être `SU` avec les deux membres en `(P)`).

---

## Bonus 2 — VLAN 900 INTERNAL_TRANSIT et OSPF entre les DSW

C'est la décision d'architecture la plus subtile de tout ce TP. Lisez bien le contexte avant de configurer.

### Le problème : routage asymétrique pendant un failover

Considérez la situation **normale** sur votre lab tel que configuré :
- DSW-01 est HSRP **active** pour VLAN 30.
- DSW-02 est HSRP **active** pour VLAN 40.

Un PC en VLAN 40 ping un PC en VLAN 30 :

1. **Aller :** requête routée par DSW-02 (passerelle du source VLAN 40). DSW-02 a aussi un SVI dans VLAN 30 (même en standby), donc il forward dans VLAN 30 directement.
2. **Retour :** la réponse part du PC VLAN 30 vers sa passerelle = VIP 10.31.30.3 = DSW-01 (active VLAN 30). DSW-01 reçoit la réponse.

→ **Aller par DSW-02, retour par DSW-01.** C'est du **routage asymétrique**.

Tant que les deux DSW ont une route valide vers les deux subnets, ça passe. **Mais** pendant un test de failover, si on `shutdown` le SVI VLAN 40 sur DSW-01, le subnet 10.31.40.0/22 disparaît de la table de routage de DSW-01. Le retour pour le VLAN 40 arrive sur DSW-01 → pas de route → **drop silencieux** → blackhole.

### La solution : VLAN 900 comme transit L3 entre les DSW + OSPF

L'idée : créer un **VLAN dédié** entre les deux DSW, leur donner à chacun une IP routée dessus, et faire tourner un protocole de routage (**OSPF**) entre eux. Comme ça, chaque DSW apprend automatiquement les subnets que l'autre connaît, et peut router le trafic asymétrique via Po2 vers l'autre DSW.

> **Pourquoi ne pas simplement passer Po2 en `no switchport` (routed) ?** Parce que Po2 doit **rester un trunk L2** pour transporter les hellos HSRP de tous les VLANs et (plus tard, en session edge/firewall) le VLAN 100 stretché pour la réplication ASA failover. Donc Po2 reste L2, et VLAN 900 piggyback dessus comme transit L3 dédié.

### Configuration

**Plan d'adressage** pour VLAN 900 (petit /30, transit point-à-point) :
- DSW-01 : 10.31.99.1/30
- DSW-02 : 10.31.99.2/30

Sur **DSW-01** :

```text
configure terminal

vlan 900
 name INTERNAL_TRANSIT
 exit

! Ajouter VLAN 900 au trunk Po2
interface Port-channel 2
 switchport trunk allowed vlan 10,20,30,40,900
 exit

! SVI dans VLAN 900
interface vlan 900
 description Transit L3 vers DSW-02
 ip address 10.31.99.1 255.255.255.252
 ! Mode point-à-point évite l'élection DR/BDR (inutile sur du P2P et
 ! source de bugs sur les images IOL/IOSv en EVE-NG)
 ip ospf network point-to-point
 no shutdown
 exit

! OSPF process
router ospf 1
 router-id 1.1.1.1
 ! Active OSPF sur les SVIs : transit + tous les SVIs utilisateurs
 ! pour que chaque DSW annonce ses subnets directement connectés
 network 10.31.99.0 0.0.0.3 area 0
 network 10.31.10.0 0.0.0.31 area 0
 network 10.31.20.0 0.0.0.31 area 0
 network 10.31.30.0 0.0.0.127 area 0
 network 10.31.40.0 0.0.3.255 area 0
 ! On veut que OSPF ne fasse RIEN sur les SVIs utilisateurs - juste annoncer
 ! le subnet. Les hosts n'ont pas à recevoir d'hellos OSPF.
 passive-interface default
 no passive-interface vlan 900
 exit

end
write memory
```

Sur **DSW-02** (mêmes commandes, IP .2, router-id 2.2.2.2) :

```text
configure terminal

vlan 900
 name INTERNAL_TRANSIT
 exit

interface Port-channel 2
 switchport trunk allowed vlan 10,20,30,40,900
 exit

interface vlan 900
 description Transit L3 vers DSW-01
 ip address 10.31.99.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown
 exit

router ospf 1
 router-id 2.2.2.2
 network 10.31.99.0 0.0.0.3 area 0
 network 10.31.10.0 0.0.0.31 area 0
 network 10.31.20.0 0.0.0.31 area 0
 network 10.31.30.0 0.0.0.127 area 0
 network 10.31.40.0 0.0.3.255 area 0
 passive-interface default
 no passive-interface vlan 900
 exit

end
write memory
```

**Commandes nouvelles à connaître :**
- `ip ospf network point-to-point` sur un SVI : on dit à OSPF "cette interface est un lien point-à-point, ne fais pas d'élection DR/BDR dessus". C'est plus rapide à converger et ça évite des bugs sur les IOS virtualisés.
- `passive-interface default` + `no passive-interface vlan 900` : on désactive OSPF partout par défaut, et on le ré-active uniquement sur le SVI 900. Comme ça aucun host utilisateur ne reçoit d'hello OSPF (sécurité + bruit en moins).
- `router-id X.X.X.X` : identifiant unique du routeur OSPF, mis à la main pour éviter qu'OSPF en pioche un au hasard (et le change à chaque reboot).

### Vérification

```text
show ip ospf neighbor
```

Vous devez voir le voisin DSW-02 (depuis DSW-01) en état **FULL**. Si vous voyez INIT ou EXSTART qui reste bloqué, vérifiez les masques et le `ip ospf network point-to-point` des **deux** côtés.

```text
show ip route ospf
```

Vous devez voir les subnets de l'autre DSW marqués `O` (OSPF) — par exemple sur DSW-01, vous verrez le subnet VLAN 40 appris via OSPF en plus de son SVI connecté.

📸 **Capture demandée** :
- `show ip ospf neighbor` sur DSW-01 (avec voisin FULL)
- `show ip route ospf` sur DSW-01

### Tester le scénario de failover sans blackhole

```text
! Sur DSW-01
interface vlan 40
 shutdown
```

Avant : un PC VLAN 30 pinguait un PC VLAN 40 → retour blackholé sur DSW-01.
Après VLAN 900 + OSPF : DSW-01 a perdu son SVI VLAN 40 connecté, mais OSPF a appris la route via DSW-02 sur VLAN 900. Le retour est forwardé via Po2/VLAN 900 vers DSW-02 → ressort sur le SVI VLAN 40 de DSW-02 → arrive au PC. **Plus de blackhole.**

> **Question 28 :** dans `show ip route` sur DSW-01 après le `shutdown` du SVI VLAN 40, vous voyez la route vers 10.31.40.0/22 marquée `O` (OSPF) avec `via 10.31.99.2`. Quelle métrique OSPF (cost) cette route a-t-elle ? Pourquoi (formule du cost OSPF) ?

📸 **Capture demandée** : `show ip route 10.31.40.0` sur DSW-01 **après** shutdown du SVI VLAN 40, montrant la route apprise via OSPF.

N'oubliez pas de remonter le SVI à la fin :
```text
interface vlan 40
 no shutdown
```

---

## Récapitulatif — ce que vous avez livré

| # | Élément | Statut |
| --- | --- | --- |
| 1 | Hostnames + durcissement device de base (enable secret, console, banner) | ☐ |
| 2 | VLANs créés sur chaque switch | ☐ |
| 3 | Ports d'accès + portfast/bpduguard | ☐ |
| 4 | VLAN BLACKHOLE 999 + ports inutilisés shutdown | ☐ |
| 5 | Trunks ASW ↔ DSW (dot1q, native 99, allowed restreint, nonegotiate) | ☐ |
| 6 | LACP Port-channel 2 entre DSW-01 et DSW-02 | ☐ |
| 7 | `ip routing` + SVIs sur les deux DSW | ☐ |
| 8 | STP root primary/secondary aligné HSRP | ☐ |
| 9 | HSRP croisé actif/standby sur les 4 VLANs + auth MD5 + tracking | ☐ |
| 10 | DHCP relay sur SVIs utilisateurs | ☐ |
| 11 | Tests connectivité depuis PCs (DHCP, ping passerelle, inter-VLAN) | ☐ |
| 12 | Lien Internet (routed + HSRP outside + route par défaut) | ☐ |
| 13 | IP mgmt des ASW + default gateway | ☐ |
| 14 | SSH actif sur tous les switches + test depuis machine hôte | ☐ |
| 15 | Test failover HSRP (paquets perdus minimes) | ☐ |
| **Bonus 1** | LACP serveur Linux ↔ ASW-DC (Po3) | ☐ |
| **Bonus 2** | VLAN 900 + OSPF entre DSW + test anti-blackhole | ☐ |

### Captures d'écran attendues (récap)

1. `show running-config | section line` (étape 1)
2. `show vlan brief` sur DSW-01 et ASW-1 (étape 2)
3. `show interfaces status` sur ASW-1 (étape 3)
4. `show ip interface brief` sur ASW-1 (étape 3)
5. `show interfaces status | include 999` (étape 4)
6. `show interfaces trunk` sur DSW-01 (étape 5)
7. `show etherchannel summary` sur DSW-01 (étape 6)
8. `show ip route` avant et après `ip routing` (étape 7)
9. `show ip route connected` (étape 7)
10. `show spanning-tree vlan 30 brief` (étape 8)
11. `show standby brief` sur DSW-01 et DSW-02 (étape 9)
12. `show standby` détaillé avec track object (étape 9)
13. `show ip interface vlan 30` (helper-address) (étape 10)
14. `ip addr show` + ping passerelle depuis PCs (étape 11)
15. `show ip route` sur DSW-01 avec route par défaut (étape 12)
16. Pings depuis Adminsys vers les 3 ASW (étape 13)
17. `show ip ssh` (étape 14)
18. Session SSH depuis machine hôte (étape 14)
19. `show standby brief` après bascule HSRP (test failover)
20. `show etherchannel summary` ASW-DC pour Po3 (bonus 1)
21. `show ip ospf neighbor` + `show ip route ospf` (bonus 2)
22. `show ip route 10.31.40.0` après shutdown SVI (bonus 2)

---

## Pour la prochaine session (cet après-midi)

La session 2 portera sur les **ACL** sur cette même infrastructure. Vous appliquerez une matrice de flux Zero Trust entre les VLANs (qui peut parler à qui, sur quels ports). Le SSH que vous venez d'activer servira à vous reconnecter aux switches une fois que les ACL bloqueront les pings vers la mgmt — d'où l'importance de l'avoir activé maintenant. Pensez à bien sauvegarder vos configurations (`write memory`) avant de fermer EVE-NG.

Bon TP.
