## Historique et objectifs

### Contexte général

Mon objectif est de monter une démonstration concrète des possibilités DevOps autour des mainframes IBM, aussi bien sur z/OS que sur IBM i (AS/400).

Aujourd’hui, les discours officiels mettent en avant des environnements modernes (RAD for Z, VS Code, plugins spécifiques, etc.) que peu de développeurs ont réellement en main. Dans beaucoup d’équipes, les développeurs mainframe qui sont vraiment productifs travaillent encore exclusivement via un émulateur 3270 ou 5250, pilotent tout le cycle de vie “à la main” et restent des figures quasi intouchables : très compétents, mais avec des pratiques peu visibles, peu partageables.

Résultat : le responsable Études est souvent aveugle sur ce qui se passe réellement sur le mainframe, alors que sur les stacks Microsoft ou Unix, tout est tracé et instrumenté (Git, CI/CD, dashboards, logs, pipelines, etc.). Mon travail consiste justement à redonner cette visibilité sur les plateformes Z et i.

### Objectif 1 – Démonstration DevOps mainframe

Premier objectif : montrer, par l’exemple, que l’on peut appliquer des pratiques DevOps modernes au monde mainframe sans exiger que tous les développeurs soient équipés d’IDE “premium”.  
L’idée est de partir de la réalité actuelle (TSO/ISPF, 3270, 5250) et de faire émerger :

- des chaînes outillées de bout en bout (Git, CI, tests, déploiements),
- des scripts et automatisations pilotables depuis ISPF,
- des vues de contrôle comparables à ce qu’on a sur les applications open (Unix, Windows, cloud).

Le message clé : le mainframe n’est pas un bloc opaque, on peut le piloter et l’observer comme n’importe quelle autre plateforme.

### Objectif 2 – Cartographier et documenter le legacy

Deuxième objectif : documenter le système existant (z/OS ou iSeries) de manière vivante, structurée et exploitable.

Cela inclut :
- l’architecture globale et les grands domaines fonctionnels,
- les applications, briques techniques et interfaces,
- les chaînes batch,
- les transactions IMS et CICS,
- les bases DB2,
- les data lakes réels ou potentiels.

L’idée est de découper le mainframe en silos fonctionnels et techniques, de rendre visible :
- ce qui est bien compris,
- ce qui est flou ou mal connu,
- ce qui est carrément “caché”.

Cette documentation doit être **vivante** : la doc morte (PDF oubliés, Word jamais mis à jour) ne sert qu’à remplir un cimetière. L’objectif est de produire un reporting continu du legacy vivant : ce qui tourne aujourd’hui, ce qui change, ce qui consomme, ce qui bloque.

### Objectif 3 – Préparer les scénarios de sortie du Z ou de l’iSeries

Troisième objectif : préparer, de manière réaliste, les scénarios de sortie du mainframe (z/OS ou iSeries), quand cela a du sens.

Les options incluent :
- le portage vers des plateformes de type Unix ou cloud (TmaxSoft, écosystèmes COBOL/RPG sur Unix, etc.),
- la réduction de la “rente” matériel et logiciel IBM,
- l’accès à des compétences plus courantes (Linux, Java, cloud, DevOps).

L’idée n’est pas de “faire du replatforming pour le plaisir”, mais de transformer un ERP legacy massif en quelque chose de plus maîtrisable, avec un coût mieux contrôlé, en évitant la dépendance à des intégrateurs qui ont souvent intérêt à vendre des jours/homme plutôt qu’à trouver le chemin le plus court et le plus simple pour le client.

Une première étape naturelle est d’exploiter les assets de type **AWS Mainframe Modernization (Transform)**, en utilisant par exemple **Cardemo** comme patrimoine applicatif de référence, uniquement pour se former en reprenant leur chemin critique bien documenté.

### Note légale sur l’usage de z/OS 3.1 et Hercules

Mes travaux autour d’Hercules et de l’image z/OS 3.1 trouvée sur archive.org (image de test / developer) sont **strictement orientés démonstration et formation**.  
Ils ne visent pas à pirater des environnements clients ni à contourner des licences.

L’objectif est de disposer d’un bac à sable technique pour illustrer des scénarios de DevOps mainframe, de documentation, de cartographie et de migration.

Dans un contexte client réel, il existe des solutions **légales** et supportées pour exécuter ou migrer des charges z/OS en cloud (chez IBM ou chez des partenaires comme Popup Mainframe, ACMI, etc.), qui permettent de transporter les idées et les patterns démontrés ici vers une plateforme conforme au cadre contractuel et réglementaire du client.


### Historique des plateformes de test

Mes premiers essais ont été réalisés sur Windows 11, avec :

- z/OS 3.1 IPL correct,
- pilotage via la console Hercules,
- réseau configuré en QETH (1500.3) en mode QDIO sur une interface `tap0` en layer 3,
- côté Windows, usage de CTCI-WIN comme passerelle réseau.

