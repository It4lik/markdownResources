# Conteneurs & Sécurité

Ce document a pour but d'éclairer plusieurs aspects de sécurité autour de la conteneurisation.
Avant d'aller plus loin, il est nécessaire de préciser que **nous nous intéresserons principalement ici à Docker.** C'est en effet la technologie la plus populaire autour de la conteneurisation, sur tous les points (mouvement global, développeurs, administrateurs, etc).
De plus c'est surtout la **conteneurisation sous Linux** qui sera abordée ici. Ceci aussi pour des raisons simples :
* il existe une moins grande communauté qui l'utilise sur d'autres plateformes (peut-être Azure...)
* dans le cas de l'utilisation de conteneurs en production il est souvent recommandé d'utiliser des hôtes GNU/Linux (bien que l'outil soit de plus en plus stable sur les autres plateformes. Nous ne discuterons pas de ces points ici.)
* l'engine (ou runtime) de Docker a été pensé pour fonctionner à l'aide de technologies natives du noyau Linux. Bien que le portage soit actuellement un succès (avec Docker for Mac et Docker for Windows)  sur les autres plateformes grand public, cela reste un portage (on utilise les fonctionnalités Hyper-V de win10 et winserver côté Windows, et celles de xhyve pour MacOS)

Aussi, il est très difficile de recouvrir tous les aspects de sécurité autour de Docker et de la conteneurisation de façon générale. Nous nous intéresserons ici aux aspects incontournables touchant directement à la limitation de droits ou de surface d'exposition. Mais nous discuterons aussi de certaines problématiques autour du stockage, de la sauvegarde (disponibilité des données) ou encore à certaines autres autour de problèmes cryptographiques (accès sécurisé à un démon, signature des images, etc.).

Bien que n'étant pas sa vocation première, nous nous efforcerons dans ce document de revoir certains aspects techniques et bas-niveau qu'il est absolument indispensable de connaître et comprendre si on souhaite réellement appréhender la sécurité autour de Docker.

Ce document n'a pas non plus pour vocation d'indiquer la marche technique à suivre pour obtenir un environnement Docker sécurisé et robuste. Nous ne venons ici que mettre en évidence certaines problématiques et y apporter des éléments de réponse.

Enfin, il est attendu de la part du lecteur de connaître un minimum la conteneurisation à l'aide de Docker et son écosystème (bien que beaucoup d'éléments soit présentés brièvement sur le tas).

# Introduction
Avant toute chose, il est nécessaire de rappeler très brièvement (car ce n'est pas l'objectif de ce document) les objectifs de la conteneurisation, de manière purement fonctionnelle :
* faciliter le **packaging** des applicatifs : un seul système de packaging
* faciliter le **déploiement** d'un applicatif : peu importe le système d'exploitation sous-jacent car c'est l'engine de conteneurisation qui s'occupera de faire tourner l'application
* faciliter le **développement collaboratif** : le code, une fois packagé dans une un conteneur (ou plutôt, une image) est simple à transporter d'un poste à un autre, sans se préocupper ni de l'OS ni des librairies nécessaires
* accéder à un **plus grand niveau de sécurité**
  * ajoute un niveau d'isolation (au niveau du kernel principalement)
  * les conteneurs sont stateless *par définition* (reproductibilité, conformité)

Aussi, avant d'embrayer sur le développement, j'aimerai apporter quelques faits. Les mettre en avant dans l'introduction est un choix purement subjectif, mais je pense qu'il est primordial d'avoir vent de ces faits :

