# AQI Data Pipeline & Warehouse

Pipeline de collecte automatique de la qualité de l'air (AQI) pour 5 villes,
avec stockage brut, fichier propre unifié, et data warehouse en schéma étoile.
Voir aussi [`ARCHITECTURE.md`](./ARCHITECTURE.md) pour la stack et ses justifications.

## Villes suivies

| Ville | Pays | Latitude | Longitude |
|---|---|---|---|
| Antananarivo | MG | -18.8792 | 47.5079 |
| Paris | FR | 48.8566 | 2.3522 |
| Nairobi | KE | -1.2921 | 36.8219 |
| Beijing | CN | 39.9042 | 116.4074 |
| New Delhi | IN | 28.6139 | 77.2090 |

(Définies dans `src/cities.py` — modifiable en un seul endroit.)

## Installation

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # puis remplir OWM_API_KEY et DATABASE_URL
export $(grep -v '^#' .env | xargs)   # ou utilisez direnv / python-dotenv
```

## Utilisation

```bash
# Backfill initial (à lancer une fois, rejouable sans risque)
python src/backfill.py --months 3        # ou --months 12 pour l'historique idéal

# Collecte horaire (ce que fait le cron GitHub Actions)
python src/fetch_current.py

# Reconstruction du fichier propre
python src/build_clean.py

# Chargement / mise à jour du warehouse
python src/load_warehouse.py
```

En production, ces 3 dernières étapes sont automatisées par
`.github/workflows/pipeline.yml`, déclenché chaque heure.

## Contrat de données — `data/clean/aqi_data.csv`

Une ligne = une ville + une heure. Trié par ville puis par horodatage. Dédoublonné sur (ville, timestamp).

| Colonne | Unité / format | Description |
|---|---|---|
| `city` | texte | Nom de la ville |
| `country` | code ISO2 | Pays |
| `latitude`, `longitude` | degrés décimaux | Coordonnées de la ville |
| `timestamp_utc` | ISO 8601 (UTC) | Horodatage de la mesure |
| `aqi` | entier 1–5 | Indice de qualité de l'air OpenWeatherMap (1=Bon … 5=Très mauvais) |
| `co` | µg/m³ | Monoxyde de carbone |
| `no` | µg/m³ | Monoxyde d'azote |
| `no2` | µg/m³ | Dioxyde d'azote |
| `o3` | µg/m³ | Ozone |
| `so2` | µg/m³ | Dioxyde de soufre |
| `pm2_5` | µg/m³ | Particules fines ≤ 2.5 µm |
| `pm10` | µg/m³ | Particules ≤ 10 µm |
| `nh3` | µg/m³ | Ammoniac |

## Schéma du warehouse (étoile)

- **`dim_city`** (city_id, name, country, latitude, longitude)
- **`dim_time`** (time_id, timestamp_utc, date, hour, day_of_week, is_weekend, month, year)
- **`fact_aqi`** (fact_id, city_id → dim_city, time_id → dim_time, aqi, co, no, no2, o3, so2, pm2_5, pm10, nh3)

DDL complet : [`src/schema.sql`](./src/schema.sql).

## Période couverte / trous connus

> À compléter par le groupe une fois le backfill lancé, ex :
> "Backfill du 20/04/2026 au 20/07/2026 (3 mois) pour les 5 villes.
> Trou identifié : Beijing entre le 02/06 et 03/06/2026 (indisponibilité de l'API, code 503)."

## Cohérence attendue

`nombre de lignes fact_aqi ≈ 5 villes × nombre d'heures couvertes`.
Tout écart doit être expliqué ici (pannes API, run GitHub Actions manqué, etc.).

## Connexion à la base (à compléter par le groupe)

- Fournisseur : **Neon** (Postgres serverless, offre gratuite)
- Hôte / port / nom de base : à renseigner ici pour IA1 (visible dans Neon > Project > Connection Details)
- `DATABASE_URL` : `postgresql://user:password@ep-xxxxx.region.aws.neon.tech/dbname?sslmode=require`
- Accès lecture : créer un rôle read-only dédié plutôt que de partager les identifiants admin
