# glpiworkflow — Moteur de workflows ITSM pour GLPI 11

**Version** : 1.6.5 | **Référence** : SPEC-GLPI-WF-001 v1.0 | **MO** : MO-GLPI-PLUGIN-002 v2

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
- [Fonctionnalités](#fonctionnalités)
- [Actions disponibles](#actions-disponibles)
- [Conditions disponibles](#conditions-disponibles)
  - [Demandes de service](#demandes-de-service)
  - [Incidents](#incidents)
  - [Gestion du parc](#gestion-du-parc)
  - [Base de connaissances](#base-de-connaissances)
- [Enchaînement séquentiel fiable](#enchaînement-séquentiel-fiable)
- [Cas d'usage : workflow onboarding](#cas-dusage--workflow-onboarding)
  - [Déclenchement](#déclenchement)
  - [Progression par le cron](#progression-par-le-cron)
  - [Étape 2 — Préparation du poste](#étape-2--préparation-du-poste)
  - [Étape 3 — Prise de RDV](#étape-3--prise-de-rdv)
  - [Étape 4 — Affectation du matériel](#étape-4--affectation-du-matériel)
  - [Étape 5 — Résolution automatique](#étape-5--résolution-automatique)
  - [Résultat final](#résultat-final)
- [Exécution et planification](#exécution-et-planification)
- [Gestion des droits](#gestion-des-droits)
- [Référence technique](#référence-technique)
  - [Schéma de base de données](#schéma-de-base-de-données)
  - [Valeurs de référence](#valeurs-de-référence)
- [Extensibilité](#extensibilité)
  - [Ajouter une action personnalisée](#ajouter-une-action-personnalisée)
  - [Ajouter une condition personnalisée](#ajouter-une-condition-personnalisée)
- [Tests](#tests)
- [Conformité](#conformité)

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

**Prérequis :** GLPI 11.0.x, PHP ≥ 8.1, MariaDB ≥ 10.11.

1. Déposer le dossier `glpiworkflow/` dans `/var/www/glpi/plugins/`.
2. Vider le cache : `php /var/www/glpi/bin/console cache:clear`.
3. Redémarrer le conteneur ou le service PHP.
4. Aller dans **Administration > Plugins**, activer **Workflows ITSM**.
5. Vérifier que les deux actions automatiques sont programmées dans **Administration > Actions automatiques** :

| Action automatique | Fréquence recommandée | Rôle |
|--------------------|-----------------------|------|
| `executeScheduledWorkflows` | 1 minute | Démarre les nouvelles exécutions au déclenchement |
| `resumePendingWorkflows` | 5 minutes | Fait progresser les exécutions en attente |

> **Note :** après activation, les droits sont automatiquement créés sur tous les profils existants (valeur 31 = tous les droits). Ajustez-les dans **Administration > Profils > onglet Workflows ITSM**.

> **Note :** Node.js et npm ne sont pas requis en production. Le fichier Vue.js compilé est inclus dans l'archive.

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

![Designer — vue globale](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/03_designer_global.PNG)

Pour configurer une étape, cliquer sur l'icône engrenage de la ligne correspondante. Le panneau de droite affiche les paramètres de l'action choisie.

![Configuration d'une étape avec modèle de tâche](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/04_config_etape.PNG)

Les paramètres disponibles varient selon l'action :

- **Modèle de tâche** : sélectionner un gabarit existant dans GLPI (catégorie, durée, contenu pré-remplis).
- **Contenu** : texte libre qui surcharge le modèle si renseigné.
- **Groupe assigné** : groupe fixe ou « → Groupe attribué du ticket (dynamique) ».
- **Comportement si conditions non remplies** : `Bloquante` (le workflow attend) ou `Conditionnelle` (l'étape est sautée).

### Définir les conditions

Cliquer sur l'icône filtre (entonnoir) d'une étape pour ouvrir la modale de conditions. Chaque étape peut avoir une ou plusieurs conditions combinées par opérateur `ET` (toutes vraies) ou `OU` (au moins une vraie).

![Modale de conditions](https://raw.githubusercontent.com/Lexengan/workflowitsm/main/05_conditions.PNG)

### Enregistrer et activer

Cliquer **Enregistrer** en haut à droite pour persister l'ensemble du workflow (étapes, configurations, conditions, ordre). Le bouton statut dans la liste permet de basculer le workflow entre Actif et Inactif sans le supprimer.

---

## Fonctionnalités

- **Designer drag & drop** : création visuelle de workflows par glissement d'actions, réordonnancement par drag, suppression par étape
- **8 actions natives** couvrant les principaux cas ITSM
- **28 conditions natives** réparties sur 4 domaines : demandes, incidents, parc, base de connaissances
- **Opérateurs ET / OU** entre conditions par étape
- **Deux comportements par étape** : `Bloquante` (attend que les conditions soient vraies) ou `Conditionnelle` (saute si fausses)
- **Gabarits de tâches GLPI** : chaque étape « Ajouter une tâche » peut s'appuyer sur un gabarit existant
- **Groupe dynamique** : affectation de tâche au groupe attribué du ticket au moment de l'exécution
- **Affectation de matériel** : lier l'asset du ticket au demandeur ou à l'observateur (bénéficiaire)
- **Anti-doublon** : une seule exécution par couple workflow + ticket
- **Catalogue d'actions** : activation / désactivation individuelle de chaque action et condition
- **Journal d'exécution** : traçabilité complète par workflow, étape et ticket
- **Gestion des droits** : onglet dédié dans Administration > Profils

---

## Actions disponibles

| Action | Classe | Description |
|--------|--------|-------------|
| Ajouter une tâche | `AddTaskAction` | Crée une tâche sur le ticket (gabarit GLPI, groupe fixe ou groupe attribué dynamique) |
| Ajouter un suivi | `AddFollowupAction` | Ajoute un suivi interne ou public |
| Ajouter une solution et résoudre | `AddSolutionAction` | Ajoute une solution et passe le ticket en **Résolu** |
| Changer le statut du ticket | `ChangeStatusAction` | Modifie le statut du ticket vers une valeur configurée |
| Affecter à un groupe | `AssignGroupAction` | Affecte le ticket à un groupe spécifique |
| Affecter le matériel au demandeur | `AssignAssetToRequesterAction` | Met à jour `users_id` du matériel lié sur le **demandeur** (REQUESTER, type=1) |
| Affecter le matériel au bénéficiaire | `AssignAssetToObserverAction` | Met à jour `users_id` du matériel lié sur l'**observateur** (OBSERVER, type=3) — cas onboarding |
| Ouvrir un ticket demande | `OpenTicketAction` | Crée un nouveau ticket de type demande |

---

## Conditions disponibles

Le catalogue complet est accessible depuis **Plugins > Workflows ITSM > Catalogue d'actions**. Chaque condition peut être activée ou désactivée indépendamment.

### Demandes de service

| Condition | Classe | Paramètres clés |
|-----------|--------|-----------------|
| Catégorie ITIL particulière | `CategoryCreatedCondition` | `itilcategories_id` |
| Statut de tâche | `TaskStatusCondition` | `task_scope` (count_at_least / any / all / last), `min_count`, `status` |
| Ticket ouvert depuis (heures) | `TicketOpenSinceCondition` | `hours` |
| SLA bientôt dépassé | `SlaAboutToBreachCondition` | `minutes_before` |
| SLA dépassé | `SlaBreachedCondition` | — |

### Incidents

| Condition | Classe | Paramètres clés |
|-----------|--------|-----------------|
| Priorité au moins | `PriorityAtLeastCondition` | `priority` |
| Type de ticket | `TicketTypeCondition` | `ticket_type` (Incident / Demande) |
| Ticket réouvert | `TicketReopenedCondition` | — |

### Gestion du parc

| Condition | Classe | Paramètres clés |
|-----------|--------|-----------------|
| Matériel lié | `AssetLinkedCondition` | `itemtype` |
| Garantie expirée | `WarrantyExpiredCondition` | `itemtype` |
| Âge du matériel | `AssetAgeCondition` | `itemtype`, `max_age_years` |

### Base de connaissances

| Condition | Classe | Paramètres clés |
|-----------|--------|-----------------|
| Article KB lié | `KbArticleLinkedCondition` | — |

---

## Enchaînement séquentiel fiable

Pour orchestrer des tâches séquentielles (chaque tâche se déclenche seulement quand la précédente est terminée), utiliser `TaskStatusCondition` avec le scope `count_at_least` et des seuils croissants :

| Étape | Action | Condition |
|-------|--------|-----------|
| 1 | Crée tâche A | `CategoryCreatedCondition` |
| 2 | Crée tâche B | `count_at_least, min_count=1, status=2` |
| 3 | Crée tâche C | `count_at_least, min_count=2, status=2` |
| 4 | Résout le ticket | `count_at_least, min_count=3, status=2` |

Valeurs du paramètre `status` (vérifié `Planning.php:83-85`) :

| Valeur | Constante | Libellé |
|--------|-----------|---------|
| `0` | `Planning::INFO` | Information |
| `1` | `Planning::TODO` | À faire |
| `2` | `Planning::DONE` | Fait |

---

## Cas d'usage : workflow onboarding

Ce workflow illustre les possibilités du plugin sur un processus réel en cinq étapes.

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

Dans la même passe du cron, l'étape 5 s'enchaîne immédiatement : la solution est ajoutée et le ticket passe en **Résolu**.

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

**Logique d'enchaînement séquentiel :** les étapes bloquantes utilisent `TaskStatusCondition` avec le scope `count_at_least` et des seuils croissants. Chaque étape attend qu'un palier supérieur de tâches terminées soit atteint, garantissant l'ordre sans cibler une tâche précise.

**Anti-doublon :** le moteur vérifie qu'aucune exécution n'existe déjà pour un couple workflow + ticket avant d'en créer une nouvelle, évitant les déclenchements multiples lors de la création de ticket.

> **Important :** entre chaque étape, le délai maximum est la fréquence du cron `resumePendingWorkflows` (5 minutes par défaut). Pour des enchaînements plus réactifs, réduire cette fréquence dans **Administration > Actions automatiques**.

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

---

## Extensibilité

### Ajouter une action personnalisée

```php
namespace GlpiPlugin\Glpiworkflow\Action;

final class MyCustomAction extends AbstractAction
{
    public function execute(string $itemtype, int $items_id, array $config): array
    {
        // Votre logique ici
        return ['success' => true, 'data' => []];
    }

    public function getConfigFields(): array
    {
        return [
            'my_param' => [
                'label'    => __('Mon paramètre'),
                'type'     => 'text',
                'required' => true,
            ],
        ];
    }

    public function getLabel(): string { return __('Mon action'); }
    public function getIcon(): string  { return 'fa-star'; }
}
```

Déclarer ensuite la classe dans `WorkflowActionRegistry::allClasses()` et activer l'action depuis le catalogue.

### Ajouter une condition personnalisée

```php
namespace GlpiPlugin\Glpiworkflow\Condition;

final class MyCondition extends AbstractCondition
{
    public function evaluate(string $itemtype, int $items_id, array $parameters): bool
    {
        // Retourner true si la condition est remplie
        return true;
    }

    public function getParamFields(): array { return []; }
    public function getLabel(): string      { return __('Ma condition'); }
    public function getDomain(): string     { return __('Mon domaine'); }
}
```

Déclarer la classe dans `WorkflowConditionRegistry::allClasses()`.

---

## Tests

```bash
./vendor/bin/phpunit phpunit/functional/
./vendor/bin/phpunit phpunit/functional/ParameterValidationTest.php
```

---

## Conformité

- ✅ PSR-4 dans `src/` — `inc/` interdit
- ✅ Contrôleurs Symfony dans `src/Controller/` — zéro fichier `front/` ou `ajax/` créé
- ✅ Zéro requête SQL brute — `DBmysqlIterator` exclusivement
- ✅ Jeton CSRF sur tous les formulaires POST (`Session::getNewCSRFToken()`) et en-têtes AJAX (`X-Glpi-Csrf-Token` + `X-Requested-With`)
- ✅ PSR-12 : 0 erreur, 0 warning (PHP_CodeSniffer 4.0.2)
- ✅ PHPStan niveau 3 : 0 erreur de logique

---

*Plugin développé pour GLPI 11.0.x — Référence MO-GLPI-PLUGIN-002 v2, SPEC-GLPI-WF-001 v1.0*
