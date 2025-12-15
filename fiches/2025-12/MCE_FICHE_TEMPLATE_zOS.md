# Fiche de maintenance z/OS – Cycle Dev → Recette → Prod

> **Objectif** : standardiser, depuis z/OS (ISPF), la création et le suivi d’une maintenance (MCE) **de bout en bout** : création → dev → tests → commit/push Git → build & déploiement automatisés sur environnements cibles (au minimum Recette utilisateur + Production).

---

## 1) Identité de la fiche

- **MCE_NBR**            : `MCE-YYYY-NNNN`  
- **Titre**              :  
- **Auteur (TSO/nom)**   :  
- **Application / Domaine** :  
- **Type**               : ☐ Correctif ☐ Évolutif ☐ Urgent ☐ Technique  
- **Priorité**           : ☐ P1 ☐ P2 ☐ P3  
- **Statut**             : `DRAFT | IN_DEV | READY_TO_PUSH | CI_RUNNING | IN_UAT | READY_FOR_PROD | IN_PROD | CLOSED | ABORTED`
- **Environnements cibles** : ☐ Z-DEV (VS01) ☐ Z-UAT (Recette) ☐ Z-PROD  
- **Références**         : (incident, demande métier, mail, etc.)

---

## 2) Dates & heures (trace complète)

- **Création fiche**     : `YYYY-MM-DD HH:MM` (Europe/Paris)  
- **Début dev**          :  
- **Fin dev**            :  
- **Début tests dev**    :  
- **Fin tests dev**      :  
- **Commit(s)**          :  
- **Push Git**           :  
- **Début CI/CD**        :  
- **Fin CI/CD**          :  
- **Début UAT**          :  
- **Fin UAT**            :  
- **Go/No-Go prod**      :  
- **Déploiement prod**   :  
- **Clôture**            :  

---

## 3) Périmètre / impact

### 3.1 Contexte
- Situation actuelle :  
- Problème / besoin :  
- Résultat attendu :  

### 3.2 Analyse d’impact
- Impacts fonctionnels :  
- Impacts techniques :  
- Datasets / modules impactés :  
- Risques :  
- Backout (retour arrière) possible ? ☐ Oui ☐ Non  
- Fenêtre de déploiement proposée :  

---

## 4) Composants z/OS concernés (sélection “à commiter”)

> **Règle** : cette section doit refléter exactement ce que le dev choisira dans ISPF pour **commit/push**.

| Type | Nom (DSN ou membre) | Détails | À commiter |
|---|---|---|---|
| COBOL | `IBMUSER.COBOL(????)` | | ☐ |
| COPY | `IBMUSER.COPY(????)` | | ☐ |
| JCL | `IBMUSER.JCL(????)` | | ☐ |
| BMS/MAP | `...` | | ☐ |
| PROC | `...` | | ☐ |
| DB2 | `...` | (DDL / BIND / packages) | ☐ |
| CICS | `...` | (RDO / CSD / PPT/PGM) | ☐ |
| IMS | `...` | | ☐ |
| Autre | `...` | | ☐ |

### 4.1 Branch / tag / stratégie Git
- **Repo** :  
- **Branch de travail** : `feature/MCE-YYYY-NNNN` (recommandé)  
- **Convention commit** : `MCE-YYYY-NNNN: <message>`  
- **Tag release (UAT/PROD)** : `uat/MCE-YYYY-NNNN` puis `prod/MCE-YYYY-NNNN`

---

## 5) Étapes standard du cycle (à compléter au fil de l’eau)

### 5.1 Création (depuis z/OS)
- [ ] Saisie des champs (section 1 + 2 + 3)
- [ ] Sélection du périmètre (section 4)
- [ ] Génération artefacts initiaux :
  - [ ] fichier fiche : `/u/ibmuser/mce/MCE-YYYY-NNNN.md` (ou dataset équivalent)
  - [ ] “snapshot” initial des composants (liste + empreintes si dispo)
- [ ] Statut → `IN_DEV`

