TP Session 1 : architecture L2/L3 et haute disponibilité
Format : TP guidé sur 2h. Vous lisez, vous configurez (copier-coller direct des commandes), vous vérifiez, vous prenez les captures demandées. Les questions intercalées ne sont pas des questions piège : elles servent à vous pousser à réfléchir ou à chercher.
Livrable : ce document complété (vos réponses aux questions et vos captures), déposé sur GitHub Classroom à la fin de la séance.
Le scénario
L'école Ynov Toulouse renouvelle complètement son infrastructure réseau. Le campus est composé de deux bâtiments. Le câblage fibre (mono et multi 10G en paire) a été refait, et le nouveau matériel est posé.
Vous venez d'être embauché comme ingénieur sécurité réseau. Le DSI vous demande un POC illustrant la mise en place d'une infrastructure répondant aux standards actuels : haute disponibilité, segmentation, politique Zero Trust.
Le pare-feu de bordure viendra dans un second temps. Cette première étape se concentre sur la sécurisation du LAN. Le budget étant limité, vous avez proposé une architecture 2 tiers (collapsed core), où le niveau core et le niveau distribution sont fusionnés.
Pour rester abordable, l'accès reste en L2, et la distribution est en L3.
Question 1 : pourquoi avoir choisi des switches L3 pour la distribution plutôt que des switches L2 plus un routeur ? Donnez au moins deux raisons.
Votre réponse :
​
Plan VLAN
VLAN
Nom
Rôle
10
MGMT
Gestion des switches (SSH, SNMP, syslog)
20
SERVERS
Serveurs
30
COLLABORATEUR
Postes des collaborateurs
40
ETUDIANT
Postes étudiants
99
NATIVE
VLAN natif des trunks (jamais utilisé pour du trafic)
999
BLACKHOLE
VLAN poubelle pour les interfaces inutilisées
Plan d'adressage IP
Un plan trop large n'est pas une bonne idée pour la sécurité. Un plan trop petit n'est pas viable non plus. On fait du VLSM :
VLAN
Subnet
Masque
Hosts utilisables
10 MGMT
10.31.10.0/27
255.255.255.224
30
20 SERVERS
10.31.20.0/27
255.255.255.224
30
30 COLLAB
10.31.30.0/25
255.255.255.128
126
40 ETUDIANT
10.31.40.0/22
255.255.252.0
1022
La marge entre les blocs (10.31.10, .20, .30, .40) est volontairement non collée. C'est un choix de lisibilité pour les ingénieurs qui maintiendront le réseau.
Question 2 : pourquoi un plan IP trop large est-il un problème de sécurité ? Pensez à ce que voit un attaquant qui scanne, et à la taille du domaine de broadcast.
Votre réponse :
​
Haute disponibilité
Redondance à deux niveaux :
Liens : agrégation via LACP, un protocole standardisé par l'IEEE (802.3ad).
Matériel et passerelle : HSRP, propriétaire Cisco. Son équivalent standard est VRRP (RFC 5798). HSRP utilise une IP virtuelle (VIP) et une MAC virtuelle partagées entre les deux switches : un actif et un standby.
Topologie
Voir le diagramme fourni dans EVE-NG. Récapitulatif des matériels et liens :
DSW-01 et DSW-02 : paire de distribution, switches L3.
Entre eux : 2 liens (e0/3 et e1/0) qui formeront un Port-channel LACP (Po1).
Vers Internet : chacun e0/0 vers le switch "Internet" (cloud NAT VMware).
ASW-1 (Bâtiment A, L2) : uplink e0/0 vers DSW-01, uplink e0/1 vers DSW-02, e0/2 Étudiant (VLAN 40), e0/3 Collaborateur (VLAN 30).
ASW-2 (Bâtiment B, L2) : uplink e0/0 vers DSW-01, uplink e0/1 vers DSW-02, e0/2 Serveur Linux (VLAN 20), e0/3 Adminsys (VLAN 10).
Les PCs TinyCore (Étudiant, Collab, Adminsys) démarrent en DHCP. Le serveur Linux porte le serveur DHCP.
À propos du LACP entre les switches d'accès et la paire de distribution : dans un environnement de production, on bundlerait les deux uplinks de chaque ASW vers la paire DSW en LACP, à condition que la paire DSW soit en stack (vue comme un seul switch logique). Or DSW-01 et DSW-02 dans notre lab sont deux équipements physiquement et logiquement séparés. LACP impose que les deux extrémités d'un bundle soient sur la même unité (physique ou logique). On ne peut donc pas faire de LACP entre un ASW et la paire DSW dans ce lab. Les deux uplinks resteront en trunks séparés, et STP s'occupera d'éviter la boucle. En production : on stack, et on LACP.
Étape 0 : préparation du lab
Ouvrez votre instance EVE-NG, importez le fichier topologie fourni, et démarrez tous les nœuds. Vérifiez que chaque switch est accessible via le port console.
Pour vous connecter à un nouveau switch jamais configuré, la méthode numéro 1 reste le port console. SSH n'est pas encore configuré, IP n'est pas encore définie. Une fois le L2 en place et une IP de management posée, on basculera sur SSH (étape dédiée plus loin).
Niveaux de prompt à connaître :
> : mode utilisateur. Accès limité, principalement des commandes de lecture.
# : mode privilégié. Accès complet à la lecture et aux commandes opérationnelles.
(config)# : mode configuration global, où on modifie la configuration.
Pour passer de l'un à l'autre :
enable
configure terminal
​
Pour sauvegarder, deux écritures équivalentes :
write memory
​
ou bien :
copy running-config startup-config
​
Vos commandes ne sont pas persistantes tant que vous n'avez pas sauvegardé. Un reload perd tout. Prenez le réflexe de faire un write memory après chaque étape validée.
Étape 1 : hostnames et premier durcissement
Avant toute autre chose, donnez à chaque équipement son hostname et appliquez quelques bases de sécurité device qui ne coûtent rien.
Le bloc ci-dessous est à coller sur chaque switch (DSW-01, DSW-02, ASW-1, ASW-2). Pensez à remplacer DSW-01 par le hostname correct selon le switch sur lequel vous êtes.
enable
configure terminal

