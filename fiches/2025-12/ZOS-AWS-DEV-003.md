# ZOS-AWS-DEV-003 – Pilotage DevOps depuis ISPF (z/OS 3.1 sur AWS)

## Métadonnées
- Date       : 2025-12-XX
- Contexte   : Démonstrateur DevOps z/OS 3.1 sur Hercules / AWS
- Thèmes     : ISPF, JCL, BPXBATCH, USS, Git, Zowe, CI/CD
- Niveau     : Avancé
- Dépend de  : ZOS-AWS-BASE-001, ZOS-AWS-NET-002
- Statut     : V1 – architecture et premiers flux

---

## 1. Objectif

Montrer qu’on peut piloter un **cycle DevOps moderne** (Git, scripts, CI)  
**directement depuis ISPF**, sans imposer d’IDE exotique aux développeurs z/OS :

- les devs restent dans leur **3270** habituel,
- mais derrière, tout est **tracé** (Git), **automatisé** (scripts, BPXBATCH)  
  et **intégrable** dans des pipelines CI/CD (Zowe, GitHub, Ansible, etc.).

---

## 2. Environnement et principes

### 2.1 Environnement technique

- z/OS 3.1 VSI sur Hercules (AWS EC2 Ubuntu)
- User de travail : `IBMUSER`
- Stockage USS principal : `/u/ibmuser` et sous-arborescence `/u/ibmuser/devops`
- Git distant : GitHub / Gitea (repos COBOL, JCL, etc.)
- Outils :
  - **BPXBATCH** pour lancer des scripts Unix depuis JCL,
  - **Git** côté USS pour versionner les sources,
  - **Zowe CLI** (objectif) pour orchestrer les jobs / transferts,
  - **ISPF** comme cockpit unique côté développeur.

### 2.2 Principes d’architecture

1. **ISPF reste le centre du monde**  
   Le développeur voit juste une **ligne de menu** ou une **option de panel** :
   - `1 – Pull depuis Git`
   - `2 – Push vers Git`
   - `3 – Lancer la chaîne CI`
   - `4 – Voir les résultats (SYSOUT / logs)`

2. **JCL comme “bridge” vers le monde Unix / Git**  
   - JCL simple, exécution via BPXBATCH :
     - `SH /u/ibmuser/devops/get_from_git.sh`
     - `SH /u/ibmuser/devops/push_to_git.sh`

3. **Scripts USS pour le travail “sale”**  
   - `get_from_git.sh` : clone / pull / export des sources vers des PDS
   - `push_to_git.sh` : récupère les PDS modifiés et commit/push sur Git
   - plus tard : scripts qui parlent à Zowe, Ansible, CI, etc.

4. **Traçabilité**  
   - chaque action ISPF → un job z/OS → des logs JES + logs USS

---

## 3. Chaîne “Pull depuis Git” (exemple IBMUSERG)

### 3.1 JCL d’entrée (exemple IBMUSERG)


Job lancé depuis ISPF (menu “DevOps” ou option `=6` par ex.) :

```jcl
//IBMUSERG JOB (ACCT),'COBOL GET',CLASS=A,MSGCLASS=X,
//         MSGLEVEL=(1,1),REGION=0M,NOTIFY=&SYSUID
//*
//*  STEP 1 : RÉCUPÉRER LES SOURCES DEPUIS GIT
//*
//STEP1   EXEC PGM=BPXBATCH,
//        PARM='SH /u/ibmuser/devops/get_from_git.sh'
//STDOUT  DD SYSOUT=*
//STDERR  DD SYSOUT=*
//STDPARM DD DUMMY

```markdown
---
### 3.2 Script /u/ibmuser/devops/get_from_git.sh

#!/bin/sh
# 1. Aller dans le repo
cd /u/ibmuser/devops/zos-cobol-repo || exit 8

# 2. Mise à jour depuis Git
git pull origin main

# 3. Export vers les PDS (ex: via cpio/pax ou utilitaire)
#    Exemple logique :
#    ./export_to_pds.sh COBOL  "IBMUSER.COBOL"
#    ./export_to_pds.sh JCL    "IBMUSER.JCL"

exit 0
	
## 4. Chaîne “Push vers Git” (sources → Git)

