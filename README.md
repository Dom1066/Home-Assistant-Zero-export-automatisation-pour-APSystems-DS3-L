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
| `sensor.entree_pvrouter_1370_power_lissee_60s` | Version lissée sur 60s du capteur ci-dessus — **capteur à créer** (voir ci-dessous), ce n'est pas fourni par une intégration |
| `sensor.shelly_production_channel_1_power` | Production PV instantanée |
| `sensor.pvrouter_1370_routed` | Puissance actuellement routée vers le chauffe-eau par le pv-routeur |
| `binary_sensor.dimmer_alarm_temp_d80a` | État "eau froide / eau chaude" du chauffe-eau |
| `input_number.seuil_gros_conso` | Seuil de déclenchement (W) de la détection gros consommateur — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration |
| `input_number.talon_conso_maison` | Consommation de base de la maison à soustraire (W) — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration |
| `input_number.pv_reference_baisse` | Mémorise le pic de production PV pour détecter une baisse — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `input_boolean.gros_consommateur_detecte` | Bascule indiquant qu'un gros consommateur est actif — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `input_boolean.force_max_power_actif` | Bascule indiquant que le mode "puissance max forcée" est actif — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `timer.gros_consommateur` | Minuteur de maintien de l'état "gros consommateur détecté" — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |
| `counter.commandes_onduleurs_jour` | Compteur du nombre de commandes envoyées aux onduleurs — **à créer** (voir ci-dessous), ce n'est pas fourni par une intégration  |

### Configuration des entités à créer

Les entités ci-dessous ne sont pas fournies par un intégration existante : ce sont des *helpers* à créer manuellement, soit via **Paramètres → Appareils et services → Aides**, soit en YAML (`configuration.yaml` ou fichier inclus, selon ton organisation).

Les capteurs (`sensor.*`) et le `binary_sensor.*` dépendent de tes intégrations physiques (compteur, Shelly, pv-router, sonde de température) et ne sont pas détaillés ici — adapte leurs `entity_id` dans le YAML de l'automatisation s'ils diffèrent chez toi.

**Via l'UI (Paramètres → Appareils et services → Aides → Créer une aide) :**

