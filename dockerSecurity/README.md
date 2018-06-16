# Conteneurs & Sécurité

Ce document a pour but d'aborder plusieurs aspects de sécurité autour de la conteneurisation.
Avant d'aller plus loin, il est nécessaire de préciser que **nous nous intéresserons principalement ici à Docker.** C'est encore aujourd'hui la solution la plus utilisée, aussi bien pour du développement que pour de la production. Par conséquent, c'est surtout la **conteneurisation sous GNU/Linux** qui sera abordée ici. Ceci pour plusieurs raisons :
* c'est le mode de fonctionnement le plus utilisé
* Docker a principalement muri au sein de l'écosystème GNU/Linux, c'est sur cette plateforme qu'il est aujourd'hui le plus stable et robuste
* c'est le choix le plus flexible : l'utilisateur pouvant avoir une totale maîtrise du fonctionnement de l'outil

Aussi, il est très difficile de couvrir tous les facettes de la sécurité autour de la conteneurisation, ou même de Docker spécifiquement. Nous nous intéresserons ici aux aspects incontournables touchant directement à la limitation de droits ou la réduction de la surface d'exposition. Nous discuterons aussi de certaines problématiques autour du stockage, de la sauvegarde (disponibilité des données) ou encore de certaines autres autour de problèmes cryptographiques (accès sécurisé à un démon, signature des images, etc.).

Bien que n'étant pas sa vocation première, nous nous efforcerons dans ce document de revoir certains aspects purement techniques qu'il est absolument indispensable de connaître et comprendre si on souhaite réellement appréhender la sécurité autour de Docker.

Ce document n'a pas non plus pour vocation d'être un guide pour sécuriser un environnement Docker ou utilisant de la conteneurisation. Nous ne venons ici que mettre en évidence certaines problématiques et y apporter des éléments de réponse. Plusieurs techniques d'atténuation des risques seront malgré tout proposées tout au long du document. 

Enfin, il est attendu de la part du lecteur de connaître un minimum la conteneurisation à l'aide de Docker et son écosystème (bien que beaucoup d'éléments soient présentés sur le tas).

# Introduction
Avant toute chose, il est nécessaire de présenter la conteneurisation, **d'un point de vue purement fonctionnel**. Quels sont les objectifs de la conteneurisation ?
* permettre un système de **packaging** unifié des applicatifs 
* faciliter le **déploiement** d'un applicatif
  * en étant indépendant de la plateforme (du moment que la solution de conteurisation est compatible)
  * en proposant un moyen unifié de gérer les applications
* faciliter le **développement collaboratif** 
  * le code, une fois packagé dans un conteneur (ou plus souvent, une *image*) est transportable d'un poste à un autre, sans se préocupper ni de l'OS ni des librairies nécessaires
* accéder à un **plus grand niveau de sécurité**
  * ajoute un niveau d'isolation (basés sur des mécanismes kernel (GNU/Linux ou Windows) ou une couche de virtualisation)
  * les conteneurs sont souvent stateless *par définition* (reproductibilité, conformité)

Aussi, avant de d'aller plus avant, j'aimerai discuter quelques faits dont il est préférable d'avoir vent. Placer cette discussion en introduction est subjectif, mais cela nous permettra de bâtir un raisonnement basé sur un socle commun.

* les conteneurs GNU/Linux reposent **exclusivement** sur des technologies kernel qui existaient déjà auparavant, qui étaient déjà utilisées pour les mêmes usages (isolation, limitation). Technologies qui ont bien muri, après plusieurs années de dév, rapprochement de la Linux Foundation, création de standards, sensibilisation globale, etc. Un même niveau de confiance peut alors être accordé à la conteneurisation pure (comme la pratique Docker) qu'au kernel lui-même.
* Docker sert notamment à déployer des applications, ainsi qu'à les exécuter et les rendre accessibles
  * le lancement d'applications est habituellement à la charge d'un outil dédié comme **`systemd`** (à l'aide des unités de services `systemd`)
  * **il est admis qu'on ait besoin des droits `root` pour systemd ? Alors pourquoi pas pour Docker ?...** C'est exactement la même surface d'exposition (lorsqu'on parle du lancement d'application), et c'est pour strictement les mêmes raisons qu'ils ont besoin des droits `root` (lecture/écriture de fichiers sensibles, écoute sur port inférieur à 1024, lancement d'applicatifs sous l'identité d'autres utilisateurs, etc.) C'est donc avec tout autant d'attention qu'on surveille un utilisateur sudo, qu'un utilisateur membre du groupe `docker`
  * le démon docker tourne avec `root`, **ce qui n'est pas le cas des conteneurs pris individuellement**. Un conteneur n'est qu'un unique processus, une simple ligne dans un ``ps``. De la même façon, `systemd` tourne sous `root`, mais les services qu'ils lancent peuvent posséder un utilisateur applicatif.
* la conteneurisation ne fait qu'apporter un niveau d'isolation supérieur. Cela soulève un grand nombre nouvelles problématiques, mais, par nature, c'est simplement un nouveau système d'isolation applicative. Et donc une hausse du niveau de sécurité, par rapport à une application qui s'exécuterait directement au sein du système. **Par nature.**
* le terme **conteneurs** désigne un concept, pas une technologie spécifique :
  * les **GNU/Linux containers** (comme Docker) reposent sur des mécanismes du kernel GNU/Linux
  * il existe des conteneurs qui **sont des VMs**, à l'instar de ce que met en place Rocket ou KataContainers
* il est souvent **peu pertinent** de comparer la VM et les conteneurs sur le plan de la sécurité
  * un **conteneur** *peut* **être une VM** comme dit au dessus
  * les conteneurs GNU/Linux et la vitualisation reposent sur des concepts fondamentalement différents
  * ce ne sont pas les mêmes usages ! Donc pas les mêmes surfaces d'exposition, pas les mêmes problématiques, etc.
  * **l'hôte de conteneurisation EST -très souvent- UNE VM** : le conteneur n'apporte qu'un niveau supérieur d'isolation, encore une fois.
  * une fois ces idées en tête, on peut comparer de façon fonctionnelle et technique la mise en place de conteneurs et VMs, cela pouvant aider à adopter un point de vue global. Mais **pour juger du niveau de sécurité des conteneurs de manière effective, c'est un raisonnement depuis zéro qu'il faut adopter et non pas un raisonnement comparatif**.