hostname DSW-01

enable secret mdp-enable

service password-encryption

banner motd ^
*****************************************************
* ACCES RESERVE - Ynov Toulouse                     *
* Toute connexion non autorisee fera l objet de     *
* poursuites. Activite tracee et journalisee.       *
*****************************************************
^

line console 0
 password mdp-console
 login
 exec-timeout 120 0
 logging synchronous
 exit

line vty 0 4
 transport input none
 exec-timeout 120 0
 exit

end
write memory
​
Quelques explications sur les commandes nouvelles :
enable secret stocke le mot de passe avec un algorithme de hachage robuste, contrairement à l'ancienne enable password qui le stocke en clair ou avec un obfuscateur trivialement réversible.
service password-encryption chiffre les mots de passe en clair qui restent dans la configuration (console, VTY).
banner motd affiche le bandeau au login. Dans plusieurs juridictions, ce bandeau conditionne la possibilité de poursuivre légalement un intrus.
exec-timeout 120 0 : déconnexion automatique après 120 minutes d'inactivité.
logging synchronous empêche les messages de log d'interrompre votre saisie en cours. C'est un confort de lecture.
transport input none sur les VTY : interdit toutes les connexions entrantes (telnet, SSH, rlogin) tant que SSH n'est pas configuré. SSH sera ouvert plus loin dans une étape dédiée.
Vérification
Déconnectez-vous d'un switch avec la commande exit puis revenez sur la console. Au prompt de connexion, donnez le mot de passe console (mdp-console) pour entrer en mode utilisateur. Puis tapez enable et donnez le mot de passe enable (mdp-enable) pour passer en mode privilégié.
Question 3 : pourquoi enable secret est-il préférable à enable password ? Cherchez la différence d'algorithme de stockage.
Votre réponse :
​
 Capture demandée : show running-config | section line sur un des switches.
Votre capture :
​
Étape 2 : création des VLANs
Sur chaque switch, créez les VLANs nécessaires à ce switch uniquement. Ne créez pas tous les VLANs sur tous les switches.
Sur DSW-01 et DSW-02
Les deux DSW portent tous les VLANs car ils font le routage inter-VLAN. Bloc identique sur les deux :
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
​
Sur ASW-1
ASW-1 porte les VLANs 10 (pour la gestion du switch), 30 (Collab), 40 (Étudiant), 99 (natif) et 999 (blackhole). Pas le 20.
enable
configure terminal

vlan 10
 name MGMT
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
​
Sur ASW-2
ASW-2 porte les VLANs 10 (Adminsys et gestion du switch), 20 (Serveur), 99 et 999.
enable
configure terminal

vlan 10
 name MGMT
 exit
vlan 20
 name SERVERS
 exit
vlan 99
 name NATIVE
 exit
vlan 999
 name BLACKHOLE
 exit

end
write memory
​
Vérification
Sur chaque switch, lancez :
show vlan brief
​
Vous devez voir les VLANs que vous venez de créer, avec leurs noms.
Question 4 : pourquoi est-ce une mauvaise pratique de créer tous les VLANs sur tous les switches "au cas où" ? Pensez à la surface d'attaque : un VLAN qui existe sur un switch est joignable si un attaquant arrive à brancher un câble sur le bon port.
Votre réponse :
​
 Capture demandée : show vlan brief sur DSW-01 et sur ASW-1.
Votre capture :
​
Étape 3 : ports d'accès
Un port d'accès est un port configuré pour appartenir à un seul VLAN. Les trames qui sortent de ce port vers le PC client sortent sans tag 802.1Q : le tag est retiré par le switch. C'est le mode utilisé pour tous les ports auxquels on branche un endpoint final (PC, serveur, imprimante, téléphone IP en simple branchement, etc.).
Deux commandes méritent une explication avant les blocs :
spanning-tree portfast fait passer le port directement en état forwarding, sans les phases listening et learning de STP. Sans ça, un PC peut attendre une trentaine de secondes avant d'avoir le réseau, et un client DHCP peut faire timeout.
spanning-tree bpduguard enable met le port en err-disabled immédiatement si un BPDU arrive dessus. Un poste utilisateur n'a aucune raison légitime d'envoyer des BPDU. Si ça arrive, c'est typiquement parce que quelqu'un a branché un mini-switch personnel, ou qu'une attaque STP est en cours.
Les deux ensemble (portfast + bpduguard) sont un standard absolu sur les ports d'accès.
Sur ASW-1
enable
configure terminal

interface ethernet 0/2
 description Poste Etudiant
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

interface ethernet 0/3
 description Poste Collaborateur
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

end
write memory
​
Sur ASW-2
enable
configure terminal

interface ethernet 0/2
 description Serveur Linux
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

interface ethernet 0/3
 description Poste Adminsys
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

end
write memory
​
Note sur no shutdown : les interfaces des switches sont allumées par défaut. On a donc supprimé no shutdown qui était inutile. On ne l'utilise que pour réactiver une interface qui a été administrativement mise en shutdown.
Vérification
Sur chaque ASW, vérifiez l'état des ports et leur VLAN d'access :
show interfaces status
show vlan brief
​
Question 5 : contre quelle attaque STP bpduguard protège-t-il ? Cherchez "STP root bridge attack".
Votre réponse :
​
 Capture demandée : show interfaces status sur ASW-1.
