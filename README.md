# Projet Virtualisation et Cloud - Groupe 09

## Membres du groupe

- DUVEAU Pierre
- MEUNIER Paul
- NICOLE Clément
- PRATOUSSY Martin


## SOMMAIRE
-   [Définitions ](#definitions)
-   [Découpage Réseau / sous-réseau et IP](#IP)
-   [Se connecter à Vcenter](#connect)
-   [Intallation un ESXi](#esxi)
-   [Installation du Contrôleur de Domaine](#dc)
-   [Installation des Utilisateurs (1, 2, 3)](#users)
-   [vCenter](#vcenter)

## Definitions <a name="definitions"></a>

- **ESXI** VMware ESXi est un hyperviseur de type entreprise de type 1 développé par VMware pour déployer et servir des ordinateurs virtuels.
- **CD** Un serveur informatique hébergeant l'annuaire Active Directory est appelé « contrôleur de domaine ». Active Directory stocke ses informations et paramètres dans une base de données distribuée sur un ou plusieurs contrôleurs de domaine, la réplication étant prise en charge nativement.
- **VSPhere** VMware vSphere est un logiciel d'infrastructure de Cloud computing de l'éditeur VMware, c'est un hyperviseur de type 1, basé sur l’architecture VMware ESXi. VMware vSphere nécessite une configuration matérielle restreinte précisée dans le guide de compatibilité VMware.
- **VCenter** vCenter Server est l'utilitaire de gestion centralisée pour VMware et est utilisé pour gérer les machines virtuelles, plusieurs hôtes ESXi et tous les composants dépendants à partir d'un emplacement centralisé unique.
- **NAS** Un serveur de stockage en réseau, également appelé stockage en réseau NAS, boîtier de stockage en réseau ou plus simplement NAS, est un serveur de fichiers autonome, relié à un réseau, dont la principale fonction est le stockage de données en un volume centralisé pour des clients réseau hétérogènes.
- **SAN** En informatique, un réseau de stockage, ou SAN, est un réseau spécialisé permettant de mutualiser des ressources de stockage.

## Découpage Réseau / sous-réseau et IP <a name="IP"></a>

D'après le schéma fournit par nos enseignants, nous savons que notre plage d'adresses IP en Management va de `192.168.17.144` à `192.168.17.159`, et que notre découpage IP se doit d'être similaire au schéma ci-dessous :

![](images/SchémaProf.png)

Nous avons donc défini le découpage suivant :

Management :bookmark_tabs: :

```txt
dc : 192.168.17.146 /24
vcsa : 192.168.17.147 /24
esxi01 : 192.168.17.148 /24
esxi02 : 192.168.17.149 /24
esxi03 : 192.168.17.150 /24
freenas : 192.168.17.151 /24
```

Prod :desktop_computer: :

````
dc : 10.20.30.1 /24
user01 : 10.20.30.2 /24
user02 : 10.20.30.3 /24
user03 : 10.20.30.4 / 24
```txt

Stockage :floppy_disk: :
```txt
esxi01 : 172.16.0.2 /24
esxi02 : 172.16.0.3 /24
esxi03 : 172.16.0.4 /24
freenas : 172.16.0.5 /24
````

vMotion  :inbox_tray: :

```txt
esxi01 : 10.50.100.2 /24
esxi02 : 10.50.100.3 /24
esxi03 : 10.50.100.4 /24

```

vSwitch :

```
VMnic0 : (Interface) pour le management (1 Vswitch)
VMnic1 : (Interface)pour la prod (1 autre Vswitch)
VMnic2 : pour ISCSI ( 1 autre autre Vswitch)
VMnic3 : pour Vmotion ( 1 autre autre autre Vswitch)
```

## Se connecter à Vcenter <a name="connect"></a>

Pour se connecter au VCenter de l'école, nous avons besoin d'un VPN.

Il vous faut installer OpenVPN :

- Ubuntu : `apt-get install openvpn`
- Windows : [Télécharger OpenVPN](https://openvpn.net/client-connect-vpn-for-windows/)

Une fois installer, vous allez devoir le configurer, pour se faire vous allez devoir télécharger le fichier OpenVPN founit par vos enseignants :

Votre VPN devrait ressembler à ça :
> Choisissez OpenVPN
![](images/OpenVPN.png)

> Renseignez votre nom d'utilisateur ainsi que votre mot de passe CPE
![](images/VPNConfig.png)

Pour mettre en place une redirection DNS :

- Effectuez la commande suivante dans un shell : `sudo nano /etc/hosts`
- Puis renseignez votre redirection DNS : 
```bash
192.168.16.10   vcenter.tpv.cpe.localdomain
```

Renseignez dans l'URL : `http://vcenter.tpv.cpe.localdomain`

Nom d'utilisateur : `virtu-09@tpv.cpe.localdomain`
Mot de passe : `cf. l'excel fournit par vos enseignants`

## Intallation un ESXi <a name="esxi"></a>

### Création de l'ESXi

Cette manipulatione est a effectuée sur le Vcenter (si vous n'êtes pas connecté sur le Vcenter, merci de regarder le point ci-dessus)

> Faire un clique-droit sur votre groupe -> 'Nouvelle machine virtuelle'

> Choisir là où va se créer la VM, ainsi que son nom :
> ![](images/esxi01-1.png)

> Choisir son cluster :
> ![](images/esxi01-2.png)

> Choisir l'emplacement de stockage votre ESXi :
> ![](images/esxi01-3.png)

> Choisir la compatibilité de votre ESXi :
> ![](images/esxi01-4.png)

> Choisir le type de machine virtuelle que l'on veut créer (`ESXi 7.0 or later` dans notre cas) :
> ![](images/esxi01-5.png)

> Configurer les différents réseaux & le stockage disque
> 
> :warning: bien mettre le disque en mode `Thin Provision | Provisionnement Dynamique`
> ![](images/esxi01-6.png)
> ![](images/esxi01-7.png)

### Paramétrage de l'ESXi :

> Vous devrez vous identifiez sur l'ESXi (root / Vmware1!)
> ![](images/esxi-01.png)

> On va ensuite configurer l'adaptateur réseau que l'on souhaite utiliser
> ![](images/esxi-02.png)
> ![](images/esxi-03.png)

> On va définir notre @IP notre Masque et Passerelle par défaut 
> ![](images/esxi-04.png)



## Installation du Contrôleur de Domaine <a name="dc"></a>
Le controlleur de dommaine aura pour rôle sur notre réseau de distribuer les différentes @IP distribuées en DHCP aux différents *"users"*, et d'installer le `VCSA` sur l'ESXi de notre choix.


### Création du DC 
> Une fois connecté sur votre ESXi, clique droit sur le menu `Machines virtuelles` -> `Créer/Enregister une VM`

>Choisir le nom de la machine ainsi que l'OS à implementer :
>![](images/DC-1.png)

> Choisir l'endroit ou l'on va stocker notre DC :
>![](images/DC-2.png)

> Ajouter les adaptateurs réseau pour qu'il puisse être connecté aux différents réseaux (Prod, Management, Stockage et vMotion)
>![](images/DC-3.png)


### Paramétrage effectué par la suite :

```
- Création d'un Active Directory (Nom : Gr9.dom)
- Création de zone DNS (Zone de recherche direct et zone de recherche inversée) pour les ESXI, VCenter, DC, sur le réseau management 
- Création de zone DNS dynamique pour le réseau de production
- Création d'un pool DHCP pour le réseau de production
- Création de la table de routage et de translation NAT pour le réseau de production
```

## Installation des Utilisateurs (1, 2, 3) <a name="users"></a>

Suivant le schéma fournit par nos enseignants, nous avons installé les VM correspondantes aux utilisateurs :
- user01 : windows
- user02 : windows
- user03 : linux (ubuntu)

L'installation est similaire à celle du DC, mais au lieu de sélectionner une image serveur, nous utiliserons une image classique.

Grâce au DC, nos utilisateurs peuvent recevoir une adresse IP dynamique en fonction d'une plage prédéfinie, peut importe l'ESXi sur lequels ils se trouvent.

## vCenter <a name="vcenter"></a>

Notre groupe a mit beaucoup de temps pour arriver à cette étape, et pour cause le fait que nous sommes 4 élèves issue d'un BTS option SLAM (développeur) nous n'avions que de faible base en virtualisation, il fût donc plus compliqué pour nous d'arriver jusque là.

De plus nous avons essayer d'installer `VCSA` (ou vCenter) à plusieurs reprises, mais les machines que nos enseignants nous on fournit, n'ont plus d'espace disques (- 100Go chacune) hots il faut un minnimum de 150Go pour passer la première étape d'installation du `VCSA`.

Nous nous somme donc retrouvé face à un dilemme, car nous n'avons pas pu plus avancer et nous devons rendre un compte rendu très pauvre du à la limitation technique qui nous est imposé.

Nous espérons que cet incident (défaut de stockage), qui nous a empêché de travailler sur les dernières scéances et à la maison ne portera pas préjudice à notre note, car nous avons été défavorisé face aux autres groupes qui ont du occuper un espace de stockage plus important nous empêchant alors de progresser d'avantage.