### 4.1 JCL d’entrée (exemple IBMUSERP)
```jcl
//IBMUSERP JOB (ACCT),'COBOL PUSH',CLASS=A,MSGCLASS=X,
//         MSGLEVEL=(1,1),REGION=0M,NOTIFY=&SYSUID
//*
//*  STEP 1 : EXPORTER LES PDS VERS USS ET PUSH GIT
//*
//STEP1   EXEC PGM=BPXBATCH,
//        PARM='SH /u/ibmuser/devops/push_to_git.sh'
//STDOUT  DD SYSOUT=*
//STDERR  DD SYSOUT=*
//STDPARM DD DUMMY
```
# 4.2 Script /u/ibmuser/devops/push_to_git.sh (vue logique)

#!/bin/sh
```sh 
cd /u/ibmuser/devops/zos-cobol-repo || exit 8

# 1. Importer les PDS vers USS
#    ./import_from_pds.sh COBOL "IBMUSER.COBOL"
#    ./import_from_pds.sh JCL   "IBMUSER.JCL"

# 2. Checker les différences
git status

# 3. Commit + push (ici simplifié, à sécuriser)
git add .
git commit -m "Modifs depuis ISPF (IBMUSER)"
git push origin main

exit 0
```
#  5. Intégration avec Zowe / CI (cible V2)

## 5.1 Rôle de Zowe CLI

À terme, l’idée est de ne plus faire seulement Git ↔ PDS, mais aussi :

soumettre des jobs de build / tests :

compilation COBOL (IGYCRCTL / IGYWCLG)

exécution de batch de tests

récupérer les SYSOUT et les injecter dans des logs artefacts CI.

Scénario type depuis un script USS :

#Exemple logique, à adapter quand Zowe est finalisé
zowe zos-jobs submit local-file build_cobol.jcl --wfo
zowe zos-jobs view all-spool-content JOB00123 > logs/build_JOB00123.txt

##  5.2 Intégration avec CI externe (GitHub / GitLab / Jenkins)

Deux approches :

CI déclenchée par push Git

Le script push_to_git.sh fait un git push

Un workflow GitHub Actions / GitLab CI récupère :

les sources

éventuellement des scripts Zowe pour parler au z/OS Hercules

Résultat : tests, analyse statique, packaging…

CI déclenchée depuis ISPF

Un JCL “CI_TRIGGER” appelle un script USS :

qui appelle GitHub API ou Ansible pour lancer un pipeline

Avantage : le dev 3270 “voit” le pipeline comme une étape z/OS.

#  6. ISPF comme cockpit (V2/V3)
## 6.1 CLIST / REXX de menu

Objectif (à faire dans une version suivante) : un menu ISPF unique :

           ZOS DEVOPS – PILOTAGE GIT / CI

   1  Pull (Git → PDS)
   2  Push (PDS → Git)
   3  Lancer la chaîne CI
   4  Voir les résultats CI
   X  Sortie


Chaque option :

soumet le JCL correspondant (IBMUSERG, IBMUSERP, IBMUSERC…)

affiche le JOBID

propose d’ouvrir directement le SYSOUT JES via SDSF.

# 7. Résultat actuel (V1)

✅ Ce qui est déjà en place ou cadré :

Environnement z/OS 3.1 stable (Hercules / AWS)

Accès ISPF complet

BPXBATCH opérationnel pour lancer des scripts USS

Scripts logiques get_from_git.sh / push_to_git.sh définis

Structure de JCL IBMUSERG / IBMUSERP claire

GitHub prêt à accueillir les sources (repos perso)

🧩 À venir (V2/V3) :

scripts d’export/import PDS ↔ USS robustes

intégration Zowe CLI

panélisation ISPF (menu DevOps)

pipeline CI externe (GitHub Actions, etc.)

# 8. Leçons et intérêt pour un client

On peut garder la culture 3270 tout en apportant :

de la visibilité au responsable Études,

de la traçabilité (Git),

de l’automatisation (scripts, CI).

La démo sur Hercules/AWS montre :

ce qui est facile à transposer sur un Z client,

comment sortir du “guru 3270 opaque” pour aller vers un modèle ouvert.

Cette approche prépare :

la cartographie détaillée du legacy,

les scénarios de migration (TmaxSoft, refonte, replatforming),

sans attendre l’arrivée miraculeuse d’un IDE unique pour tous.

# 9. Réutilisation

Base de support de formation “DevOps pour dev mainframe”.

Démonstrateur pour un POC client (z/OS sur Z ou sur cloud partenaire).

Matrice de travail pour :

d’autres langages (PL/I, RPG iSeries),

d’autres environnements (IMS, CICS, DB2, batch critique).