Votre capture :
​
Étape 4 : VLAN BLACKHOLE pour les ports inutilisés
Les ports non utilisés sur un switch sont une porte d'entrée pour quiconque a un accès physique. La best practice en production est de combiner deux mesures sur ces ports : les placer dans un VLAN poubelle (BLACKHOLE) et les administrativement éteindre. C'est de la défense en profondeur :
shutdown seul : si un admin réactive le port par erreur (mauvaise commande, restauration de config), le port redevient up dans son VLAN d'access. Si ce VLAN est le VLAN 1 par défaut, il y a souvent des services qui traînent dessus (CDP, parfois SVI de management sur vieux designs).
access vlan 999 seul : si le port est up et que quelqu'un branche un câble, la trame entre dans BLACKHOLE qui n'a aucun routage. Mais STP s'active sur le port, des BPDU sont émis, et un attaquant qui sniffe apprend des choses sur la topologie.
Les deux combinés : même si le port est réactivé par erreur, l'attaquant arrive nulle part.
Référence : CIS Cisco IOS Benchmark, recommandation "Disable unused interfaces AND assign to unused VLAN".
Tous les switches du lab ont 8 interfaces : e0/0 à e0/3 et e1/0 à e1/3.
Sur ASW-1
Ports utilisés : e0/0 (uplink DSW-01), e0/1 (uplink DSW-02), e0/2 (Étudiant), e0/3 (Collab). Ports inutilisés : e1/0 à e1/3.
enable
configure terminal

interface range ethernet 1/0 - 3
 description UNUSED - BLACKHOLE
 switchport mode access
 switchport access vlan 999
 shutdown
 exit

end
write memory
​
Sur ASW-2
Ports utilisés : e0/0 (uplink DSW-01), e0/1 (uplink DSW-02), e0/2 (Serveur Linux), e0/3 (Adminsys). Ports inutilisés : e1/0 à e1/3.
enable
configure terminal

interface range ethernet 1/0 - 3
 description UNUSED - BLACKHOLE
 switchport mode access
 switchport access vlan 999
 shutdown
 exit

end
write memory
​
Sur DSW-01 et DSW-02
Ports utilisés sur chaque DSW : e0/0 (Internet), e0/1 (ASW-1 ou ASW-2 selon le DSW), e0/2 (ASW-2 ou ASW-1 selon le DSW), e0/3 (vers l'autre DSW, Po1), e1/0 (vers l'autre DSW, Po1). Ports inutilisés : e1/1 à e1/3.
enable
configure terminal

interface range ethernet 1/1 - 3
 description UNUSED - BLACKHOLE
 switchport mode access
 switchport access vlan 999
 shutdown
 exit

end
write memory
​
Vérification
show interfaces status
​
Les interfaces dans BLACKHOLE doivent apparaître en disabled (administratively down) dans le VLAN 999.
Question 6 : un attaquant branche un câble sur un port en shutdown et dans BLACKHOLE. Que se passe-t-il pour lui ?
Votre réponse :
​
 Capture demandée : show interfaces status | include 999 sur un des switches.
Votre capture :
​
Étape 5 : trunks entre ASW et DSW
Un port trunk transporte plusieurs VLANs sur un même lien. Chaque trame conserve son tag 802.1Q, sauf celles du VLAN natif qui circulent sans tag. C'est le mode utilisé entre switches.
Deux encapsulations existent historiquement : dot1q (standard IEEE 802.1Q) et ISL (propriétaire Cisco, abandonné). Sur les IOS récents seul dot1q est disponible, mais sur certains modèles plus anciens il faut le préciser explicitement, donc on prend l'habitude de le faire.
Comme expliqué dans la section topologie, on ne fait pas de LACP entre un ASW et la paire DSW dans ce lab (limite de stack). Les deux uplinks de chaque ASW restent donc en trunks séparés.
Le VLAN natif est le VLAN dont les trames passent sans tag sur le trunk. Par défaut c'est le VLAN 1. On le change pour un VLAN inutilisé (VLAN 99 ici) pour deux raisons : éviter qu'un attaquant exploite le VLAN 1 par défaut pour des attaques de type VLAN hopping, et garder VLAN 1 propre.
Le VLAN natif doit être identique des deux côtés du trunk. Un mismatch (par exemple DSW-01 en native 99 et ASW-1 oublié en native 1) génère des messages d'erreur dans show interfaces trunk.
Sur ASW-1
enable
configure terminal

interface range ethernet 0/0 - 1
 description Uplink vers paire DSW
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,40
 exit

end
write memory
​
Sur ASW-2
enable
configure terminal

interface range ethernet 0/0 - 1
 description Uplink vers paire DSW
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 exit

end
write memory
​
Sur DSW-01 et DSW-02
enable
configure terminal

interface ethernet 0/1
 description Vers ASW-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,40
 exit

interface ethernet 0/2
 description Vers ASW-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 exit

end
write memory
​
Vérification
Sur DSW-01 et DSW-02 :
show interfaces trunk
​
Vous devez voir les deux trunks (vers ASW-1 et ASW-2) avec le bon native VLAN (99) et la bonne liste de VLANs autorisés.
Note sur switchport nonegotiate : cette commande désactive DTP (Dynamic Trunking Protocol), qui sert à négocier automatiquement le mode trunk avec le voisin. DTP est un vecteur d'attaque connu (switch spoofing). On le désactivera en session 3, dédiée au durcissement L2. Pour ce TP, on laisse DTP actif : les trunks fonctionnent sans problème car switchport mode trunk fige déjà le mode du port en dur.
Question 7 : pourquoi limiter les VLANs autorisés sur chaque trunk au strict nécessaire ? Quelle attaque facilite-t-on en permettant tous les VLANs partout ?
Votre réponse :
​
 Capture demandée : show interfaces trunk sur DSW-01.
Votre capture :
​
Étape 6 : agrégation LACP entre les DSW (Port-channel 1)
LACP, le standard IEEE 802.3ad, permet d'agréger plusieurs liens physiques en un canal logique unique. Les deux extrémités d'un bundle doivent être sur la même unité (un seul switch physique, ou plusieurs switches en stack vus comme un seul logiquement).
Dans notre lab, le seul endroit où LACP est applicable est entre DSW-01 et DSW-02 : les deux liens (e0/3 et e1/0) ont leurs deux extrémités sur la même paire d'équipements.
À propos du mode active : LACP propose plusieurs modes. active veut dire que ce côté envoie activement des paquets LACP pour former le bundle. C'est le mode recommandé des deux côtés. Le mode passive (ce côté répond seulement si l'autre demande) fonctionne aussi mais demande que l'autre côté soit en active. Si les deux côtés sont en passive, le bundle ne se forme pas. La règle simple : active des deux côtés, toujours.
Sur DSW-01
enable
configure terminal

