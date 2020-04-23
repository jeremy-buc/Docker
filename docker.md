# Cours Docker & Sécurisation
##### Rémi BOUCHET - Pierre BROCH - Alexis BRONIARCZYK - Jérémy BUC `23/04/2020`

# Présentation Docker

<center><img src="https://i.imgur.com/IClKjyk.png" width="400"/></center>

Docker est un environement complet pour le déploiement d'applications dans des containers, qui permet l'exécution de containers de façon maitrisé, docker permet donc l’utilisation de distribution et d'outils/applications de votre choix par le biais de container. Docker est donc un hyperviseur de containers
L’architecture Docker se repose sur trois parties:

-	Docker_Client qui permet la communication entre l’utilisateur et le Daemon
-	Docker_Host qui contient le Daemon Docker est permet la création de conteneur 
-	Registry qui gere l’envoi, le stockage et la récupération des images sur les différents registres

Docker peut utiliser des images vierges ou bien des images personnalisés via des scripts pour la création de container personnalisé

<center><img src="https://i.imgur.com/ybHf7dj.jpg" width="400"/></center>

---

# Sécurité classique

Docker est un outil complexe mais pratique, or, si on ne fait pas un minimum attention, nous pouvons avoir trés vite des problémes de sécurité.

Voici donc quelques conseils pratique et simplistes afin de sécuriser son infrastructure Docker : 


### Créer une partition séparée pour Docker :
  
  Par défaut, toutes les données relatives à docker sont stockées dans */var/lib/docker*. Ce répértoire est la pour stocker les images et les containes.
  
  L'inconvénient est que ce répertoire est situé sours la racine / et que docker peut prendre énormément de place, or si le répertoire racine est pleins, cela rendra votre machine hôte hors service.
  
  Une "image hack" pourrait volontairement remplir cette espace dans le but de rendre votre infrastructure inutilisable, mais aussi, docker naturellement peut remplir cette espace.
  
  