Cette configuration a permis de valider certains scénarios, mais avec de gros problèmes de stabilité réseau et un comportement très aléatoire. De plus, CTCI-WIN ne fonctionne pas sur les cartes Wi-Fi, ce qui m’a obligé à recourir à un expander branché en Ethernet, puis à utiliser l’adaptateur filaire comme passerelle dédiée pour Hercules.

Faute de support et de fiabilité suffisante, j’ai décidé de considérer cette configuration Windows 11 comme une **base historique** uniquement, et de basculer sur un environnement Unix plus robuste : Ubuntu sur AWS.

Le passage sur AWS EC2 apporte :

- la souplesse du “on demand”,
- un dimensionnement adapté (4 vCPU, 16 Go de RAM, dont 12 Go alloués à z/OS),
- un volume dédié monté sur `/mnt/zos31` pour supporter les DASD Hercules.


### Plateforme de démonstration actuelle (AWS / Ubuntu)

- **Cloud provider** : AWS
- **Instance EC2** : 4 vCPU, 16 Go RAM
- **Système hôte** : Ubuntu 22.04 (64 bits)
- **CPU** (hôte) :
  - Architecture : x86_64
  - Modes : 32-bit, 64-bit
  - Byte order : Little Endian
- **Stockage** :
  - `nvme0n1` (100 Go) : disque principal, racine `/`, `/boot`, `/boot/efi`
  - `nvme1n1` (100 Go) : disque dédié, partition `nvme1n1p1` montée sur `/mnt/zos31`
    - Ce volume contient :
      - les fichiers DASD Hercules (`*.CCKD`)
      - le fichier de configuration `hercules.cnf`
      - les scripts d’auto-IPL (`autoipl.rc`)
      - les logs (`logzOS.txt`, `hercules.log`)

### Configuration Hercules (z/OS 3.1)

Les points structurants de la configuration Hercules sont :

- **Architecture** : `ARCHLVL z/ARCH`
- **Facilités activées** : un ensemble de `FACILITY ENABLE` nécessaires au bon fonctionnement de z/OS 3.1 (ASN_LX_REUSE, PFPO, DFP_PACK_CONV, etc.), avec en complément des `FAC ENA 054 129 130 134 135` exécutés au démarrage pour les extensions vectorielles.
- **Ressources CPU** :
  - `CPUMODEL 8562` (z15 simulé)
  - `CPUSERIAL 02B7F8`
  - `MODEL T02 S03`
  - `MAINSIZE 12G` (mémoire allouée à z/OS)
  - `NUMCPU 8` (CP simulés)
- **LPAR** :
  - `LPARNAME VS01`
  - `LOADPARM DE28K2M.` pour l’IPL z/OS 3.1
- **Volumes DASD (3390)** : mappés sur le volume `/mnt/zos31`, par exemple :
  - `DE26 /mnt/zos31/D31VS1.CCKD` (vol système)
  - `DE27 /mnt/zos31/Z31VS1.CCKD` (vol IPL)
  - `DE28 /mnt/zos31/OPEVS1.CCKD`
  - `DE33 /mnt/zos31/USRVS1.CCKD`
  - etc.
- **Console et réseau** :
  - Console 3270 : `CNSLPORT 0.0.0.0:3270`
  - OSA/QETH : device `1500 QETH iface /dev/net/tun` + `1500.3 QETH ipaddr 10.10.10.2/24 mtu 1500` (réseau détaillé dans une fiche réseau dédiée).

### Services systemd associés

Plusieurs services systemd structurent le démarrage de la plateforme :

- **Hercules** : `/etc/systemd/system/hercules.service`
  - Démarre Hercules avec la configuration `/mnt/zos31/hercules.cnf`
  - Alimente un log `/mnt/zos31/hercules.log`
  - Envoie un `quit` propre via `nc` lors du `systemctl stop hercules`

- **Console 3270** : `/etc/systemd/system/c3270.service`
  - Démarre une session `c3270` dans `screen` via `start_console_screen.sh`
  - Rattaché à `hercules.service` (After/Requires)

- **Interface tun0** : `/etc/systemd/system/tun0.service`
  - Type `oneshot`, `RemainAfterExit=yes`
  - Attend l’apparition de l’interface `tun0` (jusqu’à 60s)
  - Configure :
    - adresse IP `10.10.10.1/24`
    - mise en UP de l’interface
    - route `10.10.10.0/24` via `tun0`

Côté poste Windows 11, un couple de scripts `ztun-start.cmd` / `ztun-stop.cmd` crée un tunnel SSH vers l’EC2 pour exposer en local :

- TN3270 : `localhost:3270` → `10.10.10.2:23`  
- z/OSMF : `https://localhost:10443` → `10.10.10.2:10443`