interface range ethernet 0/3, ethernet 1/0
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 99
 description Vers DSW-02 (membre Po1)
 channel-group 1 mode active
 exit

interface Port-channel 1
 description Lien inter-DSW (vers DSW-02)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40
 exit

end
write memory
​
Sur DSW-02
enable
configure terminal

interface range ethernet 0/3, ethernet 1/0
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 99
 description Vers DSW-01 (membre Po1)
 channel-group 1 mode active
 exit

interface Port-channel 1
 description Lien inter-DSW (vers DSW-01)
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40
 exit

end
write memory
​
Vérification
Sur DSW-01 et DSW-02 :
show etherchannel summary
​
Vous devez voir Po1(SU) où S veut dire L2 et U veut dire in-use. Les deux membres doivent être en (P) (bundled in port-channel).
Question 8 : dans la sortie de show etherchannel summary, que veulent dire les flags (P), (D) et (s) à côté d'une interface ?
Votre réponse :
​
 Capture demandée : show etherchannel summary sur DSW-01.
Votre capture :
​
Étape 7 : routage L3 et SVIs sur les DSW
C'est ici qu'on quitte le pur L2. Le routage inter-VLAN se fait sur les switches de distribution via les SVIs (Switch Virtual Interfaces) : une interface virtuelle par VLAN, avec une IP, qui sert de passerelle pour les hosts de ce VLAN.
ip routing rend un switch L3 capable de router entre ses SVIs. Sans elle, un switch L3 se comporte comme un switch L2.
no ip cef désactive Cisco Express Forwarding, une fonctionnalité matérielle. Sur de l'IOL ou IOSv en machine virtuelle dans EVE-NG, le hardware FIB n'existe pas et CEF casse le routage. Sur du vrai matériel en production, on garde CEF activé.
Sur DSW-01
enable
configure terminal

ip routing
no ip cef

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
​
Sur DSW-02
enable
configure terminal

ip routing
no ip cef

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
​
Vérification
Sur DSW-01 :
show ip route connected
​
Vous devez voir les 4 subnets en C (connected).
Question 9 : chaque DSW a maintenant sa propre IP de SVI dans chaque VLAN. Mettre .1 ou .2 comme passerelle sur les PCs ne donne aucune redondance. Quelle est la solution ? (Indice : ça commence par H et c'est dans le titre de l'étape 9.)
Votre réponse :
​
 Capture demandée : show ip route connected sur DSW-01.
Votre capture :
​
Étape 8 : STP root bridges
Avant HSRP, on fixe proprement les root bridges STP. Sans intervention, STP élit le switch avec le BID (Bridge ID) le plus bas. C'est souvent un switch d'accès parce qu'il a une MAC plus ancienne, ce qui donne des chemins absurdes.
La convention de design : sur chaque VLAN, le DSW qui sera HSRP active est aussi le root primary STP. C'est cohérent : le trafic L2 et le trafic routé prennent le même chemin en mode normal, ce qui évite des chemins en zigzag.
root primary ajuste automatiquement la priorité STP pour que ce switch devienne root. root secondary ajuste la priorité pour qu'il devienne root si le primary tombe.
Sur DSW-01
enable
configure terminal

spanning-tree vlan 10,30 root primary
spanning-tree vlan 20,40 root secondary

end
write memory
​
Sur DSW-02
enable
configure terminal

spanning-tree vlan 20,40 root primary
spanning-tree vlan 10,30 root secondary

end
write memory
​
Vérification
Sur DSW-01 :
show spanning-tree vlan 30 summary
​
Vous devez voir que ce switch est root pour VLAN 30. La ligne "Root bridge for VLAN0030 is this bridge." doit apparaître. Comparer le avec le résultat de la même commande sur DSW-02.
 Capture demandée : show spanning-tree vlan 30 summary sur DSW-01.
Votre capture :
​
Étape 9 : HSRP, passerelle redondante
HSRP (Hot Standby Router Protocol) permet à deux switches L3 de partager une IP virtuelle (VIP) et une MAC virtuelle. Les hosts utilisent la VIP comme passerelle. À un instant T, un seul des deux switches est active (forward) et l'autre est standby (silencieux, prêt à prendre le relais). Si l'active tombe, le standby prend la main en une à trois secondes.
Pourquoi un mode active/standby croisé entre les deux DSW ? Si DSW-01 est active pour tous les VLANs, DSW-02 ne fait rien en mode normal. Son CPU et sa bande passante sont gâchés. En croisant les rôles (DSW-01 active pour VLAN 10 et 30, DSW-02 active pour VLAN 20 et 40), on load-balance : chaque DSW porte la moitié du trafic en temps normal, et l'autre prend le relais complet en cas de panne.
Plan d'adressage HSRP
Convention : .1 est l'IP du DSW-01, .2 est l'IP du DSW-02, .3 est la VIP (la passerelle vue par les hosts).
VLAN
Subnet
DSW-01
DSW-02
VIP (passerelle hosts)
Active
10 MGMT
10.31.10.0/27
.1
.2
10.31.10.3
DSW-01
20 SERVERS
10.31.20.0/27
.1
.2
10.31.20.3
DSW-02
30 COLLAB
10.31.30.0/25
.1
.2
10.31.30.3
DSW-01
40 ETUDIANT
10.31.40.0/22
.1
.2
10.31.40.3
DSW-02
Explication des commandes nouvelles :
standby version 2 : HSRPv2 supporte plus de groupes, une MAC virtuelle différente, IPv6. À utiliser systématiquement.
standby X ip Y : VIP du groupe HSRP numéro X. Convention : le numéro de groupe est égal au numéro de VLAN, pour la lisibilité.
standby X priority N : priorité par défaut 100, la plus haute gagne.
standby X preempt : sans cette option, si l'active tombe puis revient, il ne reprend pas son rôle. Avec preempt, dès qu'il revient et que sa priorité est supérieure, il récupère le rôle active.
standby X authentication md5 key-string ... : empêche un attaquant branché sur le segment d'envoyer un hello HSRP avec priorité 255 et de détourner le rôle active (MITM passerelle).
Sur DSW-01
enable
configure terminal

