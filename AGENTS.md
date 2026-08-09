# Repository Guidelines

# Waveshare Bathroom — ESPHome / Home Assistant

## Langue de travail

**Toujours communiquer en français.**

Toutes les réponses, explications, commentaires de code, résumés de modifications et messages destinés à l'utilisateur doivent être rédigés en français, sauf demande explicite contraire.

Les noms techniques, noms de composants, propriétés YAML, IDs ESPHome, noms d'entités Home Assistant et APIs doivent évidemment conserver leur syntaxe d'origine.

---

# Objectif du projet

Ce projet pilote un :

```text
Waveshare ESP32-S3 Touch LCD 4.3" — 800×480
```

installé dans la salle de bain.

L'écran doit principalement afficher des données provenant de Home Assistant :

* météo actuelle ;
* prévisions météo horaires ;
* prévisions météo journalières ;
* température intérieure ;
* température extérieure ;
* humidité intérieure ;
* humidité extérieure ;
* Humidex extérieur ;
* d'autres données Home Assistant pourront être ajoutées plus tard.

Le device utilise :

* ESPHome ;
* ESP-IDF ;
* LVGL ;
* l'API native ESPHome connectée à Home Assistant ;
* ESPHome LVGL Designer pour construire l'interface graphique.

Le dossier du projet est :

```text
/Users/ichoui/esphome
```

Le device principal est :

```text
waveshare-bathroom
```

---

# Architecture générale

Le flux météo est :

```text
Open-Meteo
    ↓
Firebase Function
    ↓
Firestore
    ↓
Firebase HTTP endpoint
    ↓
Home Assistant
    ↓
ESPHome Native API
    ↓
Waveshare ESP32-S3
    ↓
Interface LVGL
```

---

# Firebase

Une Firebase Function récupère périodiquement les données météo depuis Open-Meteo.

Ces données sont ensuite stockées dans Firestore.

Home Assistant récupère ce dataset via un endpoint HTTP exposé par une autre Firebase Function.

Firebase est donc la source amont des données météo.

Cependant :

**le Waveshare ne doit pas interroger Firebase ou Open-Meteo directement.**

Le chemin normal doit rester :

```text
Firebase
→ Home Assistant
→ ESPHome
```

Ne pas ajouter d'appel HTTP direct vers Firebase ou Open-Meteo depuis l'ESP32 sauf demande explicite.

---

# Home Assistant

Home Assistant tourne sur :

```text
Raspberry Pi 4B
Home Assistant OS
```

Le dataset météo Firebase/Open-Meteo complet est exposé dans Home Assistant via :

```text
sensor.saint_georges_openmeteo_fb
```

Sa structure principale ressemble à :

```text
current

forecast
  hourly[]
  daily[]

supplemental
  current
  hourly[]
  daily[]
  alerts[]
```

Une entité météo Home Assistant est également générée :

```text
weather.saint_georges
```

Elle sert notamment à alimenter la carte native Home Assistant de type :

```text
weather-forecast
```

Pour le Waveshare, privilégier cependant le dataset JSON complet décrit ci-dessous plutôt que d'essayer de reproduire ou de consommer directement la carte Home Assistant.

---

# Bridge météo Home Assistant → ESPHome

Un Template Sensor Home Assistant spécifique a été créé pour ESPHome :

```text
sensor.saint_georges_esphome_weather
```

Son attribut principal est :

```text
data
```

Cet attribut contient l'intégralité du dataset météo sérialisé en JSON via Home Assistant `to_json`.

Structure générale :

```json
{
  "current": {},
  "forecast": {
    "hourly": [],
    "daily": []
  },
  "supplemental": {}
}
```

Cette architecture est volontaire.

Le but est de transporter **un seul gros objet JSON structuré**, au lieu de créer des dizaines de sensors Home Assistant pour chaque heure et chaque jour de forecast.

Ne pas éclater inutilement ce JSON en multiples entités HA.

---

# Import météo dans ESPHome

Le bridge Home Assistant est importé dans ESPHome via :

```yaml
text_sensor:
  - platform: homeassistant
    id: weather_data
    entity_id: sensor.saint_georges_esphome_weather
    attribute: data
```

Le composant JSON ESPHome est activé à la racine :

