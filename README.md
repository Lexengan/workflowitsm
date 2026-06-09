# glpiworkflow — Moteur de workflows ITSM pour GLPI 11

Plugin GLPI 11 permettant de définir, visualiser et exécuter des **workflows ITSM séquentiels automatisés** directement dans GLPI, sans développement supplémentaire. Les workflows déclenchent des actions conditionnelles sur les tickets (création de tâches, ajout de suivis, résolution, affectation de matériel) en fonction de l'évolution de l'objet ITIL et des conditions métier configurées.

---

## Sommaire

- [Rôle et objectifs](#rôle-et-objectifs)
- [Architecture](#architecture)
- [Installation](#installation)
- [Utilisation](#utilisation)
  - [Accéder aux workflows](#accéder-aux-workflows)
  - [Créer un workflow](#créer-un-workflow)
  - [Configurer une étape](#configurer-une-étape)
  - [Définir les conditions](#définir-les-conditions)
  - [Enregistrer et activer](#enregistrer-et-activer)
- [Actions disponibles](#actions-disponibles)
- [Conditions disponibles](#conditions-disponibles)
- [Cas d'usage : workflow onboarding](#cas-dusage--workflow-onboarding)
- [Exécution et planification](#exécution-et-planification)
- [Gestion des droits](#gestion-des-droits)
- [Référence technique](#référence-technique)

---

## Rôle et objectifs

`glpiworkflow` ajoute un moteur de **machine à états persistant** à GLPI. Là où les règles métier GLPI agissent une seule fois à la création ou à la modification d'un objet, un workflow peut s'étendre sur toute la durée de vie d'un ticket et réagir à chaque évolution : tâche terminée, statut changé, acteur ajouté, matériel lié.

Les objectifs principaux sont :

- **Orchestrer des processus multi-étapes** sans intervention manuelle sur l'outil (les tâches apparaissent automatiquement au bon moment, affectées au bon groupe).
- **Garantir la traçabilité** de chaque action déclenchée (journal d'exécution par étape).
- **Configurer sans coder** grâce au designer visuel drag-and-drop.
- **Réutiliser les gabarits de tâches** GLPI existants dans les étapes du workflow.

---

## Architecture

Le plugin repose sur quatre couches :

| Couche | Rôle |
|--------|------|
| **Designer** | Interface drag-and-drop de configuration des étapes et conditions |
| **Registres** | Catalogue des actions et conditions disponibles (activables/désactivables) |
| **Moteur d'exécution** | Machine à états qui évalue les conditions et déclenche les actions |
| **Actions automatiques GLPI** | Deux crons qui déclenchent et font progresser les exécutions |

Les données sont stockées dans sept tables dédiées (`glpi_plugin_glpiworkflow_*`) sans contrainte de clé étrangère sur le noyau GLPI.

---

## Installation

**Prérequis :** GLPI 11.0.x, PHP ≥ 8.1, MariaDB 10.11.

1. Déposer le dossier `glpiworkflow/` dans `/var/www/glpi/plugins/`.
2. Vider le cache : `php /var/www/glpi/bin/console cache:clear`.
3. Redémarrer le conteneur ou le service PHP.
4. Aller dans **Configuration > Plugins**, activer **Workflows ITSM**.
5. Vérifier que les deux actions automatiques sont programmées dans **Administration > Actions automatiques** : `executeScheduledWorkflows` (1 min) et `resumePendingWorkflows` (5 min).

> **Note :** Après activation, les droits sont automatiquement créés sur tous les profils existants (valeur 31 = tous les droits). Ajustez-les dans **Administration > Profils > onglet Workflows ITSM**.

---

## Utilisation

### Accéder aux workflows

Le plugin est accessible depuis le menu **Plugins > Workflows ITSM**.

![Menu Plugins > Workflows ITSM](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/01_menu.PNG)

La page d'accueil liste tous les workflows créés avec leur type ITIL, leur statut (Actif/Inactif) et les actions disponibles : Designer, Modifier, Supprimer.

![Liste des workflows](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/02_liste_workflows.PNG)

### Créer un workflow

Cliquer **+ Créer un workflow** pour définir le nom, le type ITIL (Ticket, Problème ou Changement) et l'entité de rattachement. Cliquer ensuite sur l'icône Designer (réseau) pour ouvrir l'éditeur visuel.

### Configurer une étape

Le designer présente trois zones : la palette d'actions à gauche, le canvas central avec les étapes ordonnées, et le panneau de configuration à droite.

![Designer — vue globale avec 6 étapes](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/03_designer_global.PNG)

Pour configurer une étape, cliquer sur l'icône engrenage de la ligne correspondante. Le panneau de droite affiche les paramètres de l'action choisie.

![Configuration d'une étape avec modèle de tâche](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/04_config_etape.PNG)

Les paramètres disponibles varient selon l'action :

- **Modèle de tâche** : sélectionner un gabarit existant dans GLPI (catégorie, durée, contenu pré-remplis).
- **Contenu** : texte libre qui surcharge le modèle si renseigné.
- **Groupe assigné** : groupe fixe ou « → Groupe attribué du ticket (dynamique) ».
- **Comportement si conditions non remplies** : `Bloquante` (le workflow attend) ou `Conditionnelle` (l'étape est sautée).

### Définir les conditions

Cliquer sur l'icône filtre (entonnoir) d'une étape pour ouvrir la modale de conditions. Chaque étape peut avoir une ou plusieurs conditions combinées par opérateur `ET` (toutes vraies) ou `OU` (au moins une vraie).

![Modale de conditions — Catégorie ITIL et statut de tâche](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/05_conditions.PNG)

### Enregistrer et activer

Cliquer **Enregistrer** en haut à droite pour persister l'ensemble du workflow (étapes, configurations, conditions, ordre). Le bouton statut dans la liste permet de basculer le workflow entre Actif et Inactif sans le supprimer.

---

## Actions disponibles

| Action | Description |
|--------|-------------|
| **Ajouter une tâche** | Crée une tâche sur le ticket, optionnellement depuis un gabarit GLPI. Supporte l'affectation à un groupe fixe ou au groupe attribué du ticket. |
| **Ajouter un suivi** | Ajoute un message de suivi interne ou public sur le ticket. |
| **Ajouter une solution et résoudre** | Ajoute une solution et passe le ticket en statut Résolu. |
| **Changer le statut du ticket** | Modifie le statut du ticket vers une valeur configurée. |
| **Affecter à un groupe** | Affecte le ticket à un groupe spécifique. |
| **Affecter le matériel au demandeur** | Met à jour le champ `users_id` du matériel lié au ticket en le définissant sur le demandeur (type REQUESTER). |
| **Affecter le matériel au bénéficiaire (observateur)** | Met à jour le champ `users_id` du matériel lié au ticket en le définissant sur l'observateur (type OBSERVER). Cas d'usage : onboarding où demandeur ≠ bénéficiaire. |
| **Ouvrir un ticket demande** | Crée un nouveau ticket de type demande. |

---

## Conditions disponibles

Les conditions sont regroupées par domaine dans le catalogue.

### Demandes de service

| Condition | Paramètres |
|-----------|------------|
| Catégorie ITIL particulière | `itilcategories_id` — vérifie la catégorie du ticket à sa création |
| Statut de tâche | `task_scope` (Au moins N, Au moins une, Toutes, Dernière), `min_count`, `status` |
| Ticket ouvert depuis (heures) | `hours` — durée depuis l'ouverture |
| SLA bientôt dépassé | `minutes_before` — fenêtre d'alerte avant dépassement |
| SLA dépassé | — |

### Incidents

| Condition | Paramètres |
|-----------|------------|
| Priorité au moins | `priority` |
| Type de ticket | `ticket_type` (Incident / Demande) |
| Ticket réouvert | — |

### Gestion du parc

| Condition | Paramètres |
|-----------|------------|
| Matériel lié | `itemtype` — type de matériel présent dans les éléments liés |
| Garantie expirée | `itemtype` |
| Âge du matériel | `itemtype`, `max_age_years` |

### Base de connaissances

| Condition | Paramètres |
|-----------|------------|
| Article KB lié | — |

> Le catalogue complet est accessible depuis **Plugins > Workflows ITSM > Catalogue d'actions**. Chaque action et condition peut être activée ou désactivée indépendamment.

---

## Cas d'usage : workflow onboarding

Ce workflow illustre les possibilités du plugin sur un processus réel en six étapes.

**Contexte :** un manager ou un responsable RH crée un ticket de catégorie « Demandes > Accès > Demande onboarding » pour un nouveau collaborateur. Plusieurs équipes interviennent séquentiellement.

### Déclenchement

À la création du ticket, l'étape 1 évalue la condition `CategoryCreatedCondition` (catégorie = onboarding). La première tâche apparaît immédiatement, affectée à l'équipe N2 Sécurité & Accès.

![Ticket créé — première tâche générée automatiquement](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/06_ticket_tache1.PNG)

### Progression par le cron

Le cron `executeScheduledWorkflows` (1 minute) maintient les exécutions actives. À chaque passage, il évalue les conditions des étapes en attente.

![Action automatique executeScheduledWorkflows](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/07_cron.PNG)

### Étape 2 — Préparation du poste

Le technicien crée le compte AD, réalise la synchro LDAP, puis passe la tâche à **Fait**. Au passage suivant du cron, la condition `TaskStatusCondition (count_at_least, min_count=1, Fait)` devient vraie : la tâche « Préparation poste de travail » est créée, affectée au **groupe attribué du ticket** (dynamique).

![Tâche 2 créée après passage de la tâche 1 à Fait](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/08_ticket_tache2.PNG)

### Étape 3 — Prise de RDV

Après passage de la tâche 2 à Fait (`min_count=2`), la tâche « Prise de rendez-vous » apparaît. L'observateur **Rousseau Claire** (le nouvel arrivant, ajouté manuellement par le technicien après synchro LDAP) est visible dans les acteurs.

![Tâche 3 créée — observateur ajouté](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/09_ticket_tache3_observateur.PNG)

### Étape 4 — Affectation du matériel

L'ordinateur **INJECT-LAPTOP-EVA-001** a été lié au ticket dans les éléments liés. À l'étape 4 (`min_count=3`), `AssignAssetToObserverAction` lit l'observateur du ticket et met à jour `users_id` sur l'ordinateur.

![Ordinateur lié au ticket visible dans les éléments](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/10_ticket_ordinateur_lie.PNG)

### Étape 5 — Résolution automatique

Dans la même passe du cron, l'étape 5 s'enchaîne immédiatement : la solution « Comptes et habilitations créés. Matériel préparé et mis à disposition du nouvel arrivant. » est ajoutée et le ticket passe en **Résolu**.

![Ticket résolu avec solution automatique](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/11_ticket_resolu.PNG)

### Résultat final

Le profil utilisateur de Rousseau Claire affiche l'ordinateur **INJECT-LAPTOP-EVA-001** dans ses éléments utilisés — affectation réalisée automatiquement par le workflow sans intervention manuelle.

![Profil utilisateur — ordinateur affecté](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/12_utilisateur_materiel.PNG)

---

## Exécution et planification

Le moteur fonctionne en mode **asynchrone** : les actions ne sont pas exécutées en temps réel mais lors des passages du cron.

| Action automatique | Fréquence | Rôle |
|--------------------|-----------|------|
| `executeScheduledWorkflows` | 1 minute | Démarre les nouvelles exécutions au déclenchement |
| `resumePendingWorkflows` | 5 minutes | Fait progresser les exécutions en attente |

**Logique d'enchaînement séquentiel :** les étapes bloquantes utilisent la condition `TaskStatusCondition` avec le scope `count_at_least` et des seuils croissants (`min_count = 1, 2, 3…`). Chaque étape attend qu'un palier supérieur de tâches terminées soit atteint, garantissant l'ordre sans cibler une tâche précise.

**Anti-doublon :** le moteur vérifie qu'aucune exécution n'existe déjà pour un couple workflow + ticket avant d'en créer une nouvelle, évitant les déclenchements multiples lors de la création de ticket (qui génère souvent un `item_add` suivi d'un `item_update`).

> **Important :** entre chaque étape, le délai maximum est la fréquence du cron `resumePendingWorkflows` (5 minutes par défaut, configurable). Pour des enchaînements plus réactifs, réduire cette fréquence dans **Administration > Actions automatiques**.

---

## Gestion des droits

Les droits du plugin sont configurables par profil dans **Administration > Profils > onglet Workflows ITSM**.

| Droit | Effet |
|-------|-------|
| **Lecture** | Consulter la liste et les logs des workflows |
| **Modification** | Modifier les paramètres d'un workflow existant |
| **Création** | Créer de nouveaux workflows |
| **Suppression** | Supprimer un workflow (cascade sur étapes et exécutions) |
| **Purge** | Purge définitive |

---

## Référence technique

### Schéma de base de données

| Table | Contenu |
|-------|---------|
| `glpi_plugin_glpiworkflow_workflows` | Définition des workflows (nom, type ITIL, actif/inactif) |
| `glpi_plugin_glpiworkflow_steps` | Étapes (action, config JSON, mode, ordre) |
| `glpi_plugin_glpiworkflow_conditions` | Conditions par étape (classe, paramètres JSON) |
| `glpi_plugin_glpiworkflow_executions` | Exécutions en cours ou terminées par ticket |
| `glpi_plugin_glpiworkflow_step_executions` | Journal d'exécution par étape (done/skipped/error) |
| `glpi_plugin_glpiworkflow_schedules` | Planifications optionnelles par workflow |
| `glpi_plugin_glpiworkflow_catalog` | État d'activation des actions/conditions |

### Valeurs de référence

| Constante | Valeur | Source |
|-----------|--------|--------|
| `Planning::INFO` | 0 | `Planning.php:83` |
| `Planning::TODO` | 1 | `Planning.php:84` |
| `Planning::DONE` | 2 | `Planning.php:85` |
| `CommonITILActor::REQUESTER` | 1 | `CommonITILActor.php:47` |
| `CommonITILActor::ASSIGN` | 2 | `CommonITILActor.php:48` |
| `CommonITILActor::OBSERVER` | 3 | `CommonITILActor.php:49` |

### Ajouter une action personnalisée

1. Créer une classe dans `src/Action/` héritant de `AbstractAction`.
2. Implémenter `execute()`, `getConfigFields()`, `getLabel()`, `getIcon()`.
3. Déclarer la classe dans `WorkflowActionRegistry::allClasses()`.
4. Activer l'action depuis le catalogue.

### Ajouter une condition personnalisée

1. Créer une classe dans `src/Condition/` héritant de `AbstractCondition`.
2. Implémenter `evaluate()`, `getParamFields()`, `getLabel()`, `getDomain()`.
3. Déclarer la classe dans `WorkflowConditionRegistry::allClasses()`.

---

*Plugin développé pour GLPI 11.0.x — Référence MO-GLPI-PLUGIN-002 v2, SPEC-GLPI-WF-001 v1.0*