interface vlan 10
 standby version 2
 standby 10 ip 10.31.10.3
 standby 10 priority 110
 standby 10 preempt
 standby 10 authentication md5 key-string Ynov-HSRP-2026!
 exit

interface vlan 20
 standby version 2
 standby 20 ip 10.31.20.3
 standby 20 priority 100
 standby 20 preempt
 standby 20 authentication md5 key-string Ynov-HSRP-2026!
 exit

interface vlan 30
 standby version 2
 standby 30 ip 10.31.30.3
 standby 30 priority 110
 standby 30 preempt
 standby 30 authentication md5 key-string Ynov-HSRP-2026!
 exit

interface vlan 40
 standby version 2
 standby 40 ip 10.31.40.3
 standby 40 priority 100
 standby 40 preempt
 standby 40 authentication md5 key-string Ynov-HSRP-2026!
 exit

end
write memory
​
Sur DSW-02 (priorités inversées)
enable
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
​
Interface tracking
Tel quel, HSRP ne sait pas si l'uplink Internet du DSW est tombé. DSW-01 reste active pour VLAN 30 même si son lien Internet est mort, donc les paquets des collaborateurs partent dans le vide.
On corrige ça en suivant l'état de l'interface uplink Internet (e0/0 sur la nouvelle topologie). Si elle tombe, on décrémente la priorité HSRP de 20. La priorité chute en dessous de celle du voisin, preempt déclenche, le voisin devient active, et le trafic ressort par le bon chemin.
Sur DSW-01
enable
configure terminal

track 1 interface ethernet 0/0 line-protocol

interface vlan 10
 standby 10 track 1 decrement 20
 exit

interface vlan 30
 standby 30 track 1 decrement 20
 exit

end
write memory
​
Sur DSW-02
enable
configure terminal

track 1 interface ethernet 0/0 line-protocol

interface vlan 20
 standby 20 track 1 decrement 20
 exit

interface vlan 40
 standby 40 track 1 decrement 20
 exit

end
write memory
​
Vérification
Sur les deux DSW :
show standby brief
​
Vous devez voir les rôles Active/Standby cohérents avec le plan.
Sur DSW-01, version détaillée pour vérifier le tracking :
show standby vlan 30
​
Vous devez voir la ligne Track object 1 state Up decrement 20.
Question 10 : pourquoi décrémenter de 20 et pas de 5 ? Calculez ce que devient la priorité de DSW-01 sur VLAN 30 après décrément, et comparez à celle de DSW-02 sur le même VLAN.
Votre réponse :
​
 Capture demandée : show standby brief sur DSW-01 et DSW-02.
Votre capture :
​
Étape 10 : lien Internet et HSRP côté outside
Le switch "Internet" sur le diagramme est un unmanaged switch (présenté sous icône routeur) qui ponte les interfaces e0/0 des deux DSW vers le réseau NAT VMware Workstation Pro (typiquement 192.168.29.0/24).
Vérification préalable : sur VMware Workstation, ouvrez Virtual Network Editor et notez la plage exacte du réseau NAT. Notez aussi l'adresse de la passerelle (souvent .2). Les premières adresses de la plage sont parfois déjà prises par l'hôte ou par d'autres services VMware. Ajustez les IPs ci-dessous si votre plage diffère.
Plan d'adressage côté Internet (en supposant 192.168.29.0/24, GW VMware = .2) :
Rôle
IP
Gateway VMware (NAT)
192.168.29.2
DSW-01 e0/0
192.168.29.101
DSW-02 e0/0
192.168.29.102
VIP HSRP outside
192.168.29.100
Sur le lien Internet, on n'a pas de VLAN ni de SVI : c'est un point à point vers le NAT. On configure donc l'interface en interface routée avec no switchport. Cela bascule l'interface du mode L2 (par défaut sur un switch) au mode L3, où elle peut porter une IP directement, exactement comme une interface de routeur.
Sur DSW-01
enable
configure terminal

interface ethernet 0/0
 description Uplink Internet (vers cloud NAT)
 no switchport
 ip address 192.168.29.101 255.255.255.0
 standby version 2
 standby 100 ip 192.168.29.100
 standby 100 priority 110
 standby 100 preempt
 standby 100 authentication md5 key-string Ynov-HSRP-2026!
 standby use-bia
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 192.168.29.2

end
write memory
​
Sur DSW-02
enable
configure terminal

interface ethernet 0/0
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
​
ip route 0.0.0.0 0.0.0.0 192.168.29.2 est la route par défaut sur chacun des DSW. Tout paquet pour une destination inconnue (typiquement Internet) part vers la passerelle VMware.
Vérification
Sur chaque DSW :
show ip route
​
ping 192.168.29.2
​
Le ping doit passer.
Question 11 : quelle est la différence pratique entre une interface vlan 10 (SVI) et une interface ethernet 0/0 no switchport ip address ... (interface routée) ? Donnez un cas d'usage où chacune est préférable.
Votre réponse :
​
 Capture demandée : show ip route sur DSW-01.
