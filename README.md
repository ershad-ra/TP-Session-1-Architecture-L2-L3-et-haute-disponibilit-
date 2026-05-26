# TP Session 1 : architecture L2/L3 et haute disponibilité

> **Format :** TP guidé sur 2h. Vous lisez, vous configurez, vous testez, vous prenez les captures demandées. Les questions intercalées ne sont pas des questions piège. Elles servent à vous pousser à réfléchir ou à chercher.
> 

> **Livrable :** ce document complété (vos réponses aux questions et vos captures), déposé sur GitHub Classroom à la fin de la séance.
> 

---

## Le scénario

L'école Ynov Toulouse renouvelle complètement son infrastructure réseau. Le campus est composé de deux bâtiments, avec une salle serveur (le **datacenter**) au premier étage du bâtiment A. Le câblage fibre (mono et multi 10G en paire) entre bâtiments et entre étages a été refait, et le nouveau matériel est posé.

Vous venez d'être embauché comme ingénieur sécurité réseau. Le DSI vous demande un **POC** illustrant la mise en place d'une infrastructure répondant aux standards actuels : haute disponibilité, segmentation, politique Zero Trust.

Le pare feu de bordure viendra dans un second temps. Cette première étape se concentre sur la sécurisation du LAN. Le budget étant limité, vous avez proposé une architecture **2 tiers (collapsed core)**, où le niveau core et le niveau distribution sont fusionnés. C'est adapté pour un campus de cette taille : une paire de distribution, un bloc datacenter, et un accès Internet.

Pour rester abordable, l'accès et le datacenter restent en **L2**, et la distribution est en **L3**.

> **Question 1 :** pourquoi avoir choisi des switches L3 pour la distribution plutôt que des switches L2 plus un routeur ? Donnez au moins deux raisons.
> 

**Votre réponse :**

```

```

> **Question 2 :** pourquoi avoir demandé que les switches de distribution et les switches d'accès du datacenter soient en paire ? Qu'est ce qu'on perd si on n'en met qu'un ?
> 

**Votre réponse :**

```

```

### Plan VLAN

Le plan complet prévoit aussi de la voix, des invités, du transit, etc. Pour le POC on garde l'essentiel :

| VLAN | Nom | Rôle |
| --- | --- | --- |
| 10 | MGMT | Gestion des switches (SSH, SNMP, syslog) |
| 20 | SERVERS | Serveurs du datacenter |
| 30 | COLLABORATEUR | Postes des collaborateurs |
| 40 | ETUDIANT | Postes étudiants |
| 99 | NATIVE | VLAN natif des trunks (jamais utilisé pour du trafic) |
| 999 | BLACKHOLE | VLAN poubelle pour les interfaces inutilisées |

### Plan d'adressage IP

Un plan trop large n'est pas une bonne idée pour la sécurité. Un plan trop petit n'est pas viable non plus. On fait du **VLSM** :

| VLAN | Subnet | Masque | Hosts utilisables |
| --- | --- | --- | --- |
| 10 MGMT | 10.31.10.0/27 | 255.255.255.224 | 30 |
| 20 SERVERS | 10.31.20.0/27 | 255.255.255.224 | 30 |
| 30 COLLAB | 10.31.30.0/25 | 255.255.255.128 | 126 |
| 40 ETUDIANT | 10.31.40.0/22 | 255.255.252.0 | 1022 |

La marge entre les blocs (10.31.**10**, .**20**, .**30**, .**40**) est volontairement non collée. C'est un choix de lisibilité pour les ingénieurs qui maintiendront le réseau.

> **Question 3 :** pourquoi un plan IP trop large est il un problème de sécurité ? Pensez à ce que voit un attaquant qui scanne, et à la taille du domaine de broadcast.
> 

**Votre réponse :**

```

```

### Haute disponibilité

Redondance à deux niveaux :

- **Liens** : agrégation via **LACP**, un protocole standardisé par l'IEEE (802.3ad).
- **Matériel et passerelle** : **HSRP**, propriétaire Cisco. Son équivalent standard est **VRRP** (RFC 5798). HSRP utilise une IP virtuelle (VIP) et une MAC virtuelle partagées entre les deux switches : un actif et un standby.

### Topologie

Voir le diagramme fourni dans EVE-NG. Récapitulatif des matériels et liens :

- **DSW-01 et DSW-02** : paire de distribution, switches L3.
    - Entre eux : 2 liens (e0/3 et e1/0) qui formeront un Port-channel LACP.
    - Vers Internet : chacun e1/1 vers le switch "Internet" (cloud NAT VMware).
