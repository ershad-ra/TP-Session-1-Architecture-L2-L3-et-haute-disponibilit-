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

<aside>
💡

![image.png](attachment:88b4479d-73b7-4998-a8fa-752d49dacf9a:image.png)

</aside>

```

```