Votre capture :
​
Étape 11 : DHCP relay sur les SVIs utilisateurs
Le serveur DHCP est dans le VLAN 20 (sur 10.31.20.10). Les PCs en VLAN 30 et 40 sont dans des broadcast domains différents. Leurs requêtes DHCP DISCOVER sont des broadcasts qui ne franchissent pas les frontières L3 sans aide.
La solution est la commande ip helper-address sur le SVI du VLAN client. Le DSW reçoit le DISCOVER en broadcast, le réencapsule en unicast vers l'IP du serveur DHCP, et relaie la réponse en retour vers le client.
Sur DSW-01
enable
configure terminal

interface vlan 30
 ip helper-address 10.31.20.20
 exit

interface vlan 40
 ip helper-address 10.31.20.20
 exit

end
write memory
​
Sur DSW-02
enable
configure terminal

interface vlan 30
 ip helper-address 10.31.20.20
 exit

interface vlan 40
 ip helper-address 10.31.20.20
 exit

end
write memory
​
Vérification
Sur DSW-01 :
show ip interface vlan 30 | include Helper
​
Vous devez voir la ligne "Helper address is 10.31.20.10".
Question 12 : pourquoi le DHCP relay doit-il être configuré sur les deux DSW, et pas seulement sur l'active HSRP ?
Votre réponse :
​
 Capture demandée : show ip interface vlan 30 | include Helper sur DSW-01.
Votre capture :
​
Étape 12 : configuration des PCs et tests
Sur les PCs TinyCore, les VLAN 30 et 40 démarrent en DHCP. Adminsys (VLAN 10) est en statique.
Adminsys (TinyCore en VLAN 10)
sudo ifconfig eth0 10.31.10.20 netmask 255.255.255.224 up
sudo route add default gw 10.31.10.3
​
Collab (VLAN 30) et Étudiant (VLAN 40)
Attention ! Il faut activer le serveur DHCP ! Le service serveur DHCP est déjà installé sur le serveur. Il suffit de configurer le serveur sur une adresse IP statique et démarrer le service :
# 1. Set the static IP and subnet mask
ifconfig eth0 10.31.20.20 netmask 255.255.255.224 up

# 2. Add the default gateway
route add default gw 10.31.20.3

# 3. Start the DHCP server manually
rc-service dnsmasq start
​
Laissez le client DHCP faire son travail sur les PCs :
sudo udhcpc -i eth0
​
Vérification
Depuis chaque PC :
ip addr show eth0
ip route
ping -c 3 <VIP du VLAN du PC>     # par exemple 10.31.30.3 depuis Collab
ping -c 3 10.31.20.10             # serveur Linux
​
Depuis Collab, pinguez Étudiant et vice versa pour valider l'inter-VLAN :
ping -c 3 10.31.40.50    # depuis Collab vers Étudiant (adapter à l'IP réelle)
​
Question 13 : depuis Collab (VLAN 30), pinguez Étudiant (VLAN 40). Ça doit fonctionner. Sur quel DSW le paquet routé est-il passé à l'aller ? Et au retour ? (Indice : regardez le plan HSRP active.)
Votre réponse :
​
 Capture demandée : ip addr show eth0 et un ping réussi vers la passerelle, depuis un PC Collab et un PC Étudiant.
Votre capture :
​
Étape 13 : IP de management des switches L2
Les switches d'accès (ASW-1, ASW-2) sont en L2 : ils n'ont pas de routage. On leur donne une seule IP dans le VLAN 10 (MGMT) pour SSH/SNMP/syslog, et une default gateway pointant sur la VIP HSRP.
Convention : adresses finissant en .10X pour les ASW.
Switch
IP mgmt
ASW-1
10.31.10.101/27
ASW-2
10.31.10.102/27
Sur un switch L2, on n'utilise pas ip route 0.0.0.0 0.0.0.0 mais bien ip default-gateway. La différence : ip route suppose que le routage IP est activé (ip routing), ce qui n'est pas le cas sur un switch L2. ip default-gateway est l'équivalent "L2-only" et sert au switch pour ses propres paquets (réponses SSH, syslog, NTP, etc.).
Sur ASW-1
enable
configure terminal

interface vlan 10
 description Mgmt SVI
 ip address 10.31.10.11 255.255.255.224
 no shutdown
 exit

ip default-gateway 10.31.10.3

end
write memory
​
Sur ASW-2
enable
configure terminal

interface vlan 10
 description Mgmt SVI
 ip address 10.31.10.12 255.255.255.224
 no shutdown
 exit

ip default-gateway 10.31.10.3

end
write memory
​
Vérification
Depuis le PC Adminsys (10.31.10.20), pingez les deux ASW :
ping -c 2 10.31.10.11    # ASW-1
ping -c 2 10.31.10.12    # ASW-2
​
Question 14 : votre instinct dit "il suffit d'ajouter ip routing sur les ASW pour pouvoir utiliser ip route à la place". Pourquoi ce n'est pas une bonne idée sur un switch d'accès ?
Votre réponse :
​
 Capture demandée : pings réussis depuis Adminsys vers les 2 ASW.
Votre capture :
​
Étape 14 : activation du SSH sur les switches
La session 2 (ACL) et la session 3 (RADIUS) ont besoin que SSH soit actif sur tous les switches. On configure le SSH ici en authentification locale. La centralisation via RADIUS sera ajoutée en session 3, et les ACL de filtrage d'accès en session 2.
Le bloc ci-dessous est à coller sur chaque switch (DSW-01, DSW-02, ASW-1, ASW-2).
enable
configure terminal

ip domain-name ynov-toulouse.lab

crypto key generate rsa modulus 2048

ip ssh version 2

username admin privilege 15 secret mdp-admin-local

line vty 0 4
 login local
 transport input ssh
 exec-timeout 120 0
 exit