### 5.2 Pull / synchro Git → z/OS (baseline)
> Utilise votre flow actuel via **SYS1.SBPXEXEC(GITXPORT)** + panel ISPF (regex).

- Regex utilisée : `^(HELLO|HELLOJR|COB.*)$` *(exemple)*
- Hôte Linux : `10.10.10.1`
- User Linux : `ubuntu`
- Script : `/u/ibmuser/devops/gitexport.sh`
- [ ] Baseline importée dans les PDS de travail
- [ ] Statut inchangé (reste `IN_DEV`)

### 5.3 Développement
- [ ] Modifications effectuées
- [ ] Revue locale (pair / auto-contrôle)
- Preuves (optionnel) : captures, logs, listings, etc.

### 5.4 Tests DEV (JCL / CICS / DB2 / batch)
- [ ] Jeux de tests exécutés
- [ ] Résultats validés
- Liens vers outputs (spool / logs / datasets) :
  - Job(s) :  
  - Spool(s) :  
  - Fichiers :  

➡️ **Statut →** `READY_TO_PUSH`

### 5.5 Commit & Push (choix précis des éléments)
- [ ] Sélection des éléments à commiter (section 4, cases cochées)
- [ ] Génération diff (si disponible) / listing des changements
- [ ] Commit réalisé : `<hash>`  
- [ ] Push réalisé : `<branch>`  

➡️ **Statut →** `CI_RUNNING`

---

## 6) CI/CD déclenché par Git (z/OS cible)

> **Principe** : le push déclenche un workflow qui va exécuter sur l’environnement cible :
> 1) **pull** du repo / artefacts, 2) **build** (compile/link/bind), 3) **smoke tests**, 4) **packaging**, 5) **déploiement**.

### 6.1 Matrice d’environnements (exemple)
| Environnement | LPAR | Objectif | Déclencheur |
|---|---|---|---|
| DEV | VS01 | Dév / tests unitaires | push sur feature |
| UAT | (à définir) | Recette utilisateur | tag `uat/*` ou merge sur `release/uat` |
| PROD | (à définir) | Production | tag `prod/*` + approbation |

### 6.2 Résultats pipeline
- Run ID / lien :  
- Étapes OK : ☐ Pull ☐ Build ☐ Tests ☐ Packaging ☐ Deploy  
- Logs principaux :  
- Artifacts générés : (load modules, DB2 packages, etc.)

➡️ **Statut →** `IN_UAT` (si déployé UAT) ou `READY_FOR_PROD` (si UAT validée)

---

## 7) Recette utilisateur (UAT)

- [ ] Plan de tests UAT validé
- [ ] Résultat UAT : ☐ OK ☐ KO  
- Décisions / actions correctives :  

➡️ **Statut →** `READY_FOR_PROD` (si OK)

---

## 8) Déploiement Production

### 8.1 Go/No-Go
- Date/heure comité :  
- Participants :  
- Décision : ☐ GO ☐ NO-GO  
- Risques résiduels acceptés :  

### 8.2 Plan de déploiement PROD (checklist)
- [ ] Communication (début)  
- [ ] Sauvegarde/backup (si applicable)  
- [ ] Déploiement automatisé (pipeline)  
- [ ] Vérifs post-déploiement (smoke tests)  
- [ ] Communication (fin)  

### 8.3 Backout (si KO)
- Condition de déclenchement :  
- Étapes retour arrière :  
- Validation retour arrière :  

➡️ **Statut →** `IN_PROD` puis `CLOSED`

---

## 9) Journal horodaté (audit)

| Date/Heure | Acteur | Étape | Détail / preuve |
|---|---|---|---|
| YYYY-MM-DD HH:MM | | Création | |
|  |  | Baseline import | |
|  |  | Tests DEV | |
|  |  | Commit/Push | |
|  |  | CI/CD | |
|  |  | UAT | |
|  |  | PROD | |
|  |  | Clôture | |

---

## 10) Annexes
- Diffs / listings :  
- Références techniques (JCL, PROC, paramètres) :  
- Notes :  