<center> <img class="img" src="https://i.imgur.com/5Z1G0Yl.jpg" width="400"/></center>

  
### Maintenez votre systéme hôte à jour :
  Cela est souvent répété, mais il est essentiel que tout les élements en relation avec votre systéme docker soit à jours (**Linux//Kernel//Docker Engine**)
  
  Fuyez les versions instables et mettez à jours votre noyau docker également.
  
<center><img src="https://i.imgur.com/e1JeEDj.png" alt="drawing" width="400"/></center>

### Interdire les communications entre docker :
  Par défaut, la communication entre tous les containers est possible, sans forcément utiliser la fonction "link", un outil malveillant ou une image corrumpu pourrait facilement pratiquer une technique de "sniffing" et donc par conséquent, voir tout ce qui se passe sur le sous réseau Docker.
  
  C'est une technique dangereuse car dans la majorité des cas, il n'y a pas de connexion sécurisée sur le réseaux "**docker0**"
  
  Il faut donc interdire la communication entre les containers, et appliquer la fonction "link" entre les containers qui ont besoins de communiquer
  
  Cette fonction est native à Docker, il suffit d'intégrer sur le daemon la fonction : **-icc=false**
  
  Sous Debian // Ubuntu dans le fichier */etc/default/docker* modifier la variable "DOCKER_OPTS" comme ceci : 

```sh  
  DOCKER_OPTS="-icc=false"
``` 
<center><img src="https://i.imgur.com/2YMSYpe.jpg" alt="drawing" width="400"/></center>


### Ne pas utiliser "privileged" pour n'importe qu'elle image :
  
  Quand un container est lancé avec l'argument "-privileged", Docker va donc lui donner tout les droits, y compris de lancer des nouveaux container sur la machine Hôte (Docker in Docker)
  
  Un container de ne doit pas avoir plus de droit que nécessaire, c'est pour cela qu'il existe les fonctions **"-cap-add" "-cap-drop"**, afin d'ajuster les droits sur vos containers.
  
  Par exemple, un container de NTP doit avoir le droit de modifier l'heure du systéme hôte donc on peut faire comme ceci : 
  
```sh  
  docker run -d --cap-add SYS_TIME ntpd
``` 

 <center><img src="https://i.imgur.com/w9DEMa1.jpg" width="250"/></center>

### Ne pas choisir n'importe quel registry

La plupart des images sont téléchargeables depuis le registry Docker Hub. D’une manière générale, préférer toujours les images “official” proposées et validées par Docker. Si ce n’est pas le cas, assurez que l’image a suffisamment été téléchargée
et à priori “testés” par les autres utilisateurs.

N’hésitez pas non plus à aller vérifier le Dockerfile sur Github 

Il est important également de vérifier les images intermédiaires utilisées et remonter les différentes images jusqu’à une image official ou une image from scratch.

Il est possible que vous ne trouviez pas l’image sur Docker Hub, vérifier donc la réputation du registry que vous allez utiliser, mais également que la communication établie avec ce nouveau registry se fera de manière sécurisé. 

Par défaut, Docker bloque la commande pull si ce n’est pas le cas.

### Créer un utilisateur dans votre Dockerfile

La plupart de vos applications n'ont pas besoin d’être lancé en root.

Par conséquent, il n’est pas nécessaire d’utiliser le user root dans vos containers.

La création d’un utilisateur dans votre image peut se faire directement depuis le Dockerfile avec les instructions suivantes :
```sh  
  RUN useradd -d /home/myappuser -m -s /bin/bash myappuser
  USER myappuser 
``` 

Pour vérifier l'utilisateur utilisé par vos containers faire ceci : 
  
  ```sh  
docker ps -q | xargs docker inspect --format '{{ .Id }}: User={{.Config.User}}'
``` 

Si User est vide, vos containers sont lancés en root.

Pour les images, vérifier la présence de l'instruction "user" dans le Dockerfile :
 ```sh  
FROM apacheOnlyRoot
USER www-data
``` 
### Ne lancer pas vos containers en root

Cela rejoint le point précédent, mais si votre image ne nécessite l’utilisateur root, il est préférable de ne pas lancer le container avec l’utilisateur root

Voici la commande pour le faire : 

```sh  
docker run -u <Username or ID> <Run args> <Container Image Name or ID> <Command>
``` 

Un exemple avec une image apache :

```sh  
docker run -u www-data -d -t apache
``` 
Cette option est également disponible dans "docker-compose" :
```sh  
user: www-data
``` 
<center><img src="https://i.imgur.com/2soA3X5.jpg" width="300"/></center>

### Ne mapper que les ports utiles : 

Docker propose un mapping entre les ports déclarés par le Dockerfile et ceux ouvert sur votre système hôte. L’option *-P* (en majuscule)  permet lancer un container en exposant tous les ports déclarés dans votre Dockerfile.

N’utiliser jamais l’option -P (en majuscule), préférez l’option -p (en minuscule) qui permet mapper les ports un par un, ex :
```sh  
user: www-datadocker run -p 80:80 apache
```

De plus, si vous n’avez pas besoin de rendre public l’adresse IP, vous pouvez spécifier sur quel interface réseau vous souhaitez écouter : 
```sh  
docker run -p 192.168.0.1:9200:9200 elasticsearch
```

<center><img src="https://i.imgur.com/9YlYQTy.jpg" width="400"/></center>

### Les permissions sur les fichiers :

De la même manière qu’on doit vérifier les permissions du répertoire */var/www* pour un serveur WEB, il faut également vérifier les permissions pour un container faisant tourner un serveur apache.



Docker est certes très facile à prendre en main, mais il faut pour autant comprendre les notions de base d’un système unix pour assurer à minima la sécurité d’un container tournant sous Linux grâce aux permissions et aux groupes utilisateurs.

---

# Sécurité avancée


### Utilisation des cgroups pour limiter les ressources

Les groupes de contrôle(ou cgroups) sont une fonctionnalité au niveau du noyau qui permet à Docker de contrôler les ressources auxquelles chaque container peut accéder, garantissant ainsi une bonne multi-tenacité des containers.

Les groupes de contrôle permettent à Docker de partager les ressources matérielles disponibles et, si nécessaire, de définir des limites et des contraintes pour les containers. 

Un bon exemple de cela est l'application de limites à la quantité de mémoire disponible pour un container spécifique afin qu'il ne puisse pas épuiser les ressources de l'hôte.

### Namespaces

Docker profite de Namespaces Linux pour fournir l'espace de travail isolé que nous appelons un container. 
Lorsqu'un container est déployé, Docker crée un ensemble de Namespaces pour ce container spécifique, l'isolant de tous les autres containers en cours d'exécution. 

Les différents Namespaces créés pour un container comprennent :

- PID : Chaque fois qu'un programme démarre, un numéro d'identification unique est attribué au Namespace qui est différent du système hôte. Chaque container possède son propre Namespace PID pour ses processus

- MNT :  Chaque container dispose de son propre Namespace pour les chemins d'accès aux répertoires de montage.

- NET :  Chaque container dispose de sa propre vue de la pile réseau, ce qui évite un accès privilégié aux sockets ou aux interfaces d'un autre container. 

- UTS :  Il assure l'isolation entre les identificateurs du système, le nom d'hôte et le nom de domaine NIS. 

- CIP :  Le Namespace de la communication entre processus (IPC) crée un regroupement où les containers ne peuvent voir et communiquer qu'avec d'autres processus dans le même Namespace IPC

### Limiter les autorisations root dans un container Docker grâce au contrôle des capabilities 

Traditionnellement, l'utilisateur root a accès à toutes les capabilities ; les utilisateurs non root ont un ensemble de capabilities plus restreint, mais ont la possibilité d'augmenter leur accès au niveau root par l'utilisation de sudo ou de setuid binaries. 
> **Cela peut constituer un risque pour la sécurité.**
Les paramètres par défaut de Docker sont conçus pour limiter les capabilities de Linux et réduire ce risque.

Cela réduit la possibilité que les vulnérabilités au niveau des applications soient exploitées pour permettre une escalade vers un utilisateur root entièrement privilégié. 

Dans la plupart des cas, les containers d'application n'ont pas besoin de toutes les capabilities attribuées à l'utilisateur root puisqu'une grande majorité des tâches nécessitant ce niveau de privilège sont traitées par l'environnement de l'OS en dehors du container. 

Par conséquent, les containers peuvent s'exécuter avec un ensemble de capabilities réduit sans que l'application n'en pâtisse. 

Cela augmente le niveau de sécurité global du système et rend l'exécution des applications plus sûre par défaut. Comme les capabilities des containers sont fondamentalement limitées, il est également difficile de provoquer des dommages au niveau du système lors d'une intrusion, même si l'intrus parvient à s'infiltrer jusqu'à la racine d'un container.

### Protégez-vous d'une forkbomb dans un container

Une forkbomb est une attaque par déni de service, elle consiste à créer des processus qui vont eux-mêmes générer d'autres processus, ce qui aura pour effet de faire planter votre hôte, car celui-ci accepte un certains nombres d'éxecution simultanées.
<center>
:(FORKBOMB){:|:&};:
</center>
<center><img src="https://i.imgur.com/1CC50s7.png" width="200"/></center>
Vous pouvez vous en protéger, en limitant le nombre de process accepté durant le lancement du container en utilisant l'option :  `--pids-limit `
 
### Fonctionnement Rootless

Le mode Rootless permet d'exécuter le daemon Docker et les containers en tant qu'utilisateur non-root, afin de réduire les vulnérabilités potentielles du daemon et du container.

Le mode Rootless exécute les daemons et containers Docker dans le user Namespace. Dans ce mode, le daemon et le container s'exécutent ***sans privilèges d'utilisateur root.*** Ce fonctionnemnt réduis alors, considérablement les failles de sécurité.

Voici une conférence sur les détails de la mise en œuvre du mode Rootless et les améliorations prévues : [Hardening Docker daemon with Rootless mode](https://www.youtube.com/watch?v=Qq78zfXUq18)

Ce mode Rootless est d'ailleurs, très souvent associé à podman, présenté plus bas.

---

# Alternative 

### - LXC Linux Container 

LXC (linux containers) est l'ensemble bien connu d'outils, modèles, bibliothèque et connecteurs pour différents langages. LXC est bas niveau, très flexible et couvre à peu près toutes les technologies de confinement supportées par le noyau.

LXC est prêt pour la production. La version 1.0 est supportée pour 5 ans (jusqu'en avril 2019) avec des mises à jour de sécurité et correctifs .
L'objectif de LXC est d'offrir un environnement neutre, non influencé par des distributions ou fournisseurs.

LXC est un système de virtualisation, utilisant l'isolation comme méthode de cloisonement au niveau du système d'exploitation, qui permet de créer des conteneurs.
Il ne faut pas confondre LXC et LXD, LXD et une surcouche logicielle à LXC.


### - LXD 

LXD est la nouvelle expérience LXC avec une seule commande intuitive pour contrôler tous vos conteneurs. Les conteneurs sont gérés par le réseau de façon complètement transparente grâce à une interface REST. Il marche aussi à grande échelle, s'intégrant avec OpenStack.

LXD a été annoncé en début Novembre 2014 et est en développement actif.
Les bénéfices de LXD sont:

- une interface de commende performante
- une haute scalabilité
- une sécurité améliorer
- un meilleur controle et gestion des ressources
- un management du stockage et des interfaces réseaux

### - rkt de CoreOS


RKT est un container runtime créé par les équipes de CoreOS, avec une première release fin novembre 2014 et qui est en constante évolution depuis .
RKT se concentre sur les points suivants:

- La sécurité: la signature ainsi que l’intégrité des images est vérifiée par défaut au téléchargement ou au lancement d'un container
- La composabilité: est à la fois interne et externe, en interne il permet le support de plusieurs moteurs d'exécution pour les containers et en externe propose une compatibilité avec d'autres orchestrateurs tels que Kubernetes
- La compatibilité: toutes les images Docker sont supportées nativement


### - Podman
<img src="https://i.imgur.com/wXxNnTW.png" width="180"/>

Podman est un projet open source. Le projet a initialement émergé au sein du projet CRI-O, sous le nom de kpod. Podman est spécialisé dans l’exécution des containers. 

Podman se distingue de Docker par, entre autres, son fonctionnement sans daemon et sa possibilité de fonctionner sans droits root, et donc de pouvoir lancer different container en utilisant différent namespaces ce qui donne une meilleur séparation entre les containers.
Podman permet de construire et d'utiliser des containers sans compromettre la sécurité en utilisant des accées non root au utilisateurs(rootless).

Tous les containers sont lancés par runC sans dépendre d’un processus unique.
Podman a été conçu pour fonctionner de pair avec Kubernetes. Pour à la fois lancer des Pods* ou générer des manifests à partir d'un Pod existant.

CRI-O (Container Runtime Interface) vient de remplacer le daemon docker.
Plus sure, CRI-O déclanche une instance de runC par pod.Tout les pod ne sont pas lié a un seul daemon qui est un SPOF(single point of failure). 

*Pods = Un pod est un groupe d'un ou plusieurs containers.

---



# Conclusion

Il existe de nombreux avantages à utiliser Docker dans les domaines Web et informatique. Les services informatiques fournissent de plus en plus de services orientés Web et les conteneurs peuvent être un élément clé pour accélérer leur transition vers le cloud. 

Contrairement à un serveur virtuel sous Linux, ce conteneur ne nécessite que des centaines de Mo de disques. L'empreinte mémoire est également réduite, car nous n'utilisons de la mémoire que pour les applications (pas de couche OS). Par conséquent, il démarre plus rapidement et peut également être déplacé d'une machine à une autre. 

Avec une puissance de traitement rapide et une grande efficacité, de nombreuses personnes développent et testent dans des conteneurs Docker. Les développeurs peuvent rapidement et facilement posséder un environnement de développement sans avoir à demander un déploiement de VM à leur fournisseur de services ou à leur équipe d'infrastructure.
 
Des technologies comme Docker changeront les habitudes de travail et les limites des rôles, et elles accéléreront certainement l'adoption du cloud. 
Cela doit être en harmonie avec les développeurs et les équipes d'infrastructure dans l'esprit de ***Devops*** afin d'atteindre l'efficacité et l'agilité dans le développement continu des chaînes de production intégrées.

Ce développement devra également suivre au niveau de la sécurité, comme nous avons pu le décrire tout au long de ce document, afin de garantir une fiabilité, l'intégrité des données et des systèmes impactés par ces containers.


## 

[![Button](https://i.imgur.com/bfX43uC.png)](https://github.com/IUT-Beziers/secudocker-asur-2020-bouchet-broch-broniarczyk-buc/blob/master/Cours-Docker.md) [![Button](https://i.imgur.com/2t5fGU1.png)](https://hub.docker.com/)
Copyright © 2020 Toutes reproductions ou utilisation de ce document sans l'accord de ces propriétaires est interdites.