end
write memory
​
Explication des commandes :
ip domain-name : nécessaire pour générer la clé RSA. Le FQDN du switch est construit à partir du hostname et de ce domain-name.
crypto key generate rsa modulus 2048 : génère la paire de clés RSA. La modulus 2048 bits est le minimum acceptable aujourd'hui. 1024 et 512 sont à proscrire.
ip ssh version 2 : impose SSHv2. SSHv1 a des failles connues et ne doit jamais être autorisé.
username admin privilege 15 secret ... : crée un compte local avec niveau de privilège maximal (15) et mot de passe haché.
login local : sur les VTY, on authentifie via la base locale.
transport input ssh : autorise SSH uniquement sur les VTY. Telnet (qui passe en clair) reste interdit.
Vérification (test SSH depuis Adminsys)
Depuis le PC Adminsys :
ssh admin@10.31.10.11
​
Donnez le mot de passe mdp-admin-local. Vous devez arriver dans le prompt du switch en mode utilisateur.
Question 15 : pourquoi transport input ssh (uniquement SSH) plutôt que transport input ssh telnet (les deux) ? Quelle erreur classique fait-on en laissant telnet activé "au cas où" ?
Votre réponse :
​
 Capture demandée : session SSH réussie depuis Adminsys vers un des switches.
Votre capture :
​
Étape 15 : NAT côté Internet pour les PCs
À ce stade, les PCs ont une IP, leur passerelle pointe sur la VIP HSRP, et les DSW ont une route par défaut vers la passerelle VMware (192.168.29.2). Mais si un PC essaie de pinger 8.8.8.8, ça ne marche pas.
Pourquoi ? Parce que la requête sort avec une source 10.31.30.X (Collab par exemple). Cette IP n'est pas routable sur Internet. Quand la réponse de 8.8.8.8 retournerait, elle serait destinée à 10.31.30.X, qui n'existe nulle part sur Internet.
La solution est le NAT (PAT) sur le DSW : avant d'envoyer le paquet vers la passerelle VMware, le DSW remplace l'IP source privée par sa propre IP publique (ici, son IP outside dans le réseau VMware NAT). VMware fait ensuite un deuxième NAT vers l'extérieur. Les réponses suivent le chemin inverse.
Les concepts à retenir :
ip nat inside sur les interfaces côté LAN, ip nat outside sur l'interface côté Internet. Le routeur sait ainsi dans quel sens traduire.
overload active le mode PAT (Port Address Translation) : plusieurs sources internes partagent une seule IP publique, distinguées par le port TCP/UDP source.
L'ACL standard définit quelles sources internes ont le droit d'être natées. C'est aussi un moyen de filtrer : si un subnet n'est pas dans l'ACL, il ne sort pas vers Internet.
Sur DSW-01
enable
configure terminal

ip access-list standard NAT-INTERNAL
 permit 10.31.10.0 0.0.0.31
 permit 10.31.20.0 0.0.0.31
 permit 10.31.30.0 0.0.0.127
 permit 10.31.40.0 0.0.3.255
 exit

interface ethernet 0/0
 ip nat outside
 exit

interface vlan 10
 ip nat inside
 exit
interface vlan 20
 ip nat inside
 exit
interface vlan 30
 ip nat inside
 exit
interface vlan 40
 ip nat inside
 exit

ip nat inside source list NAT-INTERNAL interface ethernet 0/0 overload

end
write memory
​
Sur DSW-02
enable
configure terminal

ip access-list standard NAT-INTERNAL
 permit 10.31.10.0 0.0.0.31
 permit 10.31.20.0 0.0.0.31
 permit 10.31.30.0 0.0.0.127
 permit 10.31.40.0 0.0.3.255
 exit

interface ethernet 0/0
 ip nat outside
 exit

interface vlan 10
 ip nat inside
 exit
interface vlan 20
 ip nat inside
 exit
interface vlan 30
 ip nat inside
 exit
interface vlan 40
 ip nat inside
 exit

ip nat inside source list NAT-INTERNAL interface ethernet 0/0 overload

end
write memory
​
Vérification
Depuis un PC Collab :
# gateway VMware
ping 192.168.29.2

# Internet (si NAT VMware route bien sortant)
ping 8.8.8.8
ping google.com
​
Pendant que le ping tourne, sur DSW-01 (ou DSW-02 selon qui est HSRP active pour VLAN 30) :
show ip nat translations
​
Vous devez voir des entrées avec inside global = 192.168.29.101 (ou .102), inside local = 10.31.30.X.
Question 16 : votre PC Collab a l'IP 10.31.30.50. Quand il pingue 8.8.8.8 et que vous regardez la trame en sortie sur e0/0 de DSW-01, quelle est l'IP source ? Et l'IP destination ?
Votre réponse :
​
 Capture demandée : show ip nat translations sur DSW-01 pendant qu'un PC pingue 8.8.8.8.
Votre capture :
​
Étape 16 : test de failover HSRP
Avant de simuler la panne, vérifiez quel équipement possède actuellement le rôle "Active" pour le VLAN 30. Sur DSW-01 et DSW-02, exécutez :
show standby brief
​
Depuis un PC Collab (VLAN 30), lancez un ping continu vers Internet pour observer l'impact de la bascule en temps réel :
ping google.com
​
Sur le switch que vous avez identifié comme Active à l'étape 1, simulez la perte de la connexion Internet en éteignant l'interface uplink (e0/0) :
enable
configure terminal
interface ethernet 0/0
 shutdown
 end
​
Vérification
Sur DSW-02 après bascule :
show standby brief
​
Note : N’oubliez pas d’activer l’interface e0/0 à nouveau avec la commande no shutdown.
 Capture demandée : show standby brief après bascule, montrant DSW-02 en Active pour VLAN 30.
