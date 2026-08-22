# Zero export - Pilotage micro-onduleurs APSystems DS3-L

Automatisation Home Assistant pour piloter la puissance de sortie de trois micro-onduleurs afin de minimiser l'export réseau (zéro injection), avec priorité au chauffe-eau piloté par un pv-routeur, et une détection des gros consommateurs pour éviter les à-coups de régulation.

## Sommaire

- [Objectif](#objectif)
- [Entités requises](#entités-requises)
- [Principe de fonctionnement](#principe-de-fonctionnement)
- [Triggers](#triggers)
- [Logique de détection des gros consommateurs](#logique-de-détection-des-gros-consommateurs)
- [Mode "force max power"](#mode-force-max-power)
- [Calcul de la consigne par onduleur](#calcul-de-la-consigne-par-onduleur)
- [Cas particuliers](#cas-particuliers)
- [Variables de réglage](#variables-de-réglage)
- [Installation](#installation)
- [Automatisation complémentaire : réinitialisation de minuit](#automatisation-complémentaire--réinitialisation-de-minuit)
- [Pistes d'amélioration](#pistes-daméliration)

## Objectif

L'automatisation ajuste toutes les 30 secondes la puissance maximale de sortie (`maxpwr`) de trois micro-onduleurs pour que la puissance échangée avec le réseau (`p_grid`) reste proche de zéro, tout en :

- laissant la priorité au **pv-routeur** (chauffe-eau) tant que l'eau n'est pas chaude,
- évitant de sur-réagir lors des pics de consommation d'appareils ponctuels (four, plaques, lave-linge...) — c'est le rôle de la **détection de gros consommateur**,
- limitant le nombre de commandes envoyées aux onduleurs (matériel, latence, usure) via un pas d'ajustement adaptatif.

## Entités requises

L'automatisation suppose l'existence préalable de ces entités dans Home Assistant :

| Entité | Rôle |
|---|---|
| `number.inverter_703000387xx1_maxpwr` | Consigne de puissance max, onduleur 1 |
| `number.inverter_703000390xx2_maxpwr` | Consigne de puissance max, onduleur 2 |
| `number.inverter_703000387xx3_maxpwr` | Consigne de puissance max, onduleur 3 |
| `sensor.pvrouter_1370_power` | Puissance instantanée au compteur (import/export réseau) |
| `sensor.shelly_production_channel_1_power` | Production PV instantanée |
| `sensor.pvrouter_1370_routed` | Puissance actuellement routée vers le chauffe-eau par le pv-routeur |
| `sensor.entree_pvrouter_1370_power_lissee_60s` | Puissance nette lissée sur 60s, utilisée par `is_pic_actuel` en complément de l'instantané |
| `binary_sensor.dimmer_alarm_temp_d80a` | État "eau froide / eau chaude" du chauffe-eau |
| `input_number.seuil_gros_conso` | Seuil de déclenchement (W) de la détection gros consommateur — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration |
| `input_number.talon_conso_maison` | Consommation de base de la maison à soustraire (W) — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration |
| `input_number.pv_reference_baisse` | Mémorise le pic de production PV pour détecter une baisse — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `input_boolean.gros_consommateur_detecte` | Bascule indiquant qu'un gros consommateur est actif — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `input_boolean.force_max_power_actif` | Bascule indiquant que le mode "puissance max forcée" est actif — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `timer.gros_consommateur` | Minuteur de maintien de l'état "gros consommateur détecté" — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `counter.commandes_onduleurs_jour` | Compteur du nombre de commandes envoyées aux onduleurs — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |

> **Note** : `sensor.entree_pvrouter_1370_power_lissee_60s` reste nécessaire — il alimente `p_grid_lisse`, utilisé par `is_pic_actuel` en complément de la puissance instantanée (voir [Logique de détection des gros consommateurs](#logique-de-détection-des-gros-consommateurs)). S'il devient `unknown`/`unavailable`, `p_grid_lisse` retombe silencieusement sur la valeur instantanée (repli `float(p_grid_instant)`) plutôt que de faire échouer le calcul — voir [Cas particuliers](#cas-particuliers).

### Configuration des entités à créer

Les entités ci-dessous ne sont pas fournies par un intégration existante : ce sont des *helpers* à créer manuellement, soit via **Paramètres → Appareils et services → Aides**, soit en YAML (`configuration.yaml` ou fichier inclus, selon ton organisation).

Les capteurs physiques (`sensor.pvrouter_1370_power`, `sensor.shelly_production_channel_1_power`, `sensor.pvrouter_1370_routed`) et le `binary_sensor.dimmer_alarm_temp_d80a` dépendent de tes intégrations (compteur, Shelly, pv-router, sonde de température) et ne sont pas détaillés ici — adapte leurs `entity_id` dans le YAML de l'automatisation s'ils diffèrent chez toi.

En revanche, `sensor.entree_pvrouter_1370_power_lissee_60s` n'est pas fourni par une intégration : c'est un capteur `filter` **dérivé** de `sensor.pvrouter_1370_power`, à déclarer explicitement en YAML (plateforme `filter`, pas de configuration possible via l'UI) :

```
sensor:
  - platform: filter
    name: "entree_pvrouter_1370_power_lissee_60s"
    entity_id: sensor.pvrouter_1370_power
    filters:
      - filter: outlier
        window_size: 4
        radius: 50
      - filter: lowpass
        time_constant: 4
        precision: 1
```

- À placer dans `configuration.yaml`
- Après ajout, redémarre Home Assistant ou recharge les entités **Filtre** via **Outils de développement → YAML → Recharger la configuration**, sinon `sensor.entree_pvrouter_1370_power_lissee_60s` restera `unknown` et `p_grid_lisse` retombera sur l'instantané (voir la note plus haut sur ce repli).

**Via l'UI (Paramètres → Appareils et services → Aides → Créer une aide) :**

| Aide à créer | Type | Nom / `entity_id` | Configuration |
|---|---|---|---|
| Seuil gros conso | Nombre (`input_number`) | `input_number.seuil_gros_conso` | Min : `0` — Max : `3000` — Pas : `10` — Unité : `W` — Valeur initiale : `750` |
| Talon conso maison | Nombre (`input_number`) | `input_number.talon_conso_maison` | Min : `0` — Max : `1000` — Pas : `10` — Unité : `W` — Valeur initiale : `200` |
| Référence PV baisse | Nombre (`input_number`) | `input_number.pv_reference_baisse` | Min : `0` — Max : `10000` *(à adapter à la puissance crête de ton installation)* — Pas : `1` — Unité : `W` — Valeur initiale : `0` |
| Gros consommateur détecté | Interrupteur (`input_boolean`) | `input_boolean.gros_consommateur_detecte` | Aucune option particulière — état initial `off` |
| Force max power actif | Interrupteur (`input_boolean`) | `input_boolean.force_max_power_actif` | Aucune option particulière — état initial `off` |
| Minuteur gros consommateur | Minuteur (`timer`) | `timer.gros_consommateur` | Durée : `00:04:10` — l'automatisation ne passe pas de `duration` dans ses appels `timer.start`, c'est donc cette valeur configurée sur le helper qui s'applique à chaque (re)démarrage du minuteur |
| Compteur commandes onduleurs | Compteur (`counter`) | `counter.commandes_onduleurs_jour` | Pas : `1` — Valeur initiale : `0` — Minimum : `0` |

**Équivalent en YAML** pour les *helpers* manuels ci-dessus, à placer dans `configuration.yaml` (ou dans les fichiers `input_number.yaml` / `input_boolean.yaml` / `timer.yaml` / `counter.yaml` si tu utilises des `!include`) :

```yaml
input_number:
  seuil_gros_conso:
    name: Seuil gros consommateur
    min: 0
    max: 3000
    step: 10
    unit_of_measurement: "W"
    initial: 750
    icon: mdi:flash-alert

  talon_conso_maison:
    name: Talon de consommation maison
    min: 0
    max: 1000
    step: 10
    unit_of_measurement: "W"
    initial: 200
    icon: mdi:home-lightning-bolt

  pv_reference_baisse:
    name: Référence PV avant baisse
    min: 0
    max: 10000
    step: 1
    unit_of_measurement: "W"
    initial: 0
    icon: mdi:solar-power

input_boolean:
  gros_consommateur_detecte:
    name: Gros consommateur détecté
    icon: mdi:washing-machine

  force_max_power_actif:
    name: Force max power actif
    icon: mdi:flash

timer:
  gros_consommateur:
    name: Maintien gros consommateur
    duration: "00:04:10"
    restore: true

counter:
  commandes_onduleurs_jour:
    name: Commandes onduleurs du jour
    step: 1
    initial: 0
    minimum: 0
    icon: mdi:counter
```

> **Note sur `counter.commandes_onduleurs_jour`** : ce compteur n'est jamais remis à zéro automatiquement par l'automatisation principale elle-même. C'est le rôle de l'automatisation complémentaire décrite dans [Automatisation complémentaire : réinitialisation de minuit](#automatisation-complémentaire--réinitialisation-de-minuit).

> **Après création** : redémarre Home Assistant (ou recharge les *Aides*/`timer`/`counter` via **Outils de développement → YAML → Recharger la configuration**) avant d'activer l'automatisation, sinon les templates échoueront avec des entités `unknown`.

## Principe de fonctionnement

À chaque exécution (toutes les 30s, ou à l'expiration du minuteur), l'automatisation :

1. Détecte un éventuel pic de gros consommateur et démarre/relance le minuteur de maintien si besoin.
2. Met à jour la bascule `force_max_power_actif` selon le calcul `force_max_power`.
3. Met à jour la référence de production PV si celle-ci progresse.
4. Décide s'il faut ou non ajuster les onduleurs ce cycle-ci (pour économiser des commandes inutiles).
5. Si oui, calcule la nouvelle consigne par onduleur et l'envoie, avec un pas d'ajustement limité (`pas_max_adapt`) pour éviter les à-coups.

## Triggers

| id | Déclencheur | Rôle |
|---|---|---|
| `verification_periodique` | `time_pattern` toutes les 30s | Boucle de régulation normale, et point de vérification périodique de l'état "gros consommateur" |
| `timer_expire` | `timer.finished` sur `timer.gros_consommateur` | Fin du maintien de l'état gros consommateur |

> Il n'existe pas de trigger dédié à la détection d'un pic (pas de trigger `event`/`template` séparé avec une condition `for:`). Toute la détection de gros consommateur repose sur le template `is_pic_actuel` et sur le `choose` de l'action, réévalués à chaque exécution de `verification_periodique` (donc toutes les 30s) — voir [Logique de détection des gros consommateurs](#logique-de-détection-des-gros-consommateurs). Donner un id explicite à chaque trigger (comme c'est déjà fait ici) reste une bonne pratique : ça évite de dépendre d'un id positionnel implicite et facilite la lecture des exécutions dans l'onglet **Traces**.

## Logique de détection des gros consommateurs

La variable `is_pic_actuel` détermine si le point de fonctionnement actuel constitue un pic :

```jinja
is_pic_actuel: |-
  {{ (p_grid_corrige_lisse > seuil_gros_conso) and
      (p_grid_corrige_instant > seuil_gros_conso) }}
```

Elle exige que **à la fois** la puissance nette lissée sur 60s (`p_grid_corrige_lisse`, dérivée de `sensor.entree_pvrouter_1370_power_lissee_60s`) et la puissance nette instantanée (`p_grid_corrige_instant`) dépassent `seuil_gros_conso`. Ce double critère limite les faux positifs liés au bruit de mesure instantané tout en restant réactif (pas d'attente d'un lissage seul, qui serait plus lent à monter).

La branche de détection dans le `choose` est :

```
- conditions:
    - condition: template
      value_template: "{{ is_pic_actuel }}"
  sequence:
    - action: input_boolean.turn_on
      target:
        entity_id: input_boolean.gros_consommateur_detecte
    - condition: state
      entity_id: timer.gros_consommateur
      state: idle
    - action: timer.start
      target:
        entity_id: timer.gros_consommateur
```

- Cette branche ne s'exécute que si `is_pic_actuel` est vrai — l'état courant de la bascule n'intervient plus dans la condition d'entrée.
- `input_boolean.gros_consommateur_detecte` est systématiquement remis à `on` (idempotent si déjà `on`).
- `timer.gros_consommateur` n'est (re)démarré que s'il est actuellement `idle` : le `condition: state` intégré à la séquence interrompt le reste de la branche s'il échoue. **Tant que le timer est déjà en cours (`active`), il n'est pas relancé**, même si `is_pic_actuel` reste vrai aux cycles de 30s suivants.

Conséquence pratique, selon la durée du pic :

- **Pic court (< 4 min 10s)** : le timer démarre au premier cycle où `is_pic_actuel` devient vrai, puis expire normalement après (ou pendant) la fin du pic ; `timer_expire` remet alors `gros_consommateur_detecte` à `off`.
- **Pic long (> 4 min 10s), sans jamais redescendre sous le seuil** : le timer arrive au bout de sa durée **alors que le gros consommateur tourne encore**. Le run déclenché par `timer_expire` remet la bascule à `off` pour ce cycle précis (et, comme ce run a `trigger.id == 'timer_expire'`, `is_maintien_now` y est explicitement `false`, indépendamment de `is_pic_actuel`). Au cycle périodique suivant (jusqu'à 30s plus tard), si `is_pic_actuel` est toujours vrai, la bascule repasse à `on` et le timer — de nouveau `idle` — redémarre. Il existe donc une brève fenêtre (une seule exécution, jusqu'à 30s) où le mode "maintien gros consommateur" retombe avant de reprendre automatiquement si la charge est toujours active — voir [Pistes d'amélioration](#pistes-daméliration).

Tant que `gros_consommateur_detecte` est à `on` (et hors du cycle déclenché par `timer_expire` lui-même), `is_maintien_now` reste vraie, ce qui maintient le mode `force_max_power` actif :

```jinja
is_maintien_now: >-
  {{ (is_state('input_boolean.gros_consommateur_detecte', 'on') or
  is_pic_actuel)
      and trigger.id != 'timer_expire' }}
```

## Mode "force max power"

Quand `force_max_power` est vrai, tous les onduleurs sont poussés à leur puissance maximale (`pmax`), sans calcul fin de régulation. Ce mode s'active si :

- un gros consommateur est en cours de maintien (`is_maintien_now`), **ou**
- `p_grid_corrige_instant` dépasse `seuil_force_max_haut` (800 W), **ou**
- le mode était déjà actif et `p_grid_corrige_instant` reste au-dessus de `seuil_force_max_bas` (700 W) — c'est l'**hystérésis** : elle évite un battement rapide on/off si la puissance oscille autour d'un seul seuil.

## Calcul de la consigne par onduleur

Hors mode `force_max_power`, la consigne par onduleur (`per_inverter`) est calculée à partir de :

1. **`correction_globale`** : un delta de puissance totale à appliquer, dont le calcul dépend du contexte :
   - **Eau froide** : la correction vise à ramener la puissance routée vers le chauffe-eau (`pvrouter_power`) vers `target_routeur` (760 W), avec une bande morte de ±100 W (`bande_routeur`) et un gain asymétrique — **0.6 quand il faut augmenter** la puissance des onduleurs (le routeur reçoit moins que la cible), **1.2 quand il faut la diminuer** (trop de puissance routée par rapport à la cible).
   - **Eau chaude** : la correction vise à ramener `p_grid_instant` dans une bande de 5 à 85 W (`bande_grid_basse` / `bande_grid_haute`).
2. Cette correction globale s'ajoute à `consigne_totale` — la somme des consignes (`maxpwr`) actuellement appliquées aux trois onduleurs, multipliée par le nombre de sorties par MPPT (`sorties_par_mppt`). Le résultat est réparti également entre les onduleurs, puis borné entre `pmin` (20) et `pmax` (365).
3. L'ajustement effectif appliqué est limité par **`pas_max_adapt`**, qui varie selon le contexte (jusqu'à 365 en mode forcé, aussi bas que 20 en mode eau froide pour ne pas déstabiliser le routeur, 150 si export, 120 si forte consommation, 65 sinon).

Une commande n'est envoyée à un onduleur que si la nouvelle valeur arrondie diffère de la valeur actuelle — ce qui limite l'usure et incrémente `counter.commandes_onduleurs_jour`.


## Cas particuliers

- **`production_en_baisse`** : en fin de journée (soleil descendant), si la production chute de plus de 20 W (`marge_baisse`) sous son pic mémorisé, la stratégie de régulation change pour viser la bande grid classique plutôt que le routeur, afin d'anticiper la baisse de ressource.
- **`irradiance_insuffisante`** : si la somme des consignes actuellement appliquées aux onduleurs (`consigne_totale`) dépasse largement (+30 W) la production réelle, c'est le signe qu'ils sont bridés par manque d'irradiance plutôt que par la régulation — dans ce cas, on n'ajuste pas la consigne (elle ne servirait à rien), sauf test périodique.
- **`test_periodique_du`** : si aucun onduleur n'a été ajusté depuis au moins 2 minutes, on force un cycle de recalcul même en cas d'irradiance insuffisante, pour ne pas rester bloqué indéfiniment sur une ancienne consigne.
- **Garde de disponibilité des capteurs** : aucun ajustement n'est envoyé aux onduleurs tant que `sensor.pvrouter_1370_power`, `sensor.shelly_production_channel_1_power` ou `sensor.pvrouter_1370_routed` sont à `unknown`/`unavailable`. Ce dernier capteur a été ajouté à la garde suite à un risque identifié : sans cette vérification, une perte du capteur de puissance routée était silencieusement interprétée comme `0 W routé` (à cause du repli `| float(0)` sur `pvrouter_power`). En mode eau froide, cela pouvait pousser la régulation à augmenter la puissance des onduleurs en croyant que le routeur avait besoin de plus de surplus à dériver, alors qu'il ne fonctionnait peut-être plus pour l'absorber — un scénario propice à l'export réseau, contraire à l'objectif même de l'automatisation. Cette garde ne couvre en revanche pas `sensor.entree_pvrouter_1370_power_lissee_60s` : s'il devient indisponible, `p_grid_lisse` retombe silencieusement sur `p_grid_instant` (repli `float(p_grid_instant)`) plutôt que de bloquer explicitement l'ajustement, ce qui revient de fait à comparer deux fois l'instantané dans `is_pic_actuel`.
- **`pas_de_consigne_fin_journee`** : après 17h30 (ou soleil couché), si la production réelle est faible comparée à la consigne courante par onduleur, on n'envoie plus de nouvelle consigne (évite de solliciter les onduleurs pour rien en fin de journée).

## Variables de réglage

Réglables directement dans le YAML (`variables:`) :

| Variable | Valeur | Rôle |
|---|---|---|
| `pmin` / `pmax` | 20 / 365 | Bornes de puissance par onduleur |
| `seuil_global` | 5 | Seuil minimal de correction pour éviter le bruit de commande |
| `target_routeur` | 760 | Cible de puissance routée vers le chauffe-eau |
| `bande_routeur` | 100 | Bande morte autour de la cible routeur |
| `bande_grid_basse` / `bande_grid_haute` | 5 / 85 | Bande morte visée sur `p_grid` hors mode routeur |
| `seuil_force_max_haut` / `seuil_force_max_bas` | 800 / 700 | Seuils d'hystérésis du mode force max |
| `marge_baisse` | 20 | Seuil de détection de baisse de production |
| `marge_irradiance` | 30 | Marge de tolérance pour détecter le bridage par irradiance |
| `seuil_test_periodique` | 2 (minutes) | Délai avant de forcer un recalcul |

Réglables en direct via l'UI (sans toucher au YAML) :

| Entité | Rôle |
|---|---|
| `input_number.seuil_gros_conso` | Seuil de détection gros consommateur (W), défaut 750 |
| `input_number.talon_conso_maison` | Consommation de base à soustraire (W), défaut 200 |

## Installation

1. Créer au préalable toutes les [entités requises](#entités-requises) qui n'existeraient pas déjà (`input_number`, `input_boolean`, `timer`, `counter`).
2. Dans Home Assistant : **Paramètres → Automatisations et scènes → Créer une automatisation → ⋮ → Modifier en YAML**.
3. Coller le contenu de `Zero_export_pilotage_onduleurs.yaml`.
4. Enregistrer, puis vérifier dans l'onglet **Traces** que le trigger `verification_periodique` s'exécute bien toutes les 30s sans erreur.

## Automatisation complémentaire : réinitialisation de minuit

Une **automatisation séparée** s'exécute chaque jour à minuit et effectue trois actions :

```yaml
alias: Onduleurs - Réinitialisation quotidienne minuit
description: >-
  Remet le compteur de commandes à zéro et repasse les consignes des
  micro-onduleurs à 365 W à minuit.
triggers:
  - trigger: time
    at: "00:00:00"
actions:
  - action: counter.reset
    target:
      entity_id: counter.commandes_onduleurs_jour
  - action: number.set_value
    target:
      entity_id:
        - number.inverter_703000387xx1_maxpwr
        - number.inverter_703000387xx3_maxpwr
        - number.inverter_703000390xx2_maxpwr
    data:
      value: 365
  - action: input_number.set_value
    metadata: {}
    target:
      entity_id: input_number.pv_reference_baisse
    data:
      value: 0
mode: single
```

1. **Remet la consigne des trois onduleurs à `365`** (leur `pmax`). C'est l'état de repos par défaut : les onduleurs repartent "grand ouverts" avant la reprise de la production du lendemain.
2. **Réinitialise `input_number.pv_reference_baisse` à `0`**. Cette variable sert à `production_en_baisse` (mémorisation du pic de production PV de la journée pour détecter sa décroissance en fin d'après-midi) ; sans réinitialisation, la valeur de la veille resterait en mémoire et fausserait la détection le lendemain matin.
3. **Remet `counter.commandes_onduleurs_jour` à `0`**, pour obtenir un vrai compteur journalier plutôt qu'un cumul sans fin (voir la note dans la section [Configuration des entités à créer](#configuration-des-entités-à-créer)).

**Pourquoi remettre la consigne à 365 plutôt qu'à `pmin` ou à une valeur intermédiaire ?** Parce que la nuit, la production PV est nulle : quelle que soit la consigne des onduleurs, aucune puissance n'est réellement produite ni envoyée au réseau. La valeur à 365 ne sert donc qu'à préparer la remontée du lendemain.

**Comportement au lever du soleil** : au petit matin, la production PV réelle augmente progressivement en suivant la course du soleil, indépendamment de toute consigne envoyée par l'automatisation principale. Tant que `pas_de_consigne_fin_journee` reste vrai (car `production_reelle_totale / 6 < per_inverter`, la production étant encore trop faible comparée à la consigne au repos de 365) et que `force_max_power` n'est pas actif, l'automatisation principale **n'envoie aucune nouvelle commande** aux onduleurs — elle attend que la production réelle rattrape la consigne au repos avant de reprendre la régulation fine. Ça évite d'envoyer des commandes inutiles aux onduleurs pendant qu'ils ne produisent de toute façon rien, et laisse la production suivre naturellement la hausse d'irradiance jusqu'à ce que la régulation active reprenne la main.

## Pistes d'amélioration

- Pour un gros consommateur dont la charge dépasse la durée du timer (4 min 10s) sans jamais redescendre sous le seuil, une brève fenêtre de sortie du mode "maintien" peut survenir (voir [Logique de détection des gros consommateurs](#logique-de-détection-des-gros-consommateurs)). Selon la sensibilité de l'installation, deux pistes : allonger la durée du `timer.gros_consommateur` pour couvrir la durée typique des plus longs gros consommateurs connus, ou faire en sorte que le timer soit relancé (pas seulement démarré depuis `idle`) tant que `is_pic_actuel` reste vrai — au prix de devoir alors gérer explicitement l'expiration "naturelle" autrement (par exemple via un `for:` sur un trigger `state` dédié plutôt que sur `timer.finished`).
- Ajouter `sensor.entree_pvrouter_1370_power_lissee_60s` à la garde de disponibilité des capteurs, pour éviter le repli silencieux de `p_grid_lisse` sur `p_grid_instant` en cas d'indisponibilité du capteur lissé.
- Rendre la durée du `timer.gros_consommateur`, ainsi que `seuil_gros_conso` et `talon_conso_maison`, pilotables via `input_number`, pour ajuster sans repasser par le YAML/l'aide `timer`.