* les conteneurs reposent **exclusivement** sur des technologies kernel qui existaient déjà auparavant, qui étaient déjà utilisées pour les mêmes usages. Si **aujourd'hui** (après plusieurs années de dév, rapprochement de la Linux Foundation, création de standards, sensibilisation globale, etc.) vous n'accordez pas de confiance à Docker, alors vous n'accordez pas de confiance au noyau Linux.
* on parle du déploiement, et surtout du lancement d'applications
  * lancement d'applications : c'est un des rôles de systemd (à l'aide des unités de services)
  * **il est admis qu'on ait besoin des droits `root` pour systemd ? Alors pourquoi pas pour Docker ?...** C'est exactement la même surface d'exposition (lorsqu'on parle du lancement d'application), et c'est pour strictement les mêmes raisons qu'ils ont besoin des droits `root`
  * le démon docker tourne avec `root`, **ce qui n'est pas le cas des conteneurs pris individuellement**. Un conteneur n'est qu'un unique processus, une simple ligne dans un ``ps``.   
  De la même façon, systemd tourne sous `root`, mais les services qu'ils lancent peuvent posséder un utilisateur applicatif.
* la conteneurisation ne fait qu'apporter un niveau d'isolation supérieur. Bien sûr que cela soulève un nombre incalculable de nouvelles problématiques. Mais par nature, c'est simplement un nouveau système  d'isolation applicative. Et donc une hausse du niveau de sécurité. **Par nature.**
* il est **purement inutile** de comparer la VM et les conteneurs sur le plan de la sécurité
  * les deux reposent sur des principes fondamentalement différents
  * ce ne sont pas les mêmes usages ! Donc pas les mêmes surfaces d'exposition, pas les mêmes problématiques, etc.
  * **l'hôte de conteneurisation EST UNE VM** : le conteneur n'apporte qu'un niveau supérieur d'isolation, encore une fois.
  * on peut bien sûr effectuer la comparaison avec la VM, parce qu'on connaît bien : ça peut aider à avoir une vision globale. Mais **pour juger du niveau de sécurité des conteneurs de manière effective, c'est un raisonnement depuis zéro qu'il faut adopter et non pas un raisonnement comparatif**.
* il n'y aucune raison de mettre à la trappe l'isolation réseau (permettant d'isoler des services ayant un niveau de confidentialité différents, ou des surfaces d'exposition plus ou moins grandes etc.) si on utilise de la conteneurisation.
  * une application qui tournait sur une VM dans tel sous-réseau, qui est "conteneurisée" devra tourner dans le même sous-réseau, sur un hôte de conteneur éventuellement dédié.
  * dans ce cas-là il n'y AUCUN changement, si ce n'est : facilité de mise-à-jour, de packaging, de déploiement/redéploiement, de migration, etc
  * je parle d'isolation réseau, mais il en va de même pour le stockage, la sauvegarde, etc.

# Sommaire
* [Sources](#sources)
* [Communauté & standards](#communauté--standards)
  * [Un peu d'histoire](#un-peu-dhistoire)
  * [OCI : Open Container Initiative](#open-container-initiative)
* [How do containers work ?](#how-do-containers-work-)
  * [Technologies noyau](#technologies-noyau)
  * [Rentrons dans le détail...](#rentrons-un-peu-dans-le-détail)
* [Questions récurrentes](#questions-récurrentes)
* [Sécurité & bonnes pratiques Docker](#sécurité-et-bonnes-pratiques-docker)
  * [Le démon Docker](#sécurité-du-démon-docker)
    * [Exposition de l'API](#exposition-de-lapi)
    * [Options du démon Docker](#options-du-démon-docker)
  * [Les hôtes Docker](#sécurité-des-hôtes-docker)
  * [Les images Docker](#sécurité-des-images-docker)
    * [Qu'est ce qu'une image ?](#quest-ce-quune-image-docker-)
    * [Le registre Docker](#le-registre-docker)
    * [Construction d'images](#construction-dimages)
  * [Les conteneurs Docker](#sécurité-des-conteneurs-docker)
    * [Options lors du `docker run`](#les-options)
    * [Gestion des utilisateurs](#gestion-des-utilisateurs)
      * [Qui communique avec le démon ?](#-communication-avec-le-démon-docker)
      * [Qui lance les processus ?](#-lancement-des-processus)
      * [Utilisation des namespaces de type `user`](#-user-namespace-remapping)
      * [Utilisateurs applicatifs dans un conteneur ?](#-utilisateurs-applicatifs-)
  * [Discussion autour de l'orchestration](#discussion-autour-des-systèmes-dorchestration-de-conteneurs)
  * [Réseau : aperçu](#le-réseau-dans-docker)
  * [Stockage : aperçu](#le-stockage-et-la-sauvegarde-avec-docker)
    * [Conteneurs & Données](#des-conteneurs-et-des-données)
    * [Volumes Docker](#les-volumes-docker)
    * [Volumes & frameworks d'orchestration](#gestion-des-volumes-au-sein-dun-framework-dorchestration)
    * [Sauvegarde](#la-sauvegarde)
  * [Renforcer la sécurité de Docker](#applicatifs-permettant-de-renforcer-la-sécurité-de-docker)
    * [Signature des images](#signatures-des-images--notary)
    * [Scanner de vulnérabilités pour images Docker](#scanner-de-vulnérabilités-pour-les-images--clair)
    * [Systèmes de sécurité kernel](#systèmes-de-sécurité-kernel)
      * [SELinux](#selinux)
      * [AppArmor](#apparmor)
      * [Seccomp](#seccomp)
  * [Limites de l'isolation](#limites-de-lisolation-de-type-docker)
    * [Visualisaiton des ressources de l'hôte](#visualisation-des-ressources-de-lhôte)




# Sources
Mes sources sont très nombreuses, et une grande partie de ce document a été rédigé d'une traite.
La majeure partie des informations trouvables dans ce document se trouvent dans les référentiels suivant :
* [The Linux Documentation Project](http://www.tldp.org/)
* [Documentation Red Hat](https://access.redhat.com/documentation/en/red-hat-enterprise-linux/)
  - [Red Hat Container Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/)
* [Documentation Docker](https://docs.docker.com/)
* [Documentation Kernel](https://www.kernel.org)
* la commande ```man``` (trouvable aussi [ici](http://man7.org/linux/man-pages/man7/))
  * ce sont les documentations les plus simples et complètes sur les fonctionnalités du kernel (les mans ne documentent pas que des binaires...)
  * plus clair, c'est lire le code.
  * ``man cgroups``, ``man namespaces``, etc.
* [Wiki de la Linux Foundation](https://wiki.linuxfoundation.org)
* [Site vitrine de l'OCI](https://www.opencontainers.org/)
* Recommandations de sécurité
  * [par Delve Labs](https://www.delve-labs.com/articles/docker-security-production-2/)
  * [par GDSSecurity](https://github.com/GDSSecurity/Docker-Secure-Deployment-Guidelines)

Certains autres documents constituent des lectures très intéressantes autour des conteneurs :
* Cisco et RedHat : ["Linux Containers: Why They’re in Your Future and What Has to
Happen First"](https://www.cisco.com/c/dam/en/us/solutions/collateral/data-center-virtualization/openstack-at-cisco/linux-containers-white-paper-cisco-red-hat.pdf)


**Ce document n'est donc qu'une synthèse d'un savoir communautaire.**

# Communauté & standards
## Un peu d'histoire
A l'origine, la société **Docker avait mis sur pied sa propre librairie afin de pouvoir utiliser des conteneurs**. Ce n'est ni plus ni moins qu'une interface (au sens premier du terme) vers le kernel, qui expose une API simplifiée et dédiée à la conteneurisation.
De façon simple, avant l'existence de telles librairies (et libcontainer **n'était pas** la première), il était nécessaire de créer manuellement les `cgroups` et namespaces, ainsi que de les articuler entre eux, afin d'avoir un conteneur. Avec une telle lib, tout ce travail a été simplifié.

La société a aussi mis à disposition **un outil CLI très puissant et clé-en-main**, permettant d'utiliser cette API.

Mais, l'engouement autour de la solution étant grandissant, et la croissance du nombre de client exponentielle (car répondant à un certain nombre de besoins, ce dont nous ne traiterons pas ici), d'autres engines de conteneurisation ont rapidement vu le jour.

S'est alors rapidement posée **la question de la standardisation**. On ne parle plus de Docker, qui a révélé ces technologies au plus grand nombre, mais de la conteneurisation en général, dans le monde de demain. La question n'est en effet plus de savoir si les conteneurs seront là demain, on connaît la réponse et c'est oui. La question c'est : comment ?

## Open Container Initiative
L'OCI est un projet de la Linux Foundation, s'inscrivant dans cette mouvance globale à vouloir standardiser un maximum de choses afin de permettre une évolution de l'informatique au sens large de façon simple et sans accrocs. On peut aussi par exemple noter l'[Open API Initiative](https://www.openapis.org/) qui s'inscrit dans ce même mouvement.

L'OCI propose donc des spécifications, des standards : pour les images de conteneurs, ou encore pour l'engine lui-même. Pour le runtime, on trouve par exemple la spécification [ici](https://github.com/opencontainers/runtime-spec) et son implémentation libre et open-source [ici](https://github.com/opencontainers/runc). L'implémentation porte le nom de ```runc``` et c'est désomais le runtime utilisé notamment par Docker. **Docker n'est plus, vive Docker.**

A noter que le projet est soutenu par de grands acteurs parmis lesquels Microsoft, IBM, Huawei, EMC, Docker, Dell, Google, HP, Goldman Sachs (hum hum...), Twitter, RedHat, etc.

Ceci est déterminant pour l'avenir de la conteneurisation. En effet, le développement de la conteneurisation au sens large, est désormais encadré par une fondation qui est (à priori...) la plus indépendante possible d'autres organismes et acteurs majeurs du milieu informatique. Mais tout en bénéficiant de leur soutien : le projet n'est donc pas mené de manière isolée, et il ne sera pas le fruit de la volonté d'une unique entité  pour servir ses intérêts.

**L'OCI cherche à faciliter la transition vers le monde des conteneurs, et surtout, à en pérenniser l'utilisation.**


# How do containers work ?

Nous allons voir dans cette partie (purement technique) certaines des technologies qui rendent la conteneurisation telle qu'on la connaît possible.
Cette partie n'est qu'une partie théorique et technique et n'est pas directement rattachée à la sécurité, mais est strictement indispensable pour discuter de problématiques de sécurité autour des conteneurs.

**A noter que la plupart de ces éléments sont ENFIN présents dans la documentation Docker officielle. Enfin.**

## Technologies noyau
Les `cgroups` et `namespaces` sont deux technologies kernel qui sont au centre de la conteneurisation.
Nous n'allons pas rentrer ici dans les détails, mais nous allons malgré tout présenter le fonctionnement de ces éléments pour comprendre sur quoi repose véritablement l'isolation qu'apportent la conteneurisation de type Docker. **En effet, c'est un aspect essentiel de la sécurité de Docker**.

### cgroups

**Les `cgroups` permettent d'allouer des quantités de ressources à des groupes de processus** (ses fonctionnalités sont en réalité plus larges, mais nous n'en discuterons que très peu ici). Un `cgroup` est un groupe de processus qui sont soumis à des rgèles appelées "contrôleurs".
Il existe un certain nombre de contrôleurs qui peut dépendre en fonction de la distribution (on en trouve 11 généralement). En voici quelques-uns (les principaux) afin d'avoir une idée de leur utilité :
* ***cpuset*** : permet d'allouer un coeur processeur à un groupe de tâche
* ***blkio*** : permet de limiter les actions de lecture ET d'écriture sur un périphérique de type bloc
* ***net_prio*** : permet de prioriser le trafic de certaines interfaces réseau vis-à-vis d'autres

*`man cgroups` pour une liste complète`.

Par défaut, à chaque conteneur lancé est créé un `cgroup` correspondant.

*A noter que la version 2 des `cgroups` n'a pas encore été adoptée, pour des problèmes de compatibilité. Voir [ici](https://github.com/opencontainers/runc/issues/654) et [ici](https://github.com/moby/moby/issues/16238)*.

### namespaces
**Les `namespaces` (ou ns) permettent d'isoler de manière effective des ressources.** Aux yeux des `namespaces` (et donc, du noyau), il existe un nombre limité de "ressources". La plupart ont une structure arborescente. Il en existe 6 :
* ***réseau*** (ou network stack)
  * carte réseau, interfaces, configuration, etc.
  * `ip address` ou `ifconfig` permettent de récupérer des informations du namespace courant
* ***noms d'hôtes et de domaine*** (ou UTS)
* ***points de montages*** (ou mnt)
  * points de montages, fs, etc
  * `df` retourne des informations du `ns` courant
* ***communication entre les processus*** (ou IPC)
  * ici on pense aux différents éléments de type pipes, socket, RAM partagée, queues, bus (notamment `d-bus`), etc
* ***utilisateurs*** (ou user)
  * gère users & groups
  * n'est pas utilisé par Docker dans la configuration par défaut
  * **par défaut, il n'est pas utilisé par Docker : son implémentation n'est qu'à un stade expérimental dans les kernels récents, au moment de l'écriture de ce rapport**. Il est possible de les utiliser malgré tout en activant cette fonctionnalité kernel au boot, voir [le paragraphe dédié](https://github.com/It4lik/markdownResources/tree/master/dockerSecurity#-user-namespace-remapping) pour plus d'informations.
* ***processus*** (ou plus exactement, PID)
  * arborescence de processus
  * `ps` va retourne des informations du `ns` PID courant
* ***cgroup***
  * on parle bel et bien ici d'un `namespace` de type cgroup
  * permet de lier un processus à des `cgroups`, isolés de leurs homologues

Le noyau est par exemple capable de gérer deux arborescences de processus totalement indépendantes.

### capabilities
**Les capabilities Linux sont les droits avec lesquels un processus est lancé.** `root` n'est pas un utilisateur magique. `root` n'a pas tous les droits. `root` est simplement un utilisateur qui a le pouvoir de lancer n'importe quel processus avec n'importe quelle capability.
On en en trouve un certain nombre (dépend du système, on en trouve souvent + de 35). Parmi lesquelles on a par exemple :
* ***CAP_SYS_PTRACE*** : permet d'utiliser ```ptrace``` et ```kcmp```
* ***CAP_AUDIT_WRITE*** : permet d'écrire dans les logs kernel
* ***CAP_DAC_OVERRIDE*** : permet de passer outre les permissions de lecture/écriture/exécution
* ***CAP_NET_ADMIN*** : permet de modifier les tables de routage, les règles de proxying, la possibilité de multicast, etc.
* ***CAP_NET_BIND_SERVICE*** : permet d'écouter sur un port en dessous de 1024

On reconnaît ici les superpouvoirs de `root`. Il est possible de visualiser les capabilities de chacun des processus lancés sur le système avec le binaire ```lscap``` rarement présent par défaut (quelque soit la distrib).

A l'instar de n'importe quel autre processus du sysème, un conteneur se voit attribuer un certain nombre de capabilities. Il est possible lors du lancement d'un conteneur (`docker run`) d'en ajouter (`--cap-add`) ou d'en supprimer (`--cap-drop`).

Une bonne façon de procéder lors du lancement d'un conteneur est de supprimer toutes les capabilities puis d'ajouter uniquement celles qui sont nécessaires :   
```shell
$ docker run --cap-drop ALL --cap-add SYS_TIME --name whattimeisit alpine /bin/sh
```

Les capabilities s'appliquent aussi aux binaires. On les manipule alors avec `setcap` et `getcap`. L'utilisation d'un setuid root est parfaitement équivalent à donner TOUTES les capabilities à ce binaire à l'aide de `setcap`.  
 Une autre façon de le dire est que `setcap` permet de donner de façon granulaire les capabilities à un binaire.

```shell
$ getcap /usr/bin/ping
/usr/bin/ping = cap_net_raw+ep
```

Extrait de la [documentation RedHat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/docker_selinux_security_policy) : *"Capabilites constitute the heart of the isolation of containers. If you have used capabilites in the manner described in this guide, an attacker who does not have a kernel exploit will be able to do nothing even if they have root on your system."*

### netfilter
Il existe un **framework kernel permettant de lui donner la fonction de firewall : c'est netfilter.**
Par défaut, le démon Docker utilise grandement les fonctionnalités kernel de ```netfilter``` afin de fournir du réseau aux différents conteneurs dont il est responsable. Il est évidemment possible de les visualiser à l'aide de ```iptables``` (ou ```ebtables```). Afin de pouvoir gérer le trafic, le démon crée une interface virtuelle pour chaque "réseau Docker" (on peut les visualiser avec ```docker network ls```).
Le majeure partie des règles permettent :
* de forwarder du traffic vers un conteneur spécifique
* de fermer certains ports

### Rentrons un peu dans le détail...
#### > Théorie : /proc et /sys
Avant de se pencher sur les différents drivers réseau et les différents cas d'utilisation auxquels ils répondent, par exemple, il est nécessaire de discuter un peu du fonctionnement du réseau pour un conteneur, ou en fait, plus globalement, pour une machine GNU/Linux.

Comme on le répète souvent pour Linux : tout est fichier ou processus. En l'occurrence, comme **la totalité des fonctionnalités élémentaires d'un système moderne** -des fonctionnalités comme le réseau- **sont gérées dans le kernelspace**. Des appels systèmes sont bien entendus à disposition pour pouvoir les manipuler. Mais en explorant les deux pseudo-filesystems que sont **/proc** et **/sys**, on peut obtenir énormément d'informations (en réalité, tout est là, donc c'est ici que l'on pourra obtenir le plus d'informations sur le réseau. Les commandes comme ``ip`` ou ``ifconfig`` ou des fichiers comme ``/etc/hostname`` ne font que piocher à un moment ou un autre dans ces données).

On a par exemple ``/sys/class/net`` qui contient un sous-répertoire par interface réseau de la machine (à l'intérieur se trouve un grand nombre de paramètres, contenus dans des fichiers).   
Dans le cas de Docker, avec le driver réseau de base, chaque réseau Docker est en réalité une nouvelle interface bridge. Chaque conteneur créera alors une sous-interface de ce bridge, qui apparaîtra comme un sous-répertoire supplémentaire. Evidemment, il est impossible depuis le conteneur de voir d'autres interfaces que les siennes. Ceci, grâce aux namespaces.


Pour observer cela, rendez-vous dans ``/proc/``. On y trouve énormément d'informations mais principalement deux groupes d'entités :
* des fichiers de configuration ou d'état du kernel. Par exemple :
  * ``/proc/cgroups`` affiche l'état des `cgroups` sur la machine
  * ``/proc/sys/vm/swappiness`` détermine un pourcentage à partir duquel la machine va commence à utiliser la swap en lieu et place de la RAM
* un répertoire pour chacun des processus en cours d'exécution sur la machine, portant l'ID du processus pour nom
  * dans chacun de ces répertoire se trouve un sous-répertoire ``ns`` qui contient autant d'entrées que le processus a de `namespaces` (au max, 1 de chaque type)
  * concrètement on peut, par exemple, avoir :

  ```
  $ ls /proc/16392/ns/
  cgroup	ipc  mnt  net  pid  uts
  ```

Chacun de ces fichiers est un lien symbolique (pas réellement, mais il se comporte et se présente comme si c'était le cas) vers un namespace spécifique. Par exemple :
```
$ sudo ls  -al /proc/16392/ns/pid
lrwxrwxrwx 1 root root 0 16 mai   14:22 /proc/16392/ns/pid -> pid:[4026531836]
```

**Chacun des processus peut être lancé dans un namespace différent. Ce qui l'isole des autres processus**, s'ils se trouvent dans d'autres `namespaces`.

Avec les éléments qu'on vient de voir, il devient apparent qu'on peut créer à la volée de nouvelles interfaces, et qu'il sera possible de totalement isoler le processus qui les utilisera.

**TO MOVE :** Il peut être nécessaire de prendre connaissance de **la [CNI](https://github.com/containernetworking/cni) : un standard visant à décrire comment constuire les interfaces réseau pour des conteneurs.**

#### > Explorer les namespaces : `nsenter`

Il est possible grâce à ``nsenter`` de rentrer littéralement dans un namespace et d'y exécuter du code. Il est par exemple possible d'exécuter un simple ``ps -ef`` en utilisant un namespace PID totalement isolé. Nous ne verrions alors que les processus de ce namespace.

Un cas d'utilisation de ``nsenter`` qui est très récurrent lors de la construction de conteneurs est l'utilisation de ``docker exec``. Cette commande exécute une commande dans un conteneur. Or, **un conteneur n'est qu'un processus, auquel on a attribué certaines capabilities, lancé par un utilisateur UNIX du système**, (`root` par défaut), **qui utilise des `namespaces` isolés et dont l'accès aux ressources est régi par les `cgroups`.** Donc en réalité, ``docker exec`` exécute un ``nsenter`` avec des options préconfigurées.

Petite expérience pour démontrer tout ça :
```shell
# Lancement d'un conteneur Alpine qui attend
$ docker run -d alpine sleep 99999
f9ddf5ea51f11d46478aa4265f921c069180c2ac3251713640d628367ae5ba0a

# Récupération de son PID
$ docker inspect $(docker ps -lq) -f '{{ .State.Pid }}'
19307
$ ps -ef | grep $(!!)
ps -ef | grep $(docker inspect $(docker ps -lq) -f '{{ .State.Pid }}')
root     19307 19290  0 14:45 ?        00:00:00 sleep 999999

# 19307 est le PID de sleep. 19290 est le PID du conteneur :
$ ps -ef | grep 19290
root     19290   704  0 14:45 ?        00:00:00 docker-containerd-shim f9ddf5ea51f11d46478aa4265f921c069180c2ac3251713640d628367ae5ba0a /var/run/docker/libcontainerd/f9ddf5ea51f11d46478aa4265f921c069180c2ac3251713640d628367ae5ba0a docker-runc

# Maintenant, nous voulons exécuter un ps -ef dans le conteneur. Deux possibilités : nsenter ou docker exec. Nous allons ici montrer qu'ils sont équivalents
$ docker exec $(docker ps -lq) ps -ef
PID   USER     TIME   COMMAND
    1 root       0:00 sleep 999999
   35 root       0:00 ps -ef
$ sudo nsenter -t $(docker inspect --format '{{ .State.Pid }}' $(docker ps -lq)) -m -u -i -n -p -w /bin/ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 12:45 ?        00:00:00 sleep 999999
root        73     0  0 13:03 ?        00:00:00 /bin/ps -ef
```

**On voit que l'isolation de l'arborescence de PIDs est bien fournie par les namespaces.** Et qu'elle est efficace : on ne voit que les processus du conteneur en question. Pas ceux d'éventuels autres conteneurs ou de l'hôte.

Il en va de même pour tous les autres namespaces. C'est plutôt fun/intéressant de se balader dans /proc et /sys de toute façon. Et il est parfaitement possible de créer un conteneur à la main à base de syscalls `clone()` ou `unshare()`.

#### > Création de namespace : `unshare`

# Questions récurrentes
* **Est-ce qu'un conteneur est moins secure qu'une VM ?**
Comme dit plus haut, ce n'est simplement pas comparable. Les deux reposent sur des concepts fondamentalement différents. De plus, dans la quasi-totalité des cas, l'hôte de conteneurisation est une VM. La conteneurisation n'est donc simplement qu'un niveau d'isolation supplémentaire.   


* **Si un attaquant exploite une vulnérabilité exposée par un conteneur, a-t-il une main-mise totale sur la machine sous-jacente ?**
Cela dépend complètement du type de vulnérabilité.
De façon générale, ça n'expose ni plus ni moins le système qu'avec un applicatif "classique" (lancé via un binaire, ou service comme systemd, directement depuis le système hôte (eg. directement dans les `namespaces` "principaux" de l'hôte)).
En effet, une vulnérabilité kernel qui affecte un conteneur rendra l'hôte tout aussi vulnérable car les deux partagent **"physiquement"** le même kernel.
A l'inverse, une vulnérabilité applicative (par exemple, vulnérabilité dans la version d'Apache utilisée) n'exposera pas plus le système qu'avant. Voire moins, ceci dépend de la configuration mise en place (démon, image, conteneur, cf les parties dédiées, plus bas).
Au mieux, l'attaquant sera strictement confiné dans le conteneur. Au pire, il aura une totale main-mise sur le système sous-jacent (qui, pour rappel, est une VM...). A mi-chemin, il sera capable de se rendre compte qu'il est dans un conteneur, et éventuellement obtenir des informations sur le système hôte.    

# Sécurité et bonnes pratiques Docker
## Sécurité du démon Docker
### Exposition de l'API
L'API peut être exposée *via* deux technologies :
* Par défaut, c'est un **socket UNIX** qui est utilisé (il se trouve à ```/var/run/docker.sock```). La communication est donc interne au système. La surface d'exposition est extrêmement faible, c'est le moyen à privilégier.
* Utilisation d'un **socket TCP**. Il est possible par ce biais de contrôler un démon Docker à distance, auquel cas il est très **fortement recommandé d'utiliser un tunnel de chiffrement à l'aide du protocole TLS.** Il est aussi impératif de ne rendre possible l'accès à ce service qu'avec des règles réseau très restrictives (LAN isolé, accès à ce LAN restrictif, etc.). **De façon générale, l'exposition sur un socket TCP est tout simplement déconseillée**.
### Options du démon Docker
Le démon docker (généralement géré à l'aide de ``systemd``) est lançable manuellement avec le binaire ``dockerd``. De nombreuses options sont à notre disposition, parmis lesquelles :
* ``--icc`` : permet d'activer la communication inter-conteneur. **``true`` par défaut.**
* ``--ip`` : permet de choisir l'adresse de l'hôte utilisée pour les forwardings de port. **Par défaut : 0.0.0.0**.
* ``--user`` : permet de choisir quel utilisateur lancera les conteneurs. **Par défaut, c'est root**.
* ``--userns`` : permet de choisir arbitrairement quel `user namespace` sera utilisé par le conteneur. Par défaut, un nouveau est créé à chaque `docker run`.


Ces quelques options peuvent suffire à faire peur. Et leur paramétrage par défaut est caractéristique de "this is a feature, not a vuln". Ce ne serait que pure vulnérabilité si on ne pouvait pas modifier nous-mêmes ces paramètres.  
Et je pense qu'ils ont raison de considérer ça comme une "feature". La sécurité par défaut **pourrait** en effet être renforcée. Mais une configuration très restrictive de cet outil, réalisée arbitrairement par les développeurs de Docker, engendrerait nécessairement un gros travail de configuration à tous les utilisateurs. Ce n'est pas un problème en soit, mais le travail serait effectivement énorme.   
Or, dans beaucoup de cas on utilise Docker à des fins de tests, sur notre propre ordinateur. **Dans la plupart des cas, on attend juste de Docker qu'il fonctionne.** Ces choix sont donc compréhensibles. Pour une utilisation en production, tous ces éléments restent donc bien évidemment paramétrables.

J'étais moi-même sceptique au départ, mais d'autres exemples viennent contrecarrer le sceptiscisme. Par exemple, SELinux est par défaut quasiment non configuré et trop restrictif sur les machines de type RedHat. Dans beaucoup de cas, les gens le désactivent simplement (ou le laissent afficher des warnings) alors qu'il constitue une brique extrêment robuste de la sécurité des systèmes RedHat. Résultat ? Il est inutilisé, bien qu'extrêmement puissant.
Docker a fait le choix de désactiver ces options, pour satisfaire le plus grand nombre. Et il est possible de se le permettre car le public visé va, à 95% faire tourner Docker sur un ordinateur personnel (sans l'exposer vers l'extérieur, et n'attendant de Docker qu'une unique chose : qu'il fonctionne).

**Si ces sécurités sont désactivées par défaut, c'est effectivement une feature. L'introduction de cette feature est la conséquence du public visé par Docker.**

## Sécurité des hôtes Docker
**Nous parlons ici des hôtes Docker "standalone"** : les hôtes qui ne sont PAS membres d'un quelconque framework d'orchestration (un point y sera dédié).
De fait les hôtes Docker sont très souvent des machines virtuelles.  
De plus, il est recommandé d'en utiliser pour bénéficier de tous les avantages qu'elles apportent (sur lesquels nous ne nous attarderons pas ici). Etant une machine virtuelle, l'hôte Docker se soumet aux règles habituelles concernant les machines virtuelles. Nous n'en allons pas en dresser une liste exhaustive, mais voici certaines des grandes lignes :
* **Tout hôte faisant tourner Docker, ne fait tourner que Docker.**
  * à l'image d'un serveur Web qui n'a qu'un serveur web (et une base de données, éventuellement)
  * on peut imaginer avoir les sondes de monitoring ou autres qui subsistent encore sur le système de l'hôte.
* **On ne mélange pas des services qui ont différents niveaux de sensibilité, différentes surfaces d'attaque, etc.**
  * un hôte fait tourner une unique application (souvent n-tiers) conteneurisée
  * tous les hôtes dans le même sous-réseau exposent des services de sensibilité équivalente
* **Les hôtes possèdent un utilisateur UNIX dédié au lancement de conteneur avec une configuraton extrêmement restrictive**
* On peut imaginer un tas d'autres mesures visant à augmenter le niveau de sécurité d'un hôte Docker :
  * customiser les `cgroups` du système afin de créer les `cgroups` parents de tous les `cgroups` enfants utilisés par les conteneurs
  * dédier une interface réseau au forwarding de port vers des conteneurs, tandis qu'une autre est utilisée pour le reste (joindre l'extérieur, administration, backup, monitoring, etc.)
  * etc.

En somme... Ce sont exactement les mêmes mesures que d'habitude. Et c'est normal : Docker n'apporte rien de magique, rien de supplémentaire. **Il change juste certains usages, en facilite certains, en crée d'autres, mais rien n'a été *inventé* pour Docker**. Par conséquent les habituelles bonnes pratiques s'appliquent à ce service.

Et la quasi-totalité de celle-ci sont la résultante d'un principe simple mais malgré tout central : **la politique du moindre privilège**.

## Sécurité des images Docker
### Qu'est ce qu'une image Docker ?
Les images Docker sont, par défaut, stockées dans le répertoire ``/var/lib/docker/images``. Ce sont des union filesystems (certains fs y sont dédiés (`AUFS`, `overlayFS`), d'autres systèmes de fichiers plus répandus supportent cette fonctionnalité (`zfs`, `btrfs`, `devicemapper`)).

Elles sont le résultat de la commande ``docker build`` sur un Dockerfile. Il est impossible de remonter au Dockerfile depuis l'image (on peut obtenir des informations, mais pas dans l'état original tel qu'énoncé dans le Dockerfile, ni avec autant de clarté).  
**Autrement dit, on peut juger du contenu d'une image en jugeant le contenu du Dockerfile.**

Un Dockerfile contient **obligatoirement** l'instruction ``FROM`` en tout début de fichier (il est lu de haut vers le bas lors du build). C'est l'image de départ, sur laquelle nous allons rajouter de la configuration. Il est possible de partir d'une image existante, ou d'une pseudo-image qui porte bien son nom : ``scratch`` (le Dockerfile commence donc par ``FROM scratch``. Pas besoin d'explications supplémentaires).

### Le *Registre Docker*
Un applicatif, le *Registre Docker*, permet d'héberger des images. On les récupère avec ``docker pull`` et on les envoie sur le *Registre Docker* avec ``docker push``.

Il existe un *Registre Docker* public, appelé [Docker Hub](https://hub.docker.com/), il est le *Registre Docker* ciblé par défaut par les commandes ``docker push`` et ``docker pull``.

On y trouve des dépôts de particuliers, des dépôts d'entreprises (microsoft, vmware, etc) et un dépôt particulier : [le dépôt library](https://hub.docker.com/explore/) (qui contient les "official repositories"). Le dépôt *library* contient des images qui sont vérifiées par la société Docker et qui sont directement émises par les éditeurs des solutions en question. On y trouve un certain nombre de services très communs comme alpine, busybox, apache, mysql, redis, nginx ou encore mongodb.

En résumé, les images dignes de confiance sont :
* les images contenues dans le dépôt library du Docker Hub
* les images développées en interne, basées sur des images de library
* les images développées en interne, ``FROM scratch``
* les images basées sur des images développées en interne

On peut éventuellement accorder dde la confiance à d'autres éditeurs, dont les dépôts ne sont pas dans *library* (par exemple, [vmware](https://hub.docker.com/u/vmware/)).

### Construction d'images

## Sécurité des conteneurs Docker
Ici, on fait clairement référence au lancement **d'un** conteneur, à partir d'une image, à l'aide de la commande ``docker run``.
### Les options
La commande ``docker run`` possède d'innombrables options. Seule une infime partie est utilisée au quotidien, des choses comme :
* ``-p`` : permet de forwarder un port de l'hôte vers un conteneur
* ``--hostname`` : permet de spécifier arbitrairement un nom d'hôte pour la machine
* ``-v`` : permet d'utiliser des volumes Docker

Cependant on trouve aussi un grand nombre d'options auxquelles il est **nécessaire** de s'intéresse afin d'appréhender la sécurité d'un conteneur. Parmi celles-ci, on a :
* ``--cap-drop`` qui permet de supprimer des capabilities au conteneur lancé
  * pour rappel : de chaque conteneur lancé avec ``docker run`` résultera un **unique** processus. C'est donc ce processus qui se verra supprimer des capabilities.
* politiques de restriction d'accès aux ressources à l'aide des cgroups. Beaucoup d'options en relation avec ce point :
  * ``blkio-weight`` : priorisation des accès disque, possibilité de désactiver l'écriture sur le disque avec une valeur nulle
  * ``--cpus-quota`` et ``--cpu-period`` permettent d'agir sur le CFS du processeur
  * ``--memory`` : limitation de la quantité de RAM utilisée
  * ``--tmps`` : monte un système de fichier temporaire au path indiqué
  * ``--pids-limit`` : permet de limiter le nombre d'ID de processus disponible

Avec ces quelques options, on voit qu'il est clairement possible de créer un environnement extrêmement restreint avec des règles très granulaire. Et la liste est loin d'être exhaustive.

### Gestion des utilisateurs
On trouve sur internet pas mal de confusion autour de ce sujet. Let's clarify.
#### > Communication avec le démon Docker
Le premier utilisateur auquel nous sommes confrontés est celui qui est autorisé à taper sur le socket UNIX, situé par défaut à `/var/run/docker.sock`. Par défaut aussi, seul `root` est autorisé à communiquer avec le démon Docker *via* ce socket dédié.  
* Il est possible d'ajouter des utilisateurs au groupe `docker` afin qu'ils puissent directement utiliser le socket, sans avoir besoin des droits de `root` (en utilisant `sudo` par exemple).   
**Sans configuration supplémentaire, un tel utilisateur est tout simplement root sur la machine**. C'est parfaitement équivalent à un `ALL=(ALL)   NOPASSWD: ALL` dans le fichier `sudoers`...   
En effet, il lui suffira de monter le répertoire racine de l'hôte dans le conteneur afin d'avoir, par exemple, accès à tout le système de fichiers de l'hôte en lecture/écriture.  
* Une des façons de procéder pour protéger le socket est d'utiliser `sudo` avec une configuration restrictive.  
Premièrement, il est impératif de déterminer quelles commandes `docker run` vos utilisateurs pourront passer. Dans l'exemple suivant, du fait de l'anatomie de la commande `docker run` (`docker run [OPTIONS] <image> [ENTRYPOINT]`), il est impossible de rajouter des arguments à cette commande :
```shell
$ cat /usr/bin/restricted-alpine-docker
docker run -it --rm alpine sh

$ grep user1 /etc/sudoers
user1        ALL=(ALL)       NOPASSWD: /usr/bin/restricted-alpine-docker
```
Malheureusement, dans ce cas de figure, l'utilisateur n'a quasiment aucune liberté (impossibilité de changer d'image ou d'exposer un port).
* **L'unique solution valable** est d'utiliser un plugin d'authentification. Ceux-ci permettent en effet à chaque interrogation du socket de vérifier l'identité de l'utilisateur, et surtout, de vérifier qu'il a le droit d'exécuter l'action demandée. L'un des plus connus est [authz](https://github.com/twistlock/authz).

De façon générale, **il est impératif de n'autoriser que les utilisateurs de confiance à utiliser le socket où Docker écoute**, que ce soit socket UNIX ou TCP. **La meilleure façon de réduire la surface d'attaque du socket est d'utiliser un plugin d'authentification externe.**

#### > Lancement des processus
Ici, on parle de l'utilisateur qui lancera **effectivement, sur l'hôte** les processus initiés par les conteneurs.  

Il est important de comprendre qu'un conteneur n'est qu'un niveau d'isolation géré par le kernel. De ce fait, tout processus lancé dans un conteneur est visible sur l'hôte. Voyons un exemple :
```shell
[root@docker-host ~]$ docker run -d alpine sleep 40
a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05

[root@docker-host ~]$ ps -ef | grep a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05
root     11649  3480  0 09:32 ?        00:00:00 docker-containerd-shim a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05 /var/run/docker/libcontainerd/a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05 docker-runc
root     11683 10772  0 09:32 pts/1    00:00:00 grep --color=auto a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05

[root@docker-host ~]$ ps -ef | grep 11649
root     11649  3480  0 09:32 ?        00:00:00 docker-containerd-shim a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05 /var/run/docker/libcontainerd/a28e3d8da77d590da7073626c9822d127e9e3cbbf61d985db39a1b1e8a81df05 docker-runc
root     11660 11649  0 09:32 ?        00:00:00 sleep 40
root     11689 10772  0 09:32 pts/1    00:00:00 grep --color=auto 11649
```
Le processus `sleep` est parfaitement visible depuis l'hôte. Et on constate qu'**il est lancé avec `root` sur l'hôte**. Tout autre processus lancé par ce conteneur apparaîtra comme un processus de `root` en dehors du conteneur.  
Il est possible de modifier ce comportement à l'aide de l'option `--user` (cette option prend en argument l'ID de l'utilisateur voulu) :
```shell
[root@docker-host ~]$ docker run -d --user 1000 alpine sleep 40
dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26

[root@docker-host ~]$ ps -ef | grep dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26
root     11902  3480  0 09:35 ?        00:00:00 docker-containerd-shim dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26 /var/run/docker/libcontainerd/dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26 docker-runc
root     11934 10772  0 09:36 pts/1    00:00:00 grep --color=auto dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26

[root@docker-host ~]$ ps -ef | grep 11902
root     11902  3480  0 09:35 ?        00:00:00 docker-containerd-shim dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26 /var/run/docker/libcontainerd/dcb38179b7d7e34477c53eed602e65018796f42a9a4f6b3509265e05a8335f26 docker-runc
it4      11913 11902  0 09:35 ?        00:00:00 sleep 40
root     11936 10772  0 09:36 pts/1    00:00:00 grep --color=auto 11902
```
**Ici, on voit clairement que le processus `sleep` est désormais lancé par l'utilisateur `it4` sur l'hôte**. En revanche, le conteneur lui-même est toujours lancé par `root` (c'est la ligne avec le processus `docker-containerd-shim`).

De plus, à l'intérieur du conteneur, c'est aussi l'`uid` de `it4` qui est utilisé. Sauf que, sans configuration contraire, `it4` n'existe pas dans le conteneur. On se retrouve alors avec un `uid` inconnu :
```shell
[root@docker-host ~]$ docker exec $(docker ps -lq) ps -ef
PID   USER     TIME   COMMAND
    1 1000       0:00 sleep 9999
    7 1000       0:00 ps -ef
```
#### > User namespace remapping
Il est possible de positionner **sur le démon** l'option `--userns-remap` (binaire `dockerd`, on le modifie dans l'unité de service systemd).   
Celle-ci va permettre d'exploiter les mécanismes de [`subuid`](http://man7.org/linux/man-pages/man5/subuid.5.html) et [`subgid`](http://man7.org/linux/man-pages/man5/subuid.5.html). Ceux-ci sont configurables dans des fichiers définissant quel utilisateur a le droit de manipuler quel autre utilisateur, ou plutôt quels autres `uid` et `gid` que les siens.   
Ainsi, il est possible de définir des plages d'`uid` et `gid` -dans leurs fichiers respectifs- afin de définir les IDs qui pourront être utilisés par ces nouveaux utilisateurs. On utilise souvent l'argument `default` à cette option, qui a pour effet d'utiliser un utilisateur et un groupe qui portent le nom de `dockremap` (le changer est purement cosmétique).  


**Exemple : un utilisateur `root` dans le conteneur qui correspond à un autre utilisateur sur l'hôte.**  

![](https://github.com/It4lik/markdownResources/blob/master/dockerSecurity/pics/mitigating-attack-surface-with-usernamespaces-remapping.gif)


Avec une telle configuration, vous pouvez même essayer de `-v /:/host`, l'utilisateur `root` est strictement impuissant.  

**NB1: il vous faudra activer le support du `namespace` de type `user` pour que ceci puisse fonctionner.** Sur les systèmes RHEL7/CentOS7, ce n'est qu'une feature en preview. Pour vérifier son activation c'est `/proc/cmdline`, l'option `user_namespace.enable=1` doit être positionné. Le cas échéant, reportez-vous [ici](https://github.com/procszoo/procszoo/wiki/How-to-enable-%22user%22-namespace-in-RHEL7-and-CentOS7%3F) pour plus d'informations.

**NB2: A l'heure de l'écriture de cet article, il peut s'apparenter à un cauchemar** -pour les utilisateurs non-familiers avec cette ribambelle de technos- **de faire fonctionner cette configuration de concert avec SELinux d'activé**. Pour les utilisateurs de RHEL, reportez-vous notamment à [ce ticket](https://github.com/opencontainers/runc/pull/959) qui explique que le support complet ne sera pas apporté avant la version 7.4.

**Important** : ceci est plus qu'un simple mapping d'utilisateur. En effet, à chaque user namespace son jeu d'utilisateurs, certes, mais aussi **son jeu de capabilities**. Les capabilities ont pour scope un user namespace (c'est d'ailleurs pour cette raison que l'implémentation du user namespace est bien plus complexe que les autres).   
Enfin, il est impératif de garder à l'esprit que les namespaces de type user sont comme une structure arborescente, et que le root user namespace (*e.g.* le user namespace par défaut, celui dans lequel on se trouve tout le temps) a une visibilité totale sur les autres.


#### > Utilisateurs applicatifs ?

Enfin, nous parlerons ici de la création d'un utilisateur **à l'intérieur du conteneur**, afin de l'utiliser pour faire tourner nos services. En somme, c'est la politique habituelle, celle quifait utiliser `www-data` pour faire tourner le serveur web Apache. Pour les conteneurs, c'est une question discutable...   

Il est certain qu'avec une bonne configuration (en particulier du kernel, avec seccomp, SELinux, etc) et éventuellement un remap de l'utilisateur qui lance les conteneurs (`--userns-remap` en option de `dockerd`), un utilisateur applicatif n'a **aucune** utilité en soi. En effet, dans l'exemple qui suit, il est apparaît clairement que SELinux **seul** peut empêcher un utilisateur `root` dans un conteneur de prendre le contrôle de l'hôte, même en ayant accès à tout le filesystem, et même si les processus Docker sont lancés avec `root` sur l'hôte :

![](https://github.com/It4lik/markdownResources/blob/master/dockerSecurity/pics/mitigating-attack-surface-with-SELinux.gif)

Nous ne parlons pas dans ce passage de plus grandes restrictions avec d'autres technologies (comme `seccomp`), mais il apparaît clait qu'avec une configuration robuste, les utilisateurs applicatifs dans les conteneurs sont inutiles.

**Cependant**, on voit régulièrement les outils puissants mais complexes comme `SElinux` demeurer inutilisés. Ainsi, on préférera tout de même créer des utilisateurs applicatifs dans nos conteneurs. Cela ajoute une couche de compléxité, mais aussi de sécurité. *Disons que ça ne mange pas de pain...*




## Discussion autour des systèmes d'orchestration de conteneurs
### Plus grande exposition des vulnérabilités
En utilisant un système d'orchestration de conteneurs, il est récurrent d'utiliser des politiques de HA et de redémarrage des services.
Il sera par exemple possible de demander au système de remonter un conteneur qui serait amené à être indisponible. Les raisons de cette indisponibilité peuvent se révéler très diverses : lui même a coupé (dépassement de ressources, bug, etc), l'hôte s'est coupé, etc.  
Dans le cas où l'hôte se coupe, on voit souvent des politiques qui visent à reprogrammer le conteneur sur un autre noeud.   
Si ce conteneur expose un service vulnérable, et répresente donc une faille de sécurité, qui peut amener un attaquant à pénétrer dans le système hôte, alors toutes les machines du cluster pourront potentiellement être compromises (une vulnérabilité kernel pourrait remplir ces conditions).

Il est donc important de garder à l'esprit que toutes les machines membres d'un cluster d'orchestration sont à considérer de manière équivalente en terme de sécurité. Si l'une est compromise il est à parier qu'une bonne partie du reste des machines le soit aussi.

**Même** (surtout ?) **avec un framework d'orchestration, il reste impératif de cloisônner au moins en terme de réseau les différentes applications & les différents clusters.**

## Le réseau dans Docker

### Lonely host
[Bonne ressource](http://hicu.be/category/networking) concernant le réseau sur un hôte unique, expliqué de façon simplifiée.
#### Driver par défaut : bridge
Par défaut, les réseaux créés dans Docker sont un assemblement de plusieurs technos permettant de simuler l'existence d'un réseau privé pour lequel notre hôte agirait comme un switch. Afin de simuler un switch, sont créées des bridges UNIX, une sous-interface par conteneur, des règles de routages et des règles ``iptables``.  

Ce n'est cependant ni en terme de robustesse, ni de fonctionnalités, ni même de rapidité le meilleur choix à faire en ce qui concerne le driver réseau de Docker.

#### MACVLAN et IPVLAN
Bien que différents, nous allons aggréger ces deux types de réseaux en un seul point pour une raison simple : ils répondent parfaitement aux besoins d'une VM ou d'un conteneur en terme de connectivité.
MACVLAN agit comme un switch (équipement L2) tandis que IPVLAN agit comme un routeur (L3).

Nous ne rentrerons pas dans les détails techniques ou dans des études de benchmarking, ces éléments foisonnant sur le web. Simplement à titre indicatif : il est souvent très fortement conseillé d'abandonner le driver réseau de base au profit de MACVLAN et IPVLAN.

### Multi-host networking
Il existe plusieurs plugins réseau pour Docker permettant d'accéder à un multi-host networking. Par là nous considérons un cas avec n hôtes Docker, chacun possédant n conteneurs. Un conteneur X doit pouvoir être joint par un autre, et par lui-même sur une même adresse, quelques soit leur machine hôtes respectives.

#### Driver overlay natif
Le driver natif, simplement appelé *overlay* est fonctionnel, et a pour vocation d'être utilisé dans des cadres de production. Malheureusement, de nombreux benchmarks montrent qu'il n'est que très peu performant et a parfois été pointé du doigt à cause de certains bugs.  

Il peut correctement répondre à des besoins de POC ou de tests d'applications n-tiers conteneurisées, mais est déconseillé pour des usages nécessitant un meilleur niveau de qualité.

#### Autres drivers
Il y en a un certain nombre parmi lesquels : Calico, Flannel, Contiv, OpenVSwitch ou encore Weave. Chacun possède une équipe de développeurs derrière lui, et des objectifs différents. Certains, comme OpenVSwitch ont déjà fait leur preuve dans d'autres cas d'utilisation. La majorité n'a été conceptualisée que pour répondre aux nouveaux besoins suggérés par la conteneurisation (et surtout les systèmes d'orchestration qui y sont liés).

Nous allons ici parler d'une solution permettant de coupler Calico et Flannel : [Canal](https://github.com/projectcalico/canal). En couplant les deux, il est possible d'accéder à un grand niveau de qualité et de sécurité :
* très bonnes performances des overlay networks gérés par Flannel (notamment à l'aide de VXLAN)
* routing L3 avec Calico
* ACL granulaires sur chaque hôte (Calico)
*A noter que Calico est le driver réseau de référence conseillé par Kubernetes.*

## Le stockage et la sauvegarde avec Docker
### Des conteneurs et des données

Il y a deux types de données à distinguer au sein d'un conteneur :
- ce qui est immuable : le système, la configuration applicative, etc. Se trouve dans le Dockerfile. C'est la partie **"application et son environnement"**.
- ce qui ne l'est pas : **données applicatives** (comme des bases de données, ou autres structures de données). Se trouve dans des volumes. De la sorte, ils sont facilement accessibles depuis l'extérieur, sont indépendants des conteneurs et sont persistants.

Bien souvent, on n'utilise pas directement les Dockerfiles afin de déployer des applications conteneurisées mais plutôt un *Registre Docker* privé. Ce dernier contient les images correspondant aux Dockerfiles. La sécurité des images Docker (au sens disponibilité et intégrité) est donc relative à la sécurité des données du *Registre Docker* concerné.

Pour ce qui est de la donnée, non-immuable, elle est stockée dans des volumes Docker. Ces derniers utilisent un plugin de stockage spécifique permettant d'accéder directement à des cibles de stockage externes. Autrement dit, la sécurité de ces données est gérée au niveau de ces backends de stockage et non de Docker lui-même (se reporter à la section sur les plugins de stockage ci-dessous).

### Les volumes Docker
Les volumes sont des espaces de stockage, indépendants du union filesystem sur lequel reposent les conteneurs. Ils peuvent être montés dans des conteneurs afin de fournir un espace de stockage persistant à travers les reconstructions de conteneur, ou encore, ils peuvent servir à partager des données entre plusieurs conteneurs (en cas de multiples accès en écriture, la question des accès concurrents se pose alors, nous ne la traiterons cependant pas ici. )

Il est possible d'utiliser plusieurs types de volumes, pour cela, Docker propose plusieurs **plugins de stockage**, qui n'ont, pour la plupart, PAS été développé par la société Docker, mais par les acteurs concernés. En effet, la plupart des acteurs autour du stockage ont rapidement développé un plugin afin de permettre à leur solution d'être utilisée dans le cadre de déploiement à l'aide de Docker. Ainsi, sans être exhaustif, on trouve dans cette liste :
- des plugins Cloud
  - [REX-Ray](https://github.com/codedellemc/rexray) (Digital Ocean, Ceph, EC2, GCE persistent-disks, etc) développé par Dell-EMC
  - [Azure](https://github.com/Azure/azurefile-dockervolumedriver) (SMB 3.0), etc. développé par Microsoft
- des plugins constructeur
  - [NetApp](http://libstorage.readthedocs.io/en/stable/user-guide/storage-providers/#dell-emc-scaleio) développé par... NetApp :)
  - [vSphere](https://github.com/vmware/docker-volume-vsphere) permettant d'avoir un datastore (NFS, VMFS, etc) comme stockage direct. Développé par VMWare
- des plugins spécifiques à des solutions logicielles de stockage
  - [DRBD](https://www.linbit.com/en/persistent-and-replicated-docker-volumes-with-drbd9-and-drbd-manage/) permet d'utiliser des volumes pour Docker qui sont en réalité des *ressources DRBD*
- autres plugins
  - [Portworx](https://github.com/portworx/px-dev). Solution à part entière, déployée sous la forme de conteneurs, permet d'intégrer la gestion du stockage à la stack applicative. De façon simpliste, Portworx transforme une machine en un noeud de stockage (capacité de clusterisation, politique de réplication, etc.) permettant au conteneur d'utiliser les disques sous-jacents avec des perfs équivalentes à une machine physique *(en théorie)*.

A noter que dans l'ensemble des cas, il serait potentiellement possible d'utiliser l'espace de stockage voulu sur l'hôte puis de le monter localement grâce à la gestion native des volumes. Cependant, l'impact en terme de performances serait énorme, on préférera se connecter "directement" aux diverses cibles de stockage *via* un plugin dédié. D'autres problématiques sont soulevées en cas de non-utilisation des plugins, nous ne les traiterons donc pas ici car il est très largement préférable de les utiliser.

### Gestion des volumes au sein d'un framework d'orchestration
Les différents frameworks d'orchestration proposent chacun une façon de gérer les volumes utilisés par les conteneurs. En effet, dans le cas de l'utilisation d'orchestration, le problème est différent. On ne cherche plus seulement à rendre persistent le stockage à travers des redémarrages, mais aussi consistant à travers le temps et partagé entre les différents hôtes du cluster.

Nous ne nous attarderons pas ici sur ce sujet puisqu'il serait nécessaire de détailler énormément chacunes des solutions. Nous mentionnerons cependant le fait que Kubernetes est le framework d'orchestration de conteneurs le plus utilisé à ce jour, et il fournit lui aussi de nombreuses interfaces afin d'utiliser toutes sortes de backend de stockage, à l'instar des plugins de stockage Docker (voir plus haut).

Une des configurations particulièrement robuste concernant les problématiques de stockage pour Kubernetes est l'utilisation de [GlusterFS + Heketi](https://github.com/heketi/heketi/wiki/Kubernetes-Integration).

### La sauvegarde

Il n'existe donc pas de "sauvegarde de conteneurs". La sauvegarde doit se concentrer sur :
- le stockage utilisé par le *Registre Docker*
  - assurer la sécurité des images Docker
- les différentes cibles de stockage utilisés *directement* par les conteneurs pour les données applicatives
  - assurer la sécurité et la persistance des données applicatives

De ce constat, les problématiques de sauvegarde sont les mêmes qu'à l'accoutumée.


## Applicatifs permettant de renforcer la sécurité de Docker
Il existe de nombreux applciatifs permettant de renforcer la sécurité autour de Docker. De part leur grand nombre et surtout le caractère florissant de cet écosystème, nous n'en verrons que quelques-uns.

### Signatures des images : Notary
Notary n'est pas un outil qui se limite à une utilisation pour des conteneurs, mais peut être utilisé de concert avec n'importe quelle collection d'objets. Ici nous allons discuter de son utilisation dans le cadre de la signature des images d'un *Registre Docker*.

En effet il est possible d'utiliser Notary pour permettre aux utilisateurs d'un *Registre Docker* de :
- signer les images qu'ils poussent sur un dépôt
- vérifier la signature des images afin de s'assurer de l'intégrité et de la provenance de l'image à chaque `docker pull`

Cela dit, il sera nécessaire :
- pour les admins : d'appréhender le fonctionnement de Notary (GUNs, TUF framework, key management, etc.)
- pour les utilisateurs : d'apprendre à utilise le CLI Notary, le fonctionnement des GUNs, qui sont essentiels afin d'utiliser les fonctionnalités que la solution propose

Il est à noter qu'un Registre Docker couplé à Notary ne force pas son utilisation : on pourra y pousser/récupérer des images non signées (et donc utiliser le Registre Docker de façon strictement identique à l'habitude).
Il existe désormais [un aperçu](https://docs.docker.com/notary/getting_started/) de cette technologie dans la documentation officielle Docker.

### Scanner de vulnérabilités pour les images : Clair
[Clair](https://github.com/coreos/clair) est un outil développé par CoreOS permettant l'analyse statique de vulnérabilité dans des conteneurs. Son fonctionnement repose sur [des bases de données connues par la communauté et dignes de confiance](https://github.com/coreos/clair#what-data-sources-does-clair-currently-support) (publiées par RedHat, Oracle, Debian, etc) qui contiennent l'ensemble des vulnérabilités que Clair saura détecter.

Il existe deux moyens de s'en servir :
- manuellement : utilisation d'un outil comme [`clairctl`](https://github.com/jgsqware/clairctl) qui permet de manuellement soumettre à une instance de Clair des images afin de générer des rapports HTML sur les vulnérabilités contenues dans ces dernières
- automatiquement : couplée à un Registre Docker. En effet, de cette façon, il est possible d'agir directement au moment où une image est poussée. A chaque `push`, les images sont contrôlées, et des actions pourront être déclenchées en cas de résultat(s) positif(s).
  - le Registre Docker de CoreOS ([quay.io](https://quay.io)) le propose nativement
  - l'équipe VMWare qui travaille sur Harbor souhaite y intégrer Clair (à priori fonctionnel grâce à [ce PR](https://github.com/vmware/harbor/pull/2166), mais non intégré dans la documentation pour le moment, ni dans la dernière release de la solution)

### Systèmes de sécurité kernel
#### SELinux
[Tout est ici.](https://www.mankier.com/8/docker_selinux)

L'utilisation de SELinux couplée à Docker permet principalement de :
- isoler les conteneurs de l'hôte
- isoler les conteneurs les uns par rapport aux autres

Afin de pouvoir l'utiliser, il est nécessaire d'ajouter `--selinux-enabled` au lancement du démon. On peut voir s'il est activé ou non à l'aide de la commande `docker info | grep Security` qui possède la ligne suivante en cas de bonne activation :
```shell
Security Options: SELinux
```

En cas d'activation, aucune manipulation supplémentaire n'est nécessaire afin d'accéder à un bon niveau de sécurité. En effet, une simple activation permettra déjà de générer des [labels MCS](https://www.centos.org/docs/5/html/Deployment_Guide-en-US/sec-mcs-getstarted.html) uniques pour chacun des conteneurs.

#### AppArmor

A l'instar de SELinux pour les systèmes RedHat, AppArmor est un outil de MAC permettant d'augmenter le niveau de sécurité du système et des applicatifs, mais plutôt pour les systèmes Debian-based (on le voit surtout sur Ubuntu).

Par défaut, si AppArmor est activé sur la machine hôte, le démon n'est pas surveillé par AppArmor. En revanche, *dans des conditions normales*, les conteneurs le sont avec un profil par défaut : `docker-default`.

Il est possible d'appliquer un profil particulier à un conteneur lors du `docker run` à l'aide de l'option `--security-opt apparmor=profile_name`.

### Seccomp
`seccomp` est un utilitaire intégré au kernel Linux permettant de filtrer les appels système émis par un processus.

Il est possible, lors du lancement d'un conteneur de limiter les appels système que peut réaliser un conteneur grâce à `seccomp` et plus particulièrement à l'option `--security-opt` de la commande `docker run`. Cette option prend en entrée un fichier JSON qui répertorie les appels système non-autorisés.

Dans le cadre du projet Moby, [un fichier JSON de référence](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json) a été créé. Ce dernier bloque l'ensemble des appels système non nécessaire aux conteneurs tout maintenant une compatibilité applicative complète (n'est bloquant pour aucune application conteneurisée, à priori).

Pour que l'engine Docker puisse utiliser `seccomp` il est impératif que le kernel dont il est question soit configuré avec l'option `CONFIG_SECCOMP` activée et Docker lui-même doit avoir été compilé avec `seccomp`. Dans ce cas-là, plusieurs appels système sont bloqués par défaut (se référer à )

## Maintenir à jour ses conteneurs...
Ce point est essentiel bien qu'évident. Il est impératif de garder ses conteneurs à jour... comme tout autre élément d'un parc (matériel ou logiciel).
Pour ce faire, il suffit d'analyser les Dockerfiles, qui, une fois de plus, contiennent l'application et son environnement.

## Les configurations rédibitoires (mauvaises pratiques)
### `--privileged` flag
Cette option du `docker run` est tout bonnement à bannir totalement. Elle peut être utile à des fins de tests (lancer des conteneurs depuis un conteneur en montant le socket UNIX où écoute le démon Docker par exemple), mais est à oublier le cas échéant.   

En effet, l'ajout de `--privileged` donne toutes les capabilities au conteneur et lui permet de bypasser les limites imposées par le cgroup `devices` (libre utilisation des special files, et donc du binaire `mknod`).


# Cas d'utilisation et particularités

# Limites de l'isolation de type Docker
## Visualisation des ressources de l'hôte

Du fait de certains mécanismes intrinsèquement liés au fonctionnement de Docker, il est possible de visualiser tout le pool de ressources (CPU, RAM, etc) même en cas de limitation de ces mêmes ressources.  
Par exemple, pour la RAM :

```shell
$ free -m
              total       utilisé      libre     partagé tamp/cache   disponible
Mem:           7901        4788         935         866        2177        1990
Partition d'échange:        2047           0        2047
$ docker run -m 16M alpine free -m
             total       used       free     shared    buffers     cached
Mem:          7901       6974        926        861        444       1650
-/+ buffers/cache:       4880       3021
Swap:         2047          0       2047
```



# TODO
- Les configurations rédibitoires
- use cases/particularités
- construction d'images
- relire
- réseau