```yaml
json:
```

Le parsing peut ensuite être effectué avec :

```cpp
json::parse_json(...)
```

---

# Principales clés météo disponibles

Les clés importantes sont :

```text
current

forecast.hourly[]
forecast.daily[]

supplemental.current
supplemental.hourly[]
supplemental.daily[]
supplemental.alerts[]
```

## Données météo actuelles utiles

Exemples de clés disponibles dans :

```text
current
```

```text
current.condition
current.cloud_coverage
current.native_temperature
current.native_apparent_temperature
current.native_dew_point
current.humidity
current.native_precipitation
current.native_pressure
current.native_visibility
current.native_wind_speed
current.native_wind_gust_speed
current.uv_index
current.wind_bearing
```

Le libellé météo français est disponible dans :

```text
supplemental.current.condition_label_fr
```

---

# Forecast horaire

Les prévisions horaires sont disponibles dans :

```text
forecast.hourly[]
```

Chaque entrée peut notamment contenir :

```text
datetime
is_daytime
condition
cloud_coverage
humidity
native_temperature
native_apparent_temperature
native_dew_point
native_precipitation
native_pressure
native_wind_speed
native_wind_gust_speed
precipitation_probability
uv_index
wind_bearing
```

---

# Forecast journalier

Les prévisions journalières sont disponibles dans :

```text
forecast.daily[]
```

Chaque journée peut notamment contenir :

```text
datetime
condition
cloud_coverage
humidity
native_temperature
native_templow
native_apparent_temperature
native_dew_point
native_precipitation
native_pressure
native_wind_speed
native_wind_gust_speed
precipitation_probability
uv_index
wind_bearing
```

---

# Données supplémentaires journalières

Les données complémentaires sont disponibles dans :

```text
supplemental.daily[]
```

Elles peuvent notamment contenir :

```text
date
weather_code
condition_label_fr
sunrise_at
sunset_at
visibility_mean_m
surface_pressure_mean_hpa
daylight_duration_sec
sunshine_duration_sec
```

Les alertes météo sont disponibles dans :

```text
supplemental.alerts[]
```

---

# Sensors Home Assistant utilisés

Ces IDs sont actuellement les sources de vérité du projet.

## Température intérieure

ESPHome :

```text
indoor_temperature
```

Home Assistant :

```text
sensor.thermo_int_temperature
```

## Température extérieure

ESPHome :

```text
outdoor_temperature
```

Home Assistant :

```text
sensor.thermo_ext_temperature
```

## Humidité intérieure

ESPHome :

```text
indoor_humidity
```

Home Assistant :

```text
sensor.thermo_int_humidite
```

## Humidité extérieure

ESPHome :

```text
outdoor_humidity
```

Home Assistant :

```text
sensor.thermo_ext_humidite
```

## Humidex extérieur

ESPHome :

```text
humidex_out
```

Home Assistant :

```text
sensor.thermo_ext_humidex
```

Ne pas renommer ces IDs ESPHome ni remplacer ces entités Home Assistant sans demande explicite.

---

# Interface LVGL

L'interface utilise l'intégration LVGL native d'ESPHome.

Le projet cherche volontairement à éviter autant que possible :

* le positionnement pixel par pixel ;
* la création manuelle de gros layouts LVGL ;
* le travail de design directement dans le YAML ;
* l'utilisation excessive de `x`, `y`, Flexbox ou Grid écrits à la main.

L'objectif est de laisser un maximum de travail graphique à :

```text
ESPHome LVGL Designer
```

---

# ESPHome LVGL Designer

ESPHome LVGL Designer est utilisé pour construire visuellement :

* les cartes ;
* les gauges ;
* les boutons ;
* les composants météo ;
* les layouts ;
* les couleurs ;
* les tailles ;
* les icônes ;
* la composition générale de l'écran.

Workflow attendu :

```text
ESPHome LVGL Designer
        ↓
YAML LVGL généré
        ↓
Projet ESPHome
        ↓
Binding des données HA / ESPHome
```

Le Designer gère principalement :

```text
présentation
layout
styles
dimensions
cartes
gauges
icônes
composition visuelle
```

ESPHome gère principalement :