Votre capture :
​
Bonus : VLAN 900 INTERNAL_TRANSIT et OSPF entre les DSW
C'est la décision d'architecture la plus subtile de tout ce TP. Lisez bien le contexte avant de configurer.
Le problème : routage asymétrique pendant un failover
Considérez la situation normale sur votre lab tel que configuré :
DSW-01 est HSRP active pour VLAN 30.
DSW-02 est HSRP active pour VLAN 40.
Un PC en VLAN 40 pingue un PC en VLAN 30 :
Aller : requête routée par DSW-02 (passerelle du source VLAN 40). DSW-02 a aussi un SVI dans VLAN 30 (même en standby), donc il forward dans VLAN 30 directement.
Retour : la réponse part du PC VLAN 30 vers sa passerelle, qui est la VIP 10.31.30.3, soit DSW-01 (active VLAN 30). DSW-01 reçoit la réponse.
Aller par DSW-02, retour par DSW-01 : c'est du routage asymétrique.
Tant que les deux DSW ont une route valide vers les deux subnets, ça passe. Mais pendant un test de failover, si on shutdown le SVI VLAN 40 sur DSW-01, le subnet 10.31.40.0/22 disparaît de la table de routage de DSW-01. Le retour pour le VLAN 40 arrive sur DSW-01, pas de route, drop silencieux, blackhole.
La solution : VLAN 900 comme transit L3 entre les DSW et OSPF
L'idée est de créer un VLAN dédié entre les deux DSW, leur donner à chacun une IP routée dessus, et faire tourner un protocole de routage (OSPF) entre eux. Chaque DSW apprend automatiquement les subnets que l'autre connaît, et peut router le trafic asymétrique via Po1 vers l'autre DSW.
Pourquoi ne pas simplement passer Po1 en no switchport (routed) ? Parce que Po1 doit rester un trunk L2 pour transporter les hellos HSRP de tous les VLANs, ainsi que d'autres VLANs L2 stretchés qu'on ajoutera plus tard (par exemple un VLAN pour la réplication de l'ASA en failover, vu en session edge/firewall). Donc Po1 reste L2, et VLAN 900 piggyback dessus comme transit L3 dédié.
Plan d'adressage VLAN 900
Un /30, transit point à point :
DSW-01 : 10.31.99.1/30
DSW-02 : 10.31.99.2/30
Commandes nouvelles à connaître :
ip ospf network point-to-point sur un SVI : on indique à OSPF que cette interface est un lien point à point. On évite ainsi l'élection DR/BDR. C'est plus rapide à converger et ça contourne des bugs connus sur les IOS virtualisés.
passive-interface default puis no passive-interface vlan 900 : on désactive OSPF partout par défaut, et on le réactive uniquement sur le SVI 900. Aucun host utilisateur ne reçoit d'hello OSPF (sécurité, et bruit en moins).
router-id X.X.X.X : identifiant unique du routeur OSPF, fixé à la main pour éviter qu'OSPF en pioche un au hasard (et le change à chaque reboot).
Sur DSW-01
enable
configure terminal

vlan 900
 name INTERNAL_TRANSIT
 exit

interface Port-channel 1
 switchport trunk allowed vlan 10,20,30,40,900
 exit

interface vlan 900
 description Transit L3 vers DSW-02
 ip address 10.31.99.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown
 exit

router ospf 1
 router-id 1.1.1.1
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
​
Sur DSW-02
enable
configure terminal

vlan 900
 name INTERNAL_TRANSIT
 exit

interface Port-channel 1
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
​
Vérification
Sur DSW-01 :
show ip ospf neighbor
​
Vous devez voir le voisin DSW-02 en état FULL. Si vous voyez INIT ou EXSTART qui reste bloqué, vérifiez les masques et le ip ospf network point-to-point des deux côtés.
show ip route ospf
​
Vous devez voir les subnets de l'autre DSW marqués O (OSPF).
Tester le scénario de failover sans blackhole
Sur DSW-01 :
enable
configure terminal
interface vlan 40
 shutdown
 end
​
Avant la mise en place de VLAN 900 et OSPF, un PC VLAN 30 pinguait un PC VLAN 40 et le retour était blackholé sur DSW-01. Après VLAN 900 et OSPF, DSW-01 a perdu son SVI VLAN 40 connecté, mais OSPF a appris la route vers VLAN 40 via DSW-02 sur VLAN 900. Le retour est forwardé via Po1 et VLAN 900 vers DSW-02, qui le délivre sur son propre SVI VLAN 40 vers le PC. Plus de blackhole.
show ip route 10.31.40.0
​
La route doit apparaître via OSPF (préfixe O).
N'oubliez pas de remonter le SVI à la fin :
enable
configure terminal
interface vlan 40
 no shutdown
end
write memory
​
 Capture demandée : show ip ospf neighbor et show ip route 10.31.40.0 après shutdown du SVI VLAN 40.
Votre capture :
​
Récapitulatif des livrables
#
Élément
Statut
1
Hostnames et durcissement device
☐
2
VLANs créés sur chaque switch (en limitant à ce qui est nécessaire)
☐
3
Ports d'accès avec portfast et bpduguard
☐
4
VLAN BLACKHOLE 999 et ports inutilisés shutdown
☐
5
Trunks ASW vers DSW (dot1q, native 99, allowed restreint)
☐
6
LACP Port-channel 1 entre DSW-01 et DSW-02
☐
7
ip routing et SVIs sur les deux DSW
☐
8
STP root primary/secondary aligné HSRP
☐
9
HSRP croisé actif/standby sur les 4 VLANs avec auth MD5 et tracking
☐
10
Lien Internet routed avec HSRP outside et route par défaut
☐
11
DHCP relay sur SVIs utilisateurs
☐
12
Tests de connectivité depuis les PCs
☐
13
IP mgmt des ASW avec default gateway
☐
14
SSH actif sur tous les switches
☐
15
NAT (PAT) sur les DSW pour accès Internet
☐
16
Test failover HSRP
☐
Bonus
VLAN 900 et OSPF entre DSW (anti-blackhole asymétrique)
☐
Pour la prochaine session (après-midi)
La session 2 portera sur les ACL sur cette même infrastructure. Vous appliquerez une matrice de flux Zero Trust entre les VLANs. Le SSH que vous venez d'activer servira à vous reconnecter aux switches une fois que les ACL bloqueront certains accès. Pensez à bien sauvegarder vos configurations (write memory) avant de fermer EVE-NG.
Bon TP.