| Aide à créer | Type | Nom / `entity_id` | Configuration |
|---|---|---|---|
| Seuil gros conso | Nombre (`input_number`) | `input_number.seuil_gros_conso` | Min : `0` — Max : `3000` — Pas : `10` — Unité : `W` — Valeur initiale : `750` |
| Talon conso maison | Nombre (`input_number`) | `input_number.talon_conso_maison` | Min : `0` — Max : `1000` — Pas : `10` — Unité : `W` — Valeur initiale : `200` |
| Référence PV baisse | Nombre (`input_number`) | `input_number.pv_reference_baisse` | Min : `0` — Max : `10000` *(à adapter à la puissance crête de ton installation)* — Pas : `1` — Unité : `W` — Valeur initiale : `0` |
| Gros consommateur détecté | Interrupteur (`input_boolean`) | `input_boolean.gros_consommateur_detecte` | Aucune option particulière — état initial `off` |
| Force max power actif | Interrupteur (`input_boolean`) | `input_boolean.force_max_power_actif` | Aucune option particulière — état initial `off` |
| Minuteur gros consommateur | Minuteur (`timer`) | `timer.gros_consommateur` | Durée par défaut : `00:04:10` (réécrite dynamiquement par l'automatisation à chaque déclenchement, cette valeur ne sert que de valeur de repli) |
| Compteur commandes onduleurs | Compteur (`counter`) | `counter.commandes_onduleurs_jour` | Pas : `1` — Valeur initiale : `0` — Minimum : `0` |

**Cas particulier : `sensor.entree_pvrouter_1370_power_lissee_60s`** — ce n'est pas un *helper* mais un **capteur dérivé**, à créer séparément puisqu'aucune intégration ne le fournit. La plateforme `filter` ne peut pas être configurée via l'UI (contrairement aux *Aides*) : elle doit être déclarée directement dans `configuration.yaml`, sous la clé `sensor:` :

```yaml
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

Ce capteur applique deux filtres en série :
- **`outlier`** : ignore une valeur si elle s'écarte de plus de `radius` (50 W) de la moyenne des `window_size` (4) dernières valeurs — élimine les mesures aberrantes ponctuelles.
- **`lowpass`** : lissage exponentiel avec une constante de temps de `4` (secondes), arrondi à `1` décimale — atténue les variations rapides tout en suivant les tendances réelles.

> **Important — l'`entity_id` généré doit correspondre à celui attendu par l'automatisation principale.** Avec ce `name: "entree_pvrouter_1370_power_lissee_60s"`, Home Assistant génère par défaut `sensor.entree_pvrouter_1370_power_lissee_60s`, ce qui correspond exactement à la référence utilisée dans le YAML de l'automatisation principale (variable `p_grid_lisse`) — aucun renommage manuel n'est donc nécessaire ici. Après ajout dans `configuration.yaml`, un redémarrage de Home Assistant est tout de même nécessaire (ce type de capteur ne se recharge pas via *Recharger la configuration*) — vérifie ensuite l'`entity_id` réel dans **Outils de développement → États** pour confirmer qu'il correspond bien.

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

À chaque exécution (toutes les 30s, ou sur détection d'un pic, ou à l'expiration du minuteur), l'automatisation :

1. Met à jour la bascule `force_max_power_actif` selon le calcul `force_max_power`.
2. Détecte un éventuel pic de gros consommateur et démarre le minuteur de maintien si besoin.
3. Met à jour la référence de production PV si celle-ci progresse.
4. Décide s'il faut ou non ajuster les onduleurs ce cycle-ci (pour économiser des commandes inutiles).
5. Si oui, calcule la nouvelle consigne par onduleur et l'envoie, avec un pas d'ajustement limité (`pas_max_adapt`) pour éviter les à-coups.

## Triggers

| id | Déclencheur | Rôle |
|---|---|---|
| *(sans id)* | `time_pattern` toutes les 30s | Boucle de régulation normale |
| `pic_instant` | Template : `p_grid - pvrouter - talon > seuil_gros_conso`, maintenu 30s | Détection réactive d'un pic de consommation |
| `timer_expire` | `timer.finished` sur `timer.gros_consommateur` | Fin du maintien de l'état gros consommateur |

## Logique de détection des gros consommateurs

Le trigger `pic_instant` se déclenche quand la puissance nette (compteur moins ce qui est routé vers le chauffe-eau, moins le talon de base) dépasse `seuil_gros_conso` de façon **continue pendant 30 secondes**.

La confirmation distingue ensuite deux cas — **l'entrée en détection** et le **maintien** — pour rester réactive au démarrage tout en ne s'interrompant pas pendant une charge longue :

```jinja
{{ (trigger.id == 'pic_instant')
   or (is_state('input_boolean.gros_consommateur_detecte', 'on') and is_pic_actuel) }}
```

- **Entrée en détection** : `trigger.id == 'pic_instant'`. Se déclenche dès que le trigger `pic_instant` se déclenche — c'est-à-dire après 30s de dépassement continu du seuil sur le signal brut, cette condition étant déjà intégrée au trigger lui-même (`for: "00:00:30"`). Il n'y a donc pas de re-vérification du seuil dans l'action, ni d'attente supplémentaire sur la valeur lissée.
- **Maintien** : `is_state('input_boolean.gros_consommateur_detecte', 'on') and is_pic_actuel`. Réévalué à **chaque cycle** (tous les 30s via `time_pattern`, ou plus tôt), tant que la détection est déjà active et qu'`is_pic_actuel` (qui combine instantané ET lissé, voir ci-dessous) reste vrai.

Dans les deux cas, l'action relance :

```yaml
- action: timer.start
  target:
    entity_id: timer.gros_consommateur
  data:
    duration: "00:04:10"
- action: input_boolean.turn_on
  target:
    entity_id: input_boolean.gros_consommateur_detecte
```

**Pourquoi ce découpage** : avant ce correctif, seule l'entrée en détection redémarrait le timer, et uniquement sur le flanc du trigger `pic_instant` (qui ne se redéclenche pas tant que sa condition reste vraie en continu). Résultat : si un gros consommateur tournait plus longtemps que la durée du timer (4 min 10s) sans redescendre sous le seuil entre-temps, le timer expirait **pendant que l'appareil tournait encore**, coupant la détection en plein milieu. Désormais, tant que `gros_consommateur_detecte` est `on` et que `is_pic_actuel` reste vrai, le timer est relancé à chaque cycle de 30s — il ne peut plus expirer prématurément pendant une charge continue.

`is_pic_actuel` sert de garde-fou anti-bruit pour le *maintien* (il combine l'instantané et le lissé sur 60s) :

```jinja
is_pic_actuel: |-
  {{ (p_grid_corrige_lisse > seuil_gros_conso) and
     (p_grid_corrige_instant > seuil_gros_conso) }}
```

Une fois confirmé (entrée ou maintien) :
- `timer.gros_consommateur` (re)démarre pour 4 min 10s,
- `input_boolean.gros_consommateur_detecte` passe (ou reste) à `on`.

Tant que ce booléen est `on` (ou qu'un pic est en cours), la variable `is_maintien_now` reste vraie, ce qui maintient le mode `force_max_power` actif jusqu'à l'expiration du minuteur — évitant de faire baisser prématurément les onduleurs pendant que le gros consommateur est encore en fonctionnement.

## Mode "force max power"

Quand `force_max_power` est vrai, tous les onduleurs sont poussés à leur puissance maximale (`pmax`), sans calcul fin de régulation. Ce mode s'active si :

- un gros consommateur est en cours de maintien (`is_maintien_now`), **ou**
- `p_grid_corrige_instant` dépasse `seuil_force_max_haut` (800 W), **ou**
- le mode était déjà actif et `p_grid_corrige_instant` reste au-dessus de `seuil_force_max_bas` (700 W) — c'est l'**hystérésis** : elle évite un battement rapide on/off si la puissance oscille autour d'un seul seuil.

## Calcul de la consigne par onduleur

Hors mode `force_max_power`, la consigne par onduleur (`per_inverter`) est calculée à partir de :

1. **`correction_globale`** : un delta de puissance totale à appliquer, dont le calcul dépend du contexte :
   - **Eau froide** : la correction vise à ramener la puissance routée vers le chauffe-eau (`pvrouter_power`) vers `target_routeur` (780 W), avec une bande morte de ±50 W (`bande_routeur`) et un gain asymétrique (0.6 à la baisse, 1.2 à la hausse).
   - **Eau chaude** : la correction vise à ramener `p_grid_instant` dans une bande de 5 à 85 W (`bande_grid_basse` / `bande_grid_haute`).
2. Cette correction globale s'ajoute à `consigne_totale` — la somme des consignes (`maxpwr`) actuellement appliquées aux trois onduleurs, multipliée par le nombre de sorties par MPPT (`sorties_par_mppt`). Le résultat est réparti également entre les onduleurs, puis borné entre `pmin` (20) et `pmax` (365).
3. L'ajustement effectif appliqué est limité par **`pas_max_adapt`**, qui varie selon le contexte (jusqu'à 365 en mode forcé, aussi bas que 20 en mode eau froide pour ne pas déstabiliser le routeur, 150 si export, 120 si forte consommation, 60 sinon).

Une commande n'est envoyée à un onduleur que si la nouvelle valeur arrondie diffère de la valeur actuelle — ce qui limite l'usure et incrémente `counter.commandes_onduleurs_jour`.


## Cas particuliers

- **`production_en_baisse`** : en fin de journée (soleil descendant), si la production chute de plus de 20 W (`marge_baisse`) sous son pic mémorisé, la stratégie de régulation change pour viser la bande grid classique plutôt que le routeur, afin d'anticiper la baisse de ressource.
- **`irradiance_insuffisante`** : si la somme des consignes actuellement appliquées aux onduleurs (`consigne_totale`) dépasse largement (+30 W) la production réelle, c'est le signe qu'ils sont bridés par manque d'irradiance plutôt que par la régulation — dans ce cas, on n'ajuste pas la consigne (elle ne servirait à rien), sauf test périodique.
- **`test_periodique_du`** : si aucun onduleur n'a été ajusté depuis au moins 2 minutes, on force un cycle de recalcul même en cas d'irradiance insuffisante, pour ne pas rester bloqué indéfiniment sur une ancienne consigne.
- **Garde de disponibilité des capteurs** : aucun ajustement n'est envoyé aux onduleurs tant que `sensor.pvrouter_1370_power`, `sensor.shelly_production_channel_1_power` ou `sensor.pvrouter_1370_routed` sont à `unknown`/`unavailable`. Ce dernier capteur a été ajouté à la garde suite à un risque identifié : sans cette vérification, une perte du capteur de puissance routée était silencieusement interprétée comme `0 W routé` (à cause du repli `| float(0)` sur `pvrouter_power`). En mode eau froide, cela pouvait pousser la régulation à augmenter la puissance des onduleurs en croyant que le routeur avait besoin de plus de surplus à dériver, alors qu'il ne fonctionnait peut-être plus pour l'absorber — un scénario propice à l'export réseau, contraire à l'objectif même de l'automatisation.
- **`pas_de_consigne_fin_journee`** : après 17h30 (ou soleil couché), si la production réelle est faible comparée à la consigne courante par onduleur, on n'envoie plus de nouvelle consigne (évite de solliciter les onduleurs pour rien en fin de journée).

## Variables de réglage

Réglables directement dans le YAML (`variables:`) :

| Variable | Valeur | Rôle |
|---|---|---|
| `pmin` / `pmax` | 20 / 365 | Bornes de puissance par onduleur |
| `seuil_global` | 5 | Seuil minimal de correction pour éviter le bruit de commande |
| `target_routeur` | 780 | Cible de puissance routée vers le chauffe-eau |
| `bande_routeur` | 50 | Bande morte autour de la cible routeur |
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
4. Enregistrer, puis vérifier dans l'onglet **Traces** que le trigger `time_pattern` s'exécute bien toutes les 30s sans erreur.

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

- Si la détection reste trop lente pour de vrais gros consommateurs (le lissage sur 60s met du temps à monter), envisager un capteur de lissage dédié plus court (20-30s) plutôt que de retoucher `..._lissee_60s`, qui sert peut-être à d'autres automatisations.
- Rendre `for: "00:00:30"` et la durée du `timer.gros_consommateur` pilotables via `input_number`, sur le même principe que `seuil_gros_conso`, pour ajuster sans repasser par le YAML.