```text
données Home Assistant
bindings
JSON
logique
mise à jour dynamique des widgets
actions
```

---

# Blocs générés par LVGL Designer

Les blocs de ce type :

```text
# >>> LVGL Designer - generated from the canvas. >>>
...
# <<< end LVGL Designer block <<<
```

sont générés automatiquement par le Designer.

Le Designer indique que les modifications manuelles faites à l'intérieur de ces blocs peuvent être écrasées lors d'un nouvel export.

État actuel du projet :

**le design final recherché doit être compris à l'intérieur du bloc commenté LVGL Designer.**

Pour l'instant, ne pas chercher à décider si le reste du design LVGL hors de ce bloc doit être conservé, supprimé ou réorganisé, sauf demande explicite.

Il faut donc :

1. modifier autant que possible le design dans LVGL Designer ;
2. donner des IDs propres aux widgets directement dans le Designer ;
3. exporter le YAML ;
4. effectuer les bindings ESPHome autour de ces IDs.

Éviter de refaire manuellement le design généré sauf demande explicite.

---

# Convention de nommage des widgets

Toujours préférer des IDs sémantiques.

Exemple :

```text
outdoor_humidity_card
outdoor_humidity_arc
outdoor_humidity_value
outdoor_humidity_label

indoor_humidity_card
indoor_humidity_arc
indoor_humidity_value
indoor_humidity_label

outdoor_temperature_card
outdoor_temperature_icon
outdoor_temperature_value

indoor_temperature_card
indoor_temperature_icon
indoor_temperature_value
```

Convention recommandée :

```text
<contexte>_<métrique>_<type-widget>
```

Exemple :

```text
outdoor_humidity_arc
```

Éviter autant que possible les IDs générés aléatoirement par le Designer.

---

# Binding Sensor → Widget

Les données Home Assistant sont importées via les sensors ESPHome.

Les widgets LVGL doivent ensuite être mis à jour à partir des triggers ESPHome.

Exemple pour l'humidité extérieure :

```yaml
- platform: homeassistant
  id: outdoor_humidity
  entity_id: sensor.thermo_ext_humidite
  on_value:
    then:
      - lvgl.arc.update:
          id: outdoor_humidity_arc
          value: !lambda return static_cast<int>(x);

      - lvgl.label.update:
          id: outdoor_humidity_value
          text:
            format: "%.0f%%"
            args: [x]
```

Il faut toujours bien distinguer :

```text
outdoor_humidity
```

qui est un **sensor ESPHome**,

de :

```text
outdoor_humidity_arc
outdoor_humidity_value
```

qui sont des **widgets LVGL**.

---

# Stratégie pour le widget météo

Ne pas créer :

```text
weather_hour_1_temperature
weather_hour_2_temperature
weather_hour_3_temperature
...
```

ni une multitude de sensors équivalents.

Le JSON météo doit rester structuré.

Le widget météo doit directement exploiter :

```text
weather_data
```

puis accéder aux éléments nécessaires :

```text
current

forecast.hourly[n]

forecast.daily[n]

supplemental.current

supplemental.daily[n]
```

---

# Données typiquement nécessaires au futur widget météo

## Conditions actuelles

```text
température
température ressentie
condition
humidité
vent
rafales
précipitations
pression
UV
visibilité
couverture nuageuse
```

## Prochaines heures

```text
datetime
temperature
condition
precipitation_probability
native_precipitation
humidity
wind
uv_index
```

## Prochains jours

```text
date
condition
temperature max
temperature min
precipitation_probability
native_precipitation
sunrise
sunset
```

Éviter de parser plusieurs fois le gros JSON indépendamment dans chaque widget si une couche de parsing commune permet de faire plus propre.

---

# Règles de développement pour Codex

Avant toute modification :

1. lire le YAML existant ;
2. comprendre la structure actuelle avant de modifier ;
3. préserver la configuration hardware fonctionnelle ;
4. préserver les entity IDs Home Assistant existants ;
5. préserver les IDs ESPHome existants sauf demande explicite ;
6. identifier si le bloc cible vient de LVGL Designer ;
7. ne pas supprimer un sensor ou une fonctionnalité existante simplement parce qu'elle n'est pas concernée par la modification demandée.

Ne jamais retirer silencieusement :