* la conteneurisation n'empêche pas de continuer à adopter les bonnes pratiques habituelles :
  * segmentation/isolation réseau
  * amélioration de la disponibilité des données (sauvegarde, redondance du stockage)
  * principe du moindre privilège
  * utilisation d'outils de sécurité dédiés

# Sommaire
* [Sources](#sources)
* [Communauté & standards](#communauté--standards)
  * [Un peu d'histoire](#un-peu-dhistoire)
  * [OCI : Open Container Initiative](#open-container-initiative)
* [How do containers work ?](#how-do-containers-work-)
  * [Technologies noyau](#technologies-noyau)
  * [Containers, Runtimes, Docker](#containers-runtimes-docker)
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
    * [Plus grande exposition de vulnérabilités](#plus-grande-exposition-des-vulnérabilités)
    * [Gestion de secrets](#secrets-management)
    * [Bases de données, clusters, et consistence](#base-de-données--clusters--consistence)
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
  * [Mauvaises pratiques](#les-configurations-rédibitoires-ou-mauvaises-pratiques)
  * [Quelques limites de Docker](#limites-de-docker)


# Sources
Mes sources sont nombreuses, et une grande partie de ce document a été rédigé d'une traite.
La majeure partie des informations trouvables dans ce document se trouvent dans les référentiels suivants :
* [The Linux Documentation Project](http://www.tldp.org/)
* [Documentation Red Hat](https://access.redhat.com/documentation/en/red-hat-enterprise-linux/)
  - [Red Hat Container Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/)
* [Documentation Docker](https://docs.docker.com/)
* [Documentation Kernel](https://www.kernel.org)
* LWN.net avec des discussions comme [celle-ci au sujet des cgroups](https://lwn.net/Articles/679786/)
* la commande ```man``` (trouvable aussi [ici](http://man7.org/linux/man-pages/man7/))
  * ce sont les documentations les plus simples et complètes sur les fonctionnalités du kernel (les `man`s ne documentent pas que des binaires, mais aussi des fonctionnalités kernel, des concepts, etc.)
  * plus clair que le `man`, c'est lire le code.
  * ``man cgroups``, ``man namespaces``, etc.
* [Wiki de la Linux Foundation](https://wiki.linuxfoundation.org)
* [Site vitrine de l'OCI](https://www.opencontainers.org/)
* Recommandations de sécurité
  * [par Delve Labs](https://www.delve-labs.com/articles/docker-security-production-2/)
  * [par GDSSecurity](https://github.com/GDSSecurity/Docker-Secure-Deployment-Guidelines)
* Discussions autour de la sécurité
  * [Conférence du SSTIC](https://www.sstic.org/2018/presentation/audit_de_securite_docker/) au sujet des audits en environnements de conteneurisation

Certains autres documents constituent des lectures très intéressantes autour des conteneurs :
* Cisco et RedHat : ["Linux Containers: Why They’re in Your Future and What Has to
Happen First"](https://www.cisco.com/c/dam/en/us/solutions/collateral/data-center-virtualization/openstack-at-cisco/linux-containers-white-paper-cisco-red-hat.pdf)


**Ce document n'est donc qu'une synthèse d'un savoir communautaire.**

# Communauté & standards
Cette partie est volontairement placée en première place. L'aspect communautaire d'une solution est souvent une arme très puissante pour la pérenniser.
L'établissement de standards est aussi très important : ils permettent l'abstraction d'une technologie en particulier au profit de concepts directeurs. C'est aussi dans ces standards que sont établis les bonnes pratiques, les mécanismes fondamentaux et les principaux aspects de sécurité.

## Un peu d'histoire
La conteneurisation telle qu'on la connaît aujourd'hui est le fruit de multiples avancées ces dernières années. On considère souvent l'appel système *chroot* (UNIX) comme le père des conteneurs. En effet, celui-ci permet, au sein d'un système de fichiers, de changer l'emplacement de la racine de façon arbitraire.
Les conteneurs avec lesquels nous sommes familiers sont nés aux alentours de 2008, avec LXC (développé par IBM). LXC utilisant les mêmes technologies kernel qui sont utilisées aujourd'hui par Docker. 

Cet environnement technique préparé depuis plusieurs années à créer un terrain propice pour le développement d'un outil plus accessible, outil qu'a été Docker. 

A l'origine, la société **Docker avait mis sur pied sa propre librairie afin de pouvoir utiliser des conteneurs**. Ce n'est ni plus ni moins qu'une interface (au sens premier du terme) vers le kernel, qui expose une API simplifiée et dédiée à la conteneurisation.
De façon simple, avant l'existence de telles librairies (et `libcontainer` de Docker **n'était pas** la première), il était nécessaire de créer manuellement les `cgroups` et `namespaces`, ainsi que de les articuler entre eux, afin d'avoir un conteneur. Avec une telle lib, tout ce travail technique a été abstrait pour l'utilisateur, et donc, simplifié.

La société a aussi mis à disposition **un outil CLI très puissant et clé-en-main**, permettant d'utiliser cette API.

Mais, l'engouement autour de la Docker étant grandissant, et le nombre de prospects croissant de façon exponentielle (car répondant à un certain nombre de besoins, ce dont nous ne traiterons pas ici), d'autres runtimes de conteneurisation ont rapidement vu le jour.

S'est alors rapidement posée **la question de la standardisation**. On ne parle plus de Docker, qui a révélé ces technologies au plus grand nombre, mais de la conteneurisation en général, dans le monde de demain. La question n'est en effet plus de savoir si les conteneurs seront là demain, on connaît la réponse et c'est oui. La question c'est : comment ?

## Open Container Initiative
L'OCI est un projet de la Linux Foundation, s'inscrivant dans cette mouvance globale de standardisation des usages techniques afin de permettre une évolution de l'informatique au sens large, de façon concrète, et sans accrocs. On peut aussi par exemple noter l'[Open API Initiative](https://www.openapis.org/) qui s'inscrit dans ce même mouvement.

L'OCI propose donc des spécifications, des standards : pour les images de conteneurs, ou encore pour le runtime lui-même. Pour le runtime, on trouve par exemple la spécification [ici](https://github.com/opencontainers/runtime-spec/blob/master/spec.md) et son implémentation libre et open-source [ici](https://github.com/opencontainers/runc). L'implémentation porte le nom de ```runc``` et c'est désormais le runtime utilisé par défaut par Docker (entre autres). **Docker n'est plus, vive Docker** (*Docker* ne désigne donc qu'un outil commercial).

A noter que le projet est soutenu par de grands acteurs parmis lesquels Microsoft, IBM, Huawei, EMC, Docker, Dell, Google, HP, Goldman Saaaaaachs, Twitter, RedHat, etc.

Ceci est déterminant pour l'avenir de la conteneurisation. En effet, le développement de la conteneurisation au sens large, est désormais encadré par une fondation qui est (à priori...) la plus indépendante possible d'autres organismes et acteurs majeurs du milieu informatique. Mais tout en bénéficiant de leur soutien : le projet n'est donc pas mené de manière isolée, et il ne sera pas le fruit de la volonté d'une unique entité pour servir ses intérêts.

**L'OCI cherche à faciliter la transition vers le monde des conteneurs, et surtout, à en pérenniser l'utilisation.**

# How do containers work ?

Nous allons voir dans cette partie (purement technique) certaines des technologies qui rendent la conteneurisation telle qu'on la connaît possible.
Cette partie n'est qu'une partie théorique et technique et n'est pas directement rattachée à la sécurité, mais est strictement indispensable pour discuter de problématiques de sécurité autour des conteneurs. Cela nous sert aussi de base commune pour ce qui est du vocabulaire.

## Technologies noyau
Les `cgroups` et `namespaces` sont deux technologies kernel qui sont au centre de la conteneurisation.
Nous n'allons pas rentrer ici dans les détails, mais nous allons malgré tout présenter le fonctionnement de ces éléments pour comprendre sur quoi repose véritablement l'isolation qu'apportent la conteneurisation de type Docker. **En effet, c'est un aspect essentiel de la sécurité de Docker**.

### cgroups

**Les `cgroups` permettent de grouper et labelliser des processus, afin de leur allour une quantité de ressource limitée** (ses fonctionnalités sont en réalité plus larges, mais nous n'en discuterons que très peu ici). Un `cgroup` peut être limité en fonction de plusieurs *contrôleurs* (accès au CPU, aux devices, à la mémoire, etc.)
Il existe un certain nombre de contrôleurs qui peut dépendre en fonction de la distribution (on en trouve 11 généralement). En voici quelques-uns (les principaux) afin d'avoir une idée de leur utilité :
* ***cpuset*** : permet d'allouer un coeur processeur à un groupe de tâche
* ***blkio*** : permet de limiter les actions de lecture ET d'écriture sur un périphérique de type bloc
* ***net_prio*** : permet de prioriser le trafic de certaines interfaces réseau vis-à-vis d'autres

* `man cgroups` pour une liste complète.

Par défaut, à chaque conteneur lancé est créé un `cgroup` correspondant.

*A noter que la version 2 des `cgroups` n'a pas encore été adoptée, pour des problèmes de compatibilité. Voir [ici](https://github.com/opencontainers/runc/issues/654) et [ici](https://github.com/moby/moby/issues/16238), à suivre*.

* pour jouer avec les cgroups :
  * répertoire `/sys/fs/cgroup/`
  * fichier `/proc/<PID>/cgroup` pour chaque processus

### namespaces
**Les `namespaces` (ou ns) permettent d'isoler de manière effective des ressources.** Aux yeux des `namespaces` (et donc, du noyau), il existe un nombre limité de "ressources". La plupart ont une structure arborescente. Il en existe 6 :
* ***pile réseau*** (ou network stack)
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

Le noyau est donc capable de gérer deux (ou plus) arborescences de processus totalement indépendantes.

**NB:** même si tous les namespaces portent le nom de "namespace", ils ont TOUS un fonctionnement différent. Dans l'idée, ils servent bel et bien tous à isoler un jeu de ressources d'un autre, du même type, aux yeux du système. Cependant, par exemple, les namespaces `user` (doit inclure une gestion des capabilities par namespaces) et `mount` (inclut des mécanismes de scopes et de propagation des points de montage) ont un fonctionnement qui diffèrent beaucoup des autres.  

**NB2:** un namespace continue d'exister :
* tant qu'il est utilisé
* tant qu'il existe un lien qui y fait référence (où que ce soit sur le système)

Pour jouer avec les namespaces : 
* répertoire `/proc/<PID>/ns/` qui contient des liens symboliques vers les namespaces (facilement identifiés par un ID numérique) utilisés par le processus ciblé
* `unshare`, interface directe vers l'appel système du même nom : permet de créer des namespaces
* `nsenter` permet de rentrer dans un namespace (et d'y exécuter des processus)

#### Explorer les namespaces : `nsenter`

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

#### Création de namespaces : `unshare`

Il existe un appel système qui porte le même nom : `unshare()`. Ce dernier permet tout simplement de créer de toutes pièces un nouveau namespace, d'un type particulier. La binaire `unshare` utilise cet appel système et nous permet de lancer des processus dans de nouveaux namespaces. A noter qu'il est aussi possible de l'utiliser pour lancer des processus dans des namespaces existants.

Les options de `unshare` permettent de choisir le type de namespace(s) que l'on va créer à la volée. Un des exemples les plus simples est de lancer un `bash` dans un namespace de type `net` (réseau). Par défaut, un namespace `net` vierge ne contient qu'une unique interface : la loopback.
```shell
$ unshare -n bash
$ ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Il est possible de quasiment créer un conteneur tel que Docker le fait, en une unique commande `unshare` :
```shell
$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 18:18 pts/0    00:00:00 bash
root         2     1  0 18:18 pts/0    00:00:00 ps -ef
$ ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
Congratz, you basically just created a container. (`exit` ou `CTRL+D` pour quitter le shell, et donc, sortir du namespace nouvellement créé).

### capabilities

**Les capabilities Linux sont les droits avec lesquels un processus est lancé.** `root` n'est pas un utilisateur magique. `root` n'a pas tous les droits. `root` est simplement un utilisateur qui a le pouvoir de lancer n'importe quel processus avec n'importe quelle capability.
On en en trouve un certain nombre (dépend du système, on en trouve souvent + de 35). Parmi lesquelles on a par exemple :
* ***CAP_SYS_PTRACE*** : permet d'utiliser ```ptrace``` et ```kcmp```
* ***CAP_AUDIT_WRITE*** : permet d'écrire dans les logs kernel
* ***CAP_DAC_OVERRIDE*** : permet de passer outre les permissions de lecture/écriture/exécution
* ***CAP_NET_ADMIN*** : permet de modifier les tables de routage, les règles de proxying, la possibilité de multicast, etc.
* ***CAP_NET_BIND_SERVICE*** : permet d'écouter sur un port en dessous de 1024

On reconnaît ici les superpouvoirs de `root`. Il est possible de visualiser les capabilities de chacun des processus lancés sur le système avec le binaire ```lscap``` rarement présent par défaut (quelque soit la distrib).

A l'instar de n'importe quel autre processus du sysème, un conteneur se voit attribuer un certain nombre de capabilities. Il est possible lors du lancement d'un conteneur Docker (`docker run`) d'en ajouter (`--cap-add`) ou d'en supprimer (`--cap-drop`).

Une bonne façon de procéder lors du lancement d'un conteneur est de supprimer toutes les capabilities puis d'ajouter uniquement celles qui sont nécessaires :   
```shell
$ docker run --cap-drop ALL --cap-add SYS_TIME --name whattimeisit alpine /bin/sh
```

Les capabilities s'appliquent aussi aux binaires. On les manipule alors avec `setcap` et `getcap`. L'utilisation d'un *setuid root* est parfaitement équivalent à donner TOUTES les capabilities à ce binaire à l'aide de `setcap`.  `setcap` est donc un `setuid root` mais beaucoup plus fin et granulaire.

```shell
$ getcap /usr/bin/ping
/usr/bin/ping = cap_net_raw+ep
```

Le mécanisme est le suivant :
1. un utilisateur lance un binaire
2. le kernel vérifie que l'utilisateur a les droits (mécanismes DAC, MAC, autres)
3. en cas de succès, le kernel déduit les capabilities que possèdera le processus, en fonction :
  * des capabilities du binaire spécifié
  * des capabilities du processus appelant (souvent, un shell)
4. le processus est créé

**Trick:** on peut utiliser le fichier `/etc/security/capability.conf` pour positionner des capabilities sur un utilisateur du système. En réalité, ce fichier permet de positionner des capabilities sur le shell par défaut de l'utilisateur (voir `/etc/passwd`). La syntaxe est la suivante, pour ajouter la capability CAP_SYS_PTRACE au shell de l'utilisateur `johnny` :
```shell
$ cat /etc/security/capability.conf
cap_sys_ptrace johnny
none *  # Fortement recommandé : désactive toutes les capas pour tous les users non définis dans le fichier
```

Extrait de la [documentation RedHat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/docker_selinux_security_policy) : *"Capabilites constitute the heart of the isolation of containers. If you have used capabilites in the manner described in this guide, an attacker who does not have a kernel exploit will be able to do nothing even if they have root on your system."*

Jouer avec les capas :
* `setcap` et `getcap`
* `capsh`

#### Mise en évidence

```shell
# logged as an unprivileged (but sudo) user
$ /bin/ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.098 ms

$ getcap /bin/ping
/bin/ping = cap_net_admin,cap_net_raw+p

$ cp /bin/ping ./
$ ./ping 127.0.0.1
ping: socket: Operation not permitted
$ getcap ./ping # outputs nothing
$ sudo setcap cap_net_raw+ep ./ping
$ ./ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.072 ms
```

### netfilter
Il existe un **framework kernel permettant de lui donner la fonction de firewall : c'est netfilter.**
Par défaut, le démon Docker utilise grandement les fonctionnalités kernel de ```netfilter``` afin de fournir du réseau aux différents conteneurs dont il est responsable. Il est évidemment possible de les visualiser à l'aide de ```iptables``` (ou ```ebtables```). Afin de pouvoir gérer le trafic, le démon crée une interface virtuelle pour chaque "réseau Docker" (on peut visualiser les réseaux Docker avec ```docker network ls```).
Le majeure partie des règles permettent :
* de forwarder du traffic vers un conteneur spécifique
* de fermer certains ports

**NB:** il existe un jeu de règles `netfilter` pour chaque namespace de type `net` (e.g. tous les conteneurs possèdent leurs propres stacks réseaux, règles firewall, etc.)

Jouer avec namespaces réseau/netfilter :
```shell
$ ip netns add super_net_ns
$ ls /var/run/netns/ # or ip netns list
super_net_ns

$ ip netns exec super_net_ns # permet d'exécuter des commandes dans le namespace
$ ip netns exec super_net_ns ip a # par exemple
```
### Un mot sur `/proc` et `/sys`

Comme on le répète souvent pour Linux : tout est fichier ou processus. En l'occurrence, comme **la totalité des fonctionnalités élémentaires d'un système moderne** -des fonctionnalités comme le réseau- **sont gérées dans le kernelspace**. Des appels systèmes sont à disposition pour pouvoir les manipuler. Mais en explorant les deux pseudo-filesystems que sont **/proc** et **/sys**, on peut obtenir énormément d'informations (en réalité, tout est là, donc c'est ici que l'on pourra obtenir le plus d'informations sur le réseau. Les commandes comme ``ip`` ou ``ifconfig`` ou comme ``hostname`` ne font que piocher à un moment ou un autre dans ces données).

On a par exemple ``/sys/class/net`` qui contient un sous-répertoire par interface réseau de la machine (à l'intérieur se trouve un grand nombre de paramètres, contenus dans des fichiers).   
Dans le cas de Docker, avec le driver réseau de base, chaque réseau Docker est en réalité une nouvelle interface bridge. Chaque conteneur créera alors une sous-interface de ce bridge, qui apparaîtra comme un sous-répertoire supplémentaire. Evidemment, il est impossible depuis le conteneur de voir d'autres interfaces que les siennes. Ceci, grâce aux namespaces.

Autre exemple, isoler un processus dans une namespace PID revient à utiliser un autre `/proc`, puisque ce dernier liste tous les processus du système.

## Containers, Runtimes, Docker

#### Container

Un conteneur (ou *container* en anglais) est un outil, faisant appel à des mécanismes système d'isolation et de limitation, permettant l'exécution restreinte d'un processus (et de ses fils). Il est possible de les créer et gérer à la main (dans des VMs, ou avec un n oyau GNU/Linux comme vu plus haut), mais on préfère souvent utiliser une solution dédiée.

#### Runtime

Lorsqu'il s'agit de la conteneurisation, le *runtime* fait référence à l'outil (ou l'ensemble d'outils) permettant d'abstraire les mécanismes système vis-à-vis de l'utilisateur. Aujourd'hui, on distingue deux types de runtime : 
* un outil permettant de gérer les conteneurs eux-mêmes de façon individuelle
  * abstraction des *namespaces*, *cgroups*, et *capabilities*, entre autres
* un outil permettant de gérer l'environnement de conteneurisation
  * gestion d'images, de réseau, de stockage

[`¢ontainerd`](https://containerd.io/) est aujourd'hui le runtime standard, poussé par l'OCI, et notamment utilisé par Docker et Kubernetes, allié avec `runc` pour l'exécution de conteneurs.

#### Docker (sous GNU/Linux)

L'architecture de base de Docker consiste en un démon qui écoute sur un socket (UNIX ou TCP), afin de créer des conteneurs. Le binaire `docker` permet de passer des requêtes vers l'API exposée par le démon.

Docker (sous GNU/Linux) utilise comme runtime par défaut [`¢ontainerd`](https://containerd.io/) qui embarque [`runc`](https://github.com/opencontainers/runc). `¢ontainerd` est un démon qui écoute sur un socket, c'est lui qui est chargé de recevoir les ordres utilisateur (`docker run` = "lance un conteneur !"). Lorsqu'un conteneur est lancé, `runc` est utilisé afin de créer l'environnement d'exécution bas-niveau du conteneur (*namespaces*, *cgroups*, et *capabilities*). La communication entre `¢ontainerd` est le conteneur est maintenu grâce un processus `shim`.

Mise en évidence : 
```shell
$ docker run -d alpine sleep 9999
49d63de8f4281c429f0078e6280e4a9886f4a60c87b3df139a65f1e5c9b8cad5
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
49d63de8f428        alpine              "sleep 9999"        2 seconds ago       Up 1 second                             competent_kapitsa
$ ps -ef | grep docker
root      3846     1  0 12:42 ?        00:00:15 /usr/bin/dockerd
root      3853  3846  0 12:42 ?        00:00:11 docker-containerd --config /var/run/docker/containerd/containerd.toml
root      6938  3853  0 14:30 ?        00:00:00 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/49d63de8f4281c429f0078e6280e4a9886f4a60c87b3df139a65f1e5c9b8cad5 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
$ ls /var/run/docker/runtime-runc/moby
49d63de8f4281c429f0078e6280e4a9886f4a60c87b3df139a65f1e5c9b8cad5
```
* le processus `dockerd` est le démon Docker qui réceptionne nos ordres
  * l'utilisateur communique **via** une API REST (socket UNIX ou TCP) avec le démon (par défaut dans `/var/run/docker.sock`)
  * l'utilisateur utilise un client, comme le binaire `docker`
* le processus `docker-containerd` est la version wrappée de `containerd` enmbarquée par Docker. 
  * `dockerd` et `containerd` communique à l'aide d'un socket gRPC (par défaut dans `/var/run/docker/containerd/docker-containerd.sock`)
  * `docker-containerd` peut lancer des conteneurs grâce à `runc`
* le processus `docker-containerd-shim` permet de conserver la communication entre le conteneur et `¢ontainerd`
  * cela permet de continuer à effectuer des actions ou récolter des informations sur le conteneurs pendant son exécution
  * les informations utilisées par le `shim` sont facilement mis en évidence sur la ligne (utilisation du même socket gRPC et de `runc` pour lancer le conteneur)

# Questions récurrentes
#### **Est-ce qu'un conteneur est moins secure qu'une VM ?**
Comme dit plus haut, ce n'est simplement pas comparable. Les deux reposent sur des concepts fondamentalement différents. De plus, dans la quasi-totalité des cas, l'hôte de conteneurisation est une VM. La conteneurisation n'est donc simplement qu'un niveau d'isolation supplémentaire.   


#### **Si un attaquant exploite une vulnérabilité exposée par un conteneur, a-t-il une main-mise totale sur la machine sous-jacente ?**
Cela dépend complètement du type de vulnérabilité.
De façon générale, ça n'expose ni plus ni moins le système qu'avec un applicatif "classique" (lancé via un binaire, ou service comme *systemd*, directement depuis le système hôte (eg. directement dans les `namespaces` "principaux" de l'hôte)).
En effet, une vulnérabilité kernel qui affecte un conteneur rendra l'hôte tout aussi vulnérable car les deux partagent **"physiquement"** le même kernel.
A l'inverse, une vulnérabilité applicative (par exemple, vulnérabilité dans la version d'Apache utilisée) n'exposera pas plus le système qu'avant. Voire moins, ceci dépend de la configuration mise en place (démon, image, conteneur, cf les parties dédiées, plus bas).
Au mieux, l'attaquant sera strictement confiné dans le conteneur. Au pire, il aura une totale main-mise sur le système sous-jacent (qui, pour rappel, est une VM...). A mi-chemin, il sera capable de se rendre compte qu'il est dans un conteneur, et éventuellement obtenir des informations sur le système hôte.    

#### **L'hôte c'est vraiment une VM ?** *ou* **Ca run sur du bare-metal aussi nan ?**  
On trouve effectivement un engouement autour de la "conteneurisation bare-metal". Dans l'état de l'art actuel, on parle d'installer un OS de type GNU/Linux en bare-metal, puis le démon Docker et ainsi d'être plus proche du matériel (pas de couche de virtualisation de type VM). Ceci a plusieurs effets :
- [augmenter **drastiquement** les performances](http://www.stratoscale.com/blog/containers/running-containers-on-bare-metal/)
- pas de licensing pour hyperviseurs
- induit une **sur-exploitation des capacités des machines actuelles** (aussi appelé "comment défoncer un cache CPU", on note aussi une sur-utilisation du réseau/stockage) car prévu pour de la VM (ça veut aussi dire que le plein potentiel de ces technos n'est pas encore tout à fait exploité)
  - les dernières générations de serveurs sont (presque) adaptées. Encore quelques (dizaines d') années avant de voir leur adoption à grande échelle
- **moins d'isolation** : on ne profite plus de l'isolation matérielle que nous offre la VM (notamment en virtualisation toute la couche OS) : la virtualisation a bientôt 30 ans d'utilisation derrière elle (écosystème riche, processus rodés, etc)
- Intel travail sur les KataContainers (anciennement Intel Clear Containers), optimisés pour s'exécuter en bare-metal sur certains de leurs processeurs

Donc oui, ça run sur du bare-metal. Mais il y a à la fois des avantages et des inconvénients. Certains pensent que les conteneurs ne sont qu'une première marche avant l'avènement des [unikernels](http://unikernel.org/) et des architectures serverless (l'hyperconvergence des infrastructures semblent aussi aller en ce sens).

Une réaction qui peut être adoptée est de mixer les deux, pour des usages différents. Deux possibilités :
- machines en local, maîtrisées, conteneurisation bare-metal, hautes performances, low-latency
- machines dans le cloud, hardware abstrait, conteneurisation dans de la VM

*"Generally speaking, low latency apps do better in-house, while capacity optimization, mixed workloads, and disaster recovery favor virtualization." from [here](https://morpheusdata.com/blog/2017-04-28-the-drawbacks-of-running-containers-on-bare-metal-servers)*

# Sécurité et bonnes pratiques Docker
## Sécurité du démon Docker
### Exposition de l'API
L'API peut être exposée *via* deux technologies :
* Par défaut, c'est un **socket UNIX** qui est utilisé (il se trouve à ```/var/run/docker.sock```). La communication est donc interne au système. La surface d'exposition est extrêmement faible, c'est le moyen à privilégier.
* Utilisation d'un **socket TCP**. Il est possible par ce biais de contrôler un démon Docker à distance, auquel cas il est très **fortement recommandé d'utiliser un tunnel de chiffrement à l'aide du protocole TLS.** Il est aussi impératif de ne rendre possible l'accès à ce service qu'avec des règles réseau très restrictives (LAN isolé, accès à ce LAN restrictif, etc.). **De façon générale, l'exposition sur un socket TCP est tout simplement déconseillée**.
### Options du démon Docker
Le démon docker (généralement géré à l'aide de ``systemd``) est lançable manuellement avec le binaire ``dockerd``. De nombreuses options sont à notre disposition, parmis lesquelles :
* ``--icc`` : permet d'activer la communication inter-conteneur. **``true`` par défaut.**
* ``--ip`` : permet de choisir l'adresse de l'hôte utilisée pour les forwardings de port. **Par défaut : 0.0.0.0** (toutes les interfaces seront utilisées)
* [``--userns-remap``](#-user-namespace-remapping) : permet de choisir arbitrairement quel `user namespace` sera utilisé par le conteneur, le tout en mappant les utilisateurs du nouveau namespace vers des utilisateurs du "root namespace" de type `user`. Par défaut, les namespaces `user` ne sont pas utilisés.
* `--oom-score-adjust` : détermine le OOM score du processus `dockerd`. **-500 par défaut**

Ces quelques options peuvent suffire à faire peur. Et leur paramétrage par défaut est caractéristique de "this is a feature, not a vuln". Ce ne serait que pure vulnérabilité si on ne pouvait pas modifier nous-mêmes ces paramètres.  
Et je pense qu'ils ont raison de considérer ça comme une "feature". La sécurité par défaut **pourrait** en effet être renforcée. Mais une configuration très restrictive de cet outil, réalisée arbitrairement par les développeurs de Docker, engendrerait nécessairement un gros travail de configuration à tous les utilisateurs. Ce n'est pas un problème en soit, mais le travail serait effectivement énorme.   
Or, dans beaucoup de cas on utilise Docker à des fins de tests, sur notre propre ordinateur. **Dans la plupart des cas, on attend juste de Docker qu'il fonctionne.** Ces choix sont donc compréhensibles. Pour une utilisation en production, tous ces éléments restent donc bien évidemment paramétrables.

J'étais moi-même sceptique au départ, mais d'autres exemples viennent contrecarrer le sceptiscisme. Par exemple, SELinux est par défaut quasiment non configuré et trop restrictif sur les machines de type RedHat. Dans beaucoup de cas, les gens le désactivent simplement (ou le laissent afficher des warnings) alors qu'il constitue une brique extrêment robuste de la sécurité des systèmes RedHat. Résultat ? Il est inutilisé, bien qu'extrêmement puissant.
Docker a fait le choix de désactiver ces options, pour satisfaire le plus grand nombre. Et il est possible de se le permettre car le public visé va, à 95% faire tourner Docker sur un ordinateur personnel à des fins de tests, POC, développement (e.g. sans l'exposer vers l'extérieur, et n'attendant de Docker qu'une unique chose : qu'il fonctionne).

**Si ces sécurités sont désactivées par défaut, c'est effectivement une feature. L'introduction de cette feature est la conséquence du public visé par Docker.**

## Sécurité des hôtes Docker

**TODO** : éclater cette partie

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
* Utiliser un **OS dédié**. Il en existe désormais plusieurs, qui présentent tous des avantages différents car matérialisant des concepts différents. De façon générale, on retrouve certaines caractéristiques : OS léger (stockage), Docker présent nativement, diverses optimisations kernel, ou encore la sécurité (politique moindre privilège orientée Docker uniquement, etc.) On peut par exemple citer :
  * [RancherOS](http://rancher.com/rancher-os/) - le plus léger (~20Mo), uniquement kernel opti + Docker qui remplace systemd
  * [AtomicHost](http://www.projectatomic.io/download/) - développé par RedHat, embarque la sécu native RHEL (SELinux, seccomp) et est aussi optimisé pour faire tourner des conteneurs (avec Docker)
  * [PhotonOS](https://vmware.github.io/photon/) - développé par VMWare, assez léger et limite aussi la surface d'exposition. Sa force résidant dans l'intégration avec vSphere (pour le [VIC runtime](https://vmware.github.io/vic/))
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

### Le *Hub Docker*
Un applicatif, le *Registre Docker*, permet d'héberger des images. On les récupère avec ``docker pull`` et on les envoie sur le *Registre Docker* avec ``docker push``.

Il existe un *Registre Docker* public, appelé [Docker Hub](https://hub.docker.com/), il est le *Registre Docker* ciblé par défaut par les commandes ``docker push`` et ``docker pull``.

On y trouve des dépôts de particuliers, des dépôts d'entreprises (microsoft, vmware, etc) et un dépôt particulier : [le dépôt library](https://hub.docker.com/explore/) (qui contient les "official repositories"). Le dépôt *library* contient des images qui sont vérifiées par la société Docker et qui sont directement émises par les éditeurs des solutions en question. On y trouve un certain nombre de services très communs comme alpine, busybox, apache, mysql, redis, nginx ou encore mongodb.

En résumé, les images dignes de confiance sont :
* les images contenues dans le dépôt library du Docker Hub
* les images développées en interne, basées sur des images de library
* les images développées en interne, ``FROM scratch``
* les images basées sur des images développées en interne

On peut éventuellement accorder de la confiance à d'autres éditeurs, dont les dépôts ne sont pas dans *library* (par exemple, [vmware](https://hub.docker.com/u/vmware/)).

### Un *Registre Docker* interne ?
En effet, l'utilisation d'un *Registre Docker* est le seul et unique moyen de stocker, partager et gérer des images avec sécurité, qualité et granularité :
* les images sont centralisées sur des noeuds dédiés
* tous les hôtes Docker peuvent récupérer les images sur un *Registre Docker* interne (inutile de toutes les stocker en local donc)
* possibilité de conserver une archive des images poussées
* facilité pour effectuer des sauvegardes des images sans connexions aux hôtes Docker
* confiance totale en ce registre interne pour la sécurité (disponibilité, confidentialité, authentification)
* beaucoup d'implémentations du *Registre Docker* permettent :
  * de visualiser le contenu du *Registre Docker* à l'aide d'une UI dédiée
  * gérer des utilisateurs, des groupes, et des permissions
  * d'autres fonctionnalités ([réplication d'images](https://github.com/vmware/harbor/blob/master/docs/user_guide.md#replicating-images), [émission de jetons applicatifs](http://port.us.org/features/application_tokens.html), couplage à une backend LDAP pour l'auth, etc)

### Construction d'images
#### > Clause `FROM`
La clause `FROM` doit faire référence à une image digne de confiance (voir le paragraphe juste au dessus).  
Il est aussi préférable d'éviter le tag `:latest` dans le cadre d'une utilisation en production. En effet, le tag `:latest` est régulièrement mis à jour avec des nouvelles versions de l'image en question. En cas de changements dratiques, il y a fort à parier que votre conteneur ne s'exécutera plus comme voulu, ou sera tout simplement incapable de se lancer.

#### > Overlay filesystem
Les images Docker sont stockées par défaut dans `/var/lib/docker/images`. Elles sont stockées à l'aide de filesystems particulier : des **overlay filesystems**. Un overlay filesystem repose sur un concept d'entassement de couches successives. C'est grâce à ces filesystems que Docker peut réutiliser certains "morceaux" d'images pour en construire d'autres, ou encore, c'est grâce à eux aussi que les images Docker sont immuables.  

**En effet, chaque ligne dans un Dockerfile résultera en une couche (ou "layer") supplémentaire.** Une image qui, dans son Dockerfile, présente cette même ligne pourra alors directement réutiliser la première couche qui a été build.  

Lorsque l'on stocke un grand nombre d'images il est donc préférable de penser à la réutilisation des différentes couches au moment de la rédaction des Dockerfiles.

#### > Maintien des images
Il est **impératif** de maintenir ses images. De la même façon qu'il est nécessaire de maintenir toutes autres applications packagée autrement. Tout vient d'être dit : une fois packagé, un package (image Docker ou autres) est immuable : il faut le ré-éditer afin de le mettre à jour.  

Afin de garder des images à jour, il n'est nécessaire que d'éditer le Dockerfile dont ces images ont été issues, puis de réitérer la phase de `docker build` afin de ré-éditer l'image.

## Sécurité des conteneurs Docker
Ici, on fait clairement référence au lancement **d'un** conteneur, à partir d'une image, à l'aide de la commande ``docker run``.
### Les options
La commande ``docker run`` possède d'innombrables options. Seule une infime partie est utilisée au quotidien, des choses comme :
* ``-p`` : permet de forwarder un port de l'hôte vers un conteneur
* ``--hostname`` : permet de spécifier arbitrairement un nom d'hôte pour la machine
* ``-v`` : permet d'utiliser des volumes Docker

Cependant on trouve aussi un grand nombre d'options auxquelles il est **nécessaire** de s'intéresser afin d'appréhender la sécurité d'un conteneur. Parmi celles-ci, on a :
* ``--cap-drop`` qui permet de supprimer des capabilities au conteneur lancé
  * pour rappel : de chaque conteneur lancé avec ``docker run`` résultera un **unique** processus. C'est donc ce processus qui se verra supprimer des capabilities.
* politiques de restriction d'accès aux ressources à l'aide des cgroups. Beaucoup d'options en relation avec ce point :
  * ``blkio-weight`` : priorisation des accès disque, possibilité de désactiver l'écriture sur le disque avec une valeur nulle
  * ``--cpus-quota`` et ``--cpu-period`` permettent d'agir sur le CFS du processeur
  * ``--memory`` : limitation de la quantité de RAM utilisée
  * ``--tmps`` : monte un système de fichier temporaire en RAM ([`tmpfs`](http://man7.org/linux/man-pages/man5/tmpfs.5.html) au path indiqué
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

### Secrets management

### Base de données & Clusters & Consistence

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

**NB:** de la même façon qu'il est impossible de monter deux targets de stockage sur un même point de montage, il est impossible de monter deux volumes (locaux ou distants) sur le même répertoire d'un conteneur.

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
- automatiquement : couplé à un Registre Docker. En effet, de cette façon, il est possible d'agir directement au moment où une image est poussée. A chaque `push`, les images sont contrôlées, et des actions pourront être déclenchées en cas de résultat(s) positif(s).
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

Pour que l'runtime Docker puisse utiliser `seccomp` il est impératif que le kernel dont il est question soit configuré avec l'option `CONFIG_SECCOMP` activée et Docker lui-même doit avoir été compilé avec `seccomp`. Dans ce cas-là, plusieurs appels système sont bloqués par défaut (se référer à )

## Les configurations rédibitoires (*ou mauvaises pratiques*)
### `--privileged` flag
Cette option du `docker run` est tout bonnement à bannir totalement. Elle peut être utile à des fins de tests (lancer des conteneurs depuis un conteneur en montant le socket UNIX où écoute le démon Docker par exemple), mais est à oublier le cas échéant.   

En effet, l'ajout de `--privileged` donne toutes les capabilities au conteneur et lui permet de bypasser les limites imposées par le cgroup `devices` (libre utilisation des special files, et donc du binaire `mknod`).

### Un conteneur = un processus
Il est parfaitement possible de s'amuser à effectuer toute une batterie de tests dans des conteneurs. Cependant, pour une utilisation en production, **chaque conteneur ne doit lancer qu'un seul et unique processus.** Celui-ci est contenu dans la clause `ENTRYPOINT` du Dockerfile.  

Cela peut paraître désuet mais c'est en réalité **primordial** pour une utilisation robuste des conteneurs. En adoptant ce concept, on facilite l'interopérabilité de nos conteneurs, leur capacité d'adaptation, tout en accroissant leur légèreté et donc leur portabilité.

### Hébergement de données
Par "donnée", on entend ici tout ce qui n'est pas ni l'application, ni son environnement.  

Les conteneurs sont éphémères. Toute donnée se trouvant dans un conteneur est vouée à disparaître. **Les données doivent être hébergées dans des volumes dédiés** afin d'accéder à de la persistance de données et un plus haut niveau de qualité/sécurité.

### Credentials
Ne stocker aucun identifiant dans un conteneur. Pour faire passer des identifiants dans un conteneur, on utilise souvent les variables d'environnement. Il est aussi conseillé de penser à des technologies comme [JWT](https://jwt.io/) afin de proposer de l'authentification.

# Cas d'utilisation et particularités
Nous n'allons pas ici détailler précisément les cas d'utilisation de Docker mais plutôt essayer de les classer dans de grandes catégories.

## Pour les développeurs
* travail en local
  * facile de bootstrapper un environnement complet (notamment grâce aux nombreuses images pré-packagées que l'on peut trouver sur la toile)
  * rapidité pour bootstrapper cet environnement (surtout si l'on compare à un processus équivalent qui utiliserait des machines virtuelles)
* travail collaboratif
  * permet de faire abstraction de l'OS de chacun des développeurs d'une même équipe
  * un outil comme un registre permet de partager simplement les avancées

## Des services en production
* profiter de la grande capacité de scaling
  * les conteneurs sont légers et rapides
  * frameworks d'orchestration
* grand pouvoir de répétabilité (un conteneur est immuable)
* unification du packaging
* niveau d'isolation supplémentaire (hausse du niveau de sécurité)
  * limiter la surface d'attaque
* s'ancre parfaitement dans la philosophie cloud-native apps
  * hyperconvergence facilitée
  * micro-services
  * Infrastructure-as-code (IAC)

# Limites de Docker
## Visualisation des ressources de l'hôte

Du fait de certains mécanismes intrinsèquement liés au fonctionnement de Docker, il est possible de visualiser tout le pool de ressources (CPU, RAM, etc) même en cas de limitation de ces mêmes ressources.  
Par exemple, pour la RAM :

```shell
$ free -m
              total       utilisé      libre     partagé tamp/cache   disponible
Mem:           7901        4788         935         866        2177        1990
Swap:          2047           0        2047
$ docker run -m 16M alpine free -m
             total       used       free     shared    buffers     cached
Mem:          7901       6974        926        861        444       1650
-/+ buffers/cache:       4880       3021
Swap:         2047          0       2047
```
## Utilisation en production
Même si utiliser Docker en production permet d'accéder à de nombreux avantages, cela présente aussi plusieurs inconvénients **non négligeables** :
- il est préférable d'être maître de la technologie et connaître les mécanismes sous-jacents afin de pouvoir débugger la solution si besoin est
- les Dockerfiles doivent être **maintenus** (voir paragraphe dédié (mais pas encore écrit) :) )
- il est tout aussi nécessaire de garder en place des mécanismes de supervision, sauvegarde, sécurité, etc. comme à l'habitude
- même s'il est désormais très utilisé et compte un grand nombre de contributeurs, Docker est malgré tout encore un outil relativement jeune. Il est primordial de continuer à effectuer de la veille afin d'être averti des derniers bulletins de sécurité


# TODO
- Les configurations rédibitoires
  - compléter
- use cases
  - compléter
- limites
  - prod ?  compléter
- construction d'images
  - maintenir (no latest) etc
- relire
- réseau
  - exemple IPVLAN/MACVLAN (gif)
- standards
  - **TO MOVE :** Il peut être nécessaire de prendre connaissance de **la [CNI](https://github.com/containernetworking/cni) : un standard visant à décrire comment constuire les interfaces réseau pour des conteneurs.**
- docker compose ?