- **ASW-1** (Bâtiment A, L2) : uplink e0/0 vers DSW-01, uplink e0/1 vers DSW-02, e0/2 Étudiant (VLAN 40), e0/3 Collab (VLAN 30), e1/0 vers le cloud `mgmt-sur-vlan10-pour-ssh`.
- **ASW-2** (Bâtiment B, L2) : uplink e0/0 vers DSW-01, uplink e0/1 vers DSW-02, e0/2 Collab (VLAN 30), e0/3 Adminsys (VLAN 10).
- **ASW-DC** (datacenter, L2) : uplink e0/0 vers DSW-01, uplink e0/1 vers DSW-02, e0/2 vers le serveur Linux (VLAN 20).

Les PCs TinyCore (Étudiant, Collab, Adminsys) démarrent en DHCP. Le serveur Linux dans le datacenter porte le serveur DHCP.

> **À propos du LACP entre les switches d'accès et la paire de distribution :** dans un environnement de production, on bundlerait les deux uplinks de chaque ASW vers la paire DSW en LACP, **à condition que la paire DSW soit en stack** (vue comme un seul switch logique). Or DSW-01 et DSW-02 dans notre lab sont deux équipements **physiquement et logiquement séparés** : leur control plane est indépendant, ils ne sont pas en stack. LACP impose que les deux extrémités d'un bundle soient sur la même unité (physique ou logique). On ne peut donc pas faire de LACP entre un ASW et la paire DSW dans ce lab. Les deux uplinks resteront en trunks séparés, et STP s'occupera d'éviter la boucle. **En production : on stack, et on LACP.**
> 

---

## Étape 0 : préparation du lab

Ouvrez votre instance EVE-NG, importez le fichier topologie fourni, et démarrez tous les nœuds. Vérifiez que chaque switch est accessible via le port console.

Pour vous connecter à un nouveau switch jamais configuré, la méthode numéro 1 reste **le port console**. SSH n'est pas encore configuré, IP n'est pas encore définie. Une fois le L2 en place et une IP de management posée, on basculera sur SSH (étape dédiée plus loin).

Niveaux de prompt à connaître :

- `>` : mode utilisateur. Accès limité, principalement des commandes de lecture.
- `#` : mode privilégié. Accès complet à la lecture et aux commandes opérationnelles.
- `(config)#` : mode configuration global, où on modifie la configuration.

Pour passer de l'un à l'autre :

```
enable
configure terminal
```

Pour sauvegarder, deux écritures équivalentes :

```
end
write memory
```

ou bien :

```
copy running-config startup-config
```

Vos commandes ne sont **pas** persistantes tant que vous n'avez pas sauvegardé. Un `reload` perd tout. Prenez le réflexe de faire un `write memory` après chaque étape validée.

---

## Étape 1 : hostnames et premier durcissement

Avant toute autre chose, donnez à chaque équipement son hostname, et appliquez quelques bases de sécurité device qui ne coûtent rien.

Sur **chaque switch** (adapter le hostname à l'équipement), passez en mode configuration :

```
enable
configure terminal
```

Définir le hostname :

```
hostname DSW-01
```

Définir le mot de passe enable. La commande `enable secret` stocke le mot de passe avec un algorithme de hachage, contrairement à l'ancienne `enable password` qui le stocke en clair.

```
enable secret MDP-Enable
```

Chiffrer les mots de passe restants présents dans la configuration (mots de passe console, VTY, etc.) :

```
service password-encryption
```

Définir une bannière légale. Dans plusieurs juridictions, l'affichage d'un message indiquant que l'accès est réservé conditionne la possibilité de poursuivre un intrus :

```
banner motd ^
*****************************************************
* ACCES RESERVE - Ynov Toulouse                     *
* Toute connexion non autorisee fera l objet de     *
* poursuites. Activite tracee et journalisee.       *
*****************************************************
^
```

Sécuriser le port console (mot de passe obligatoire, déconnexion automatique après 10 minutes d'inactivité) :

```
line console 0
 password MDP-Console
 login
 exec-timeout 10 0
 logging synchronous
 exit
```

`logging synchronous` empêche les messages de log d'interrompre votre saisie en cours. C'est un confort, pas une mesure de sécurité.

Fermer les lignes VTY pour l'instant. SSH sera ouvert plus loin dans une étape dédiée, une fois la crypto en place. `transport input none` interdit toutes les connexions entrantes sur les VTY (ni telnet, ni SSH, ni rlogin) :

```
line vty 0 4
 transport input none
 exec-timeout 10 0
 exit
```

Sauvegarder :

```
end
write memory
```

**Test** : déconnectez-vous d’un switch avec la commande exit et reconnecter vous à la console en donnant le mot de passe `MDP-Enable` et ensuite au niveau enable en donnant le mot de passe `MDP-Console`.

> **Question 4 :** pourquoi `enable secret` est préférable à `enable password` ? Cherchez la différence d'algorithme de stockage.
> 

**Votre réponse :**

```

```

**Capture demandée :** `show running-config | section line` sur un des switches.

**Votre capture :**

```

```