```text
indoor_temperature
outdoor_temperature
indoor_humidity
outdoor_humidity
humidex_out
weather_data
```

---

# Validation ESPHome

Ne lancer aucune commande ESPHome de sa propre initiative.

Cela inclut :

```text
esphome config
esphome compile
esphome upload
esphome logs
```

Ne pas vérifier, compiler, installer, flasher ou lancer d'OTA sans demande explicite.

Après une modification du YAML, signaler à l'utilisateur qu'il devra lancer lui-même la validation ESPHome s'il veut confirmer que le fichier est accepté par ESPHome.

---

# Principes généraux

Privilégier :

```text
simplicité
lisibilité
IDs sémantiques
réutilisabilité
bindings directs
capacités natives ESPHome
LVGL Designer pour le visuel
Home Assistant comme source de données
```

Éviter :

```text
layouts manuels complexes
pixel perfect écrit à la main
duplication de sensors
duplication des forecasts
appels directs Firebase depuis l'ESP32
abstractions inutiles
IDs générés illisibles
suppression de code existant non concerné
```

Le but n'est pas de construire une application embarquée complexe à la main.

Le but est d'utiliser :

```text
Home Assistant
+
ESPHome
+
LVGL
+
ESPHome LVGL Designer
```

pour obtenir un écran domotique propre, maintenable et simple à faire évoluer, avec le moins possible de travail graphique bas niveau.

---

## Project Structure & Module Organization

This repository contains a single ESPHome device configuration:

- `waveshare-bathroom.yaml` is the tracked source of truth for the Waveshare bathroom ESP32-S3 display.
- `secrets.yaml` is local-only and ignored; keep Wi-Fi, OTA, and other credentials there.
- `.esphome/` and `.device-builder*` files are generated by ESPHome Device Builder and should not be committed.

The YAML is organized by broad sections: substitutions, hardware, logging/connection, Home Assistant data, and the LVGL interface. The LVGL Designer block is marked as generated; avoid hand-editing inside that block unless you are intentionally accepting that future canvas exports may overwrite the change.

## Build, Test, and Development Commands

Ces commandes peuvent être utiles depuis la racine du dépôt, mais Codex ne doit pas les lancer automatiquement :

- `esphome config waveshare-bathroom.yaml` valide le YAML, les substitutions, les secrets et la configuration des composants.
- `esphome compile waveshare-bathroom.yaml` construit le firmware pour la cible ESP32-S3.
- `esphome upload waveshare-bathroom.yaml` flashe le device, généralement en OTA après la première installation.
- `esphome logs waveshare-bathroom.yaml` affiche les logs runtime pour déboguer les sensors Home Assistant, le tactile et les mises à jour LVGL.

If using ESPHome Device Builder, keep generated local state untracked and commit only intentional YAML changes.

## Coding Style & Naming Conventions

Use two-space YAML indentation and preserve ESPHome's nested structure. Prefer lowercase snake_case IDs, such as `indoor_temperature_value`, and keep related entities grouped together. Use descriptive `id` values because they are referenced by LVGL update actions.

Keep comments brief and useful. Existing comments include French labels for UI/domain concepts; match the surrounding language when editing nearby sections.

## Testing Guidelines

Demander à l'utilisateur de lancer `esphome config waveshare-bathroom.yaml` lorsqu'une validation ESPHome est nécessaire. Ne pas lancer `esphome compile`, `esphome upload`, `esphome logs` ni d'action OTA sans demande explicite. Après flash, l'utilisateur doit vérifier les mises à jour des entités Home Assistant clés et les logs d'erreur JSON/LVGL.

## Commit & Pull Request Guidelines

The current history uses short imperative messages like `Edit waveshare-bathroom.yaml`. Keep commits concise and focused, for example `Update bathroom weather layout` or `Fix outdoor humidity label`.

Pull requests should describe the user-visible device change, list validation performed, and call out any required Home Assistant entity or `secrets.yaml` updates. Include screenshots or photos when changing the display layout.

## Security & Configuration Tips

Do not commit `secrets.yaml`, generated Device Builder files, OTA passwords, Wi-Fi credentials, or API keys. Use `!secret` references for sensitive values and verify `.gitignore` still covers local runtime artifacts.
