# Plan your trip with Kayak

Projet Jedha — pipeline de recommandation de destinations françaises basé sur la météo et les hôtels, pour l'équipe marketing de Kayak.

## Contexte

L'équipe marketing veut une application capable de recommander les meilleures destinations françaises à un instant donné, en se basant sur des données réelles de météo et d'hôtellerie. Le projet se concentre sur les [35 villes les plus visitées de France](https://one-week-in.com/35-cities-to-visit-in-france/).

## Architecture du pipeline

```
villeKayak.csv (géocodage Nominatim, statique)
        │
        ▼
┌───────────────────┐
│   OpenWeather      │  prévisions J+1 à J+5, scoring "beau temps"
└─────────┬──────────┘
          ▼
      Top 5 villes
          │
          ▼
┌───────────────────┐
│  Scraping Booking   │  Selenium, 5 hôtels par ville du top 5
│  (Selenium)         │  dates alignées sur la fenêtre météo
└─────────┬──────────┘
          ▼
   CSV enrichi (météo + hôtels)
          │
          ▼
┌───────────────────┐
│   S3 (data lake)    │  destinations_enrichies.csv
└─────────┬──────────┘
          ▼
┌───────────────────┐
│  RDS PostgreSQL     │  ETL — schéma normalisé
│  (data warehouse)   │  villes (1 ligne/ville) + hotels (1 ligne/hôtel, FK city_id)
└─────────┬──────────┘
          ▼
┌───────────────────┐
│  Cartes Plotly       │  Top-5 destinations / Top-20 hôtels
└───────────────────┘
```

## Choix techniques

- **Météo dynamique, pas de fichier figé** : l'API OpenWeather gratuite (`/data/2.5/forecast`) ne couvre que les 5 prochains jours. Le score météo est donc recalculé à chaque exécution, et les dates de réservation Booking (`checkin`/`checkout`) sont alignées sur cette même fenêtre J+1 à J+5 — pas de décalage entre la période mesurée et la période réservée.
- **Score "beau temps"** : `100 - |temp_moy - 25| × 4 - pop_moy × 40`. Récompense une température proche de 25°C (pas "le plus chaud possible") et pénalise la probabilité de pluie.
- **Sélecteurs Booking** : ciblage par attributs `data-testid` (stables, posés par les développeurs pour leurs propres tests), pas par classes CSS générées au build (fragiles, changent à chaque déploiement).
- **Schéma RDS normalisé** : table `villes` (1 ligne/ville, score inclus) + table `hotels` (1 ligne/hôtel, `city_id` en clé étrangère) plutôt qu'une table unique — évite de dupliquer les infos ville sur chaque ligne d'hôtel, et correspond aux deux besoins de lecture distincts (carte villes / carte hôtels).
- **Rechargement propre à chaque run** : les tables RDS sont vidées (`TRUNCATE ... CASCADE`) avant chaque insertion, pour refléter toujours le run le plus récent plutôt que d'accumuler un historique.

## Prérequis

- Python 3.10+
- Google Chrome installé (pour Selenium/ChromeDriver)
- Un compte AWS avec accès S3 et RDS
- Une clé API [OpenWeather](https://openweathermap.org/api) (plan gratuit)

## Installation

```bash
pip install -r requirements.txt
```

## Configuration

Crée un fichier `.env` à la racine du projet (**ne jamais le committer** — voir `.gitignore`) :

```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
API_KEY=...              # clé OpenWeather

DB_HOST=...               # endpoint de l'instance RDS
DB_PORT=5432
DB_NAME=kayak
DB_USER=...
DB_PASSWORD=...
```

## Ressources AWS à créer au préalable

- **Bucket S3** — nom unique globalement, région `eu-west-3` recommandée
- **Instance RDS PostgreSQL** — via la console AWS (*Standard/Full configuration* → PostgreSQL → template *Free tier*), avec :
  - Accès public activé
  - Security group autorisant le port `5432` en entrée
  - Un nom de base initiale (`kayak`)

## Utilisation

Ouvrir `Kayak_clean.ipynb` et exécuter les cellules dans l'ordre (*Restart & Run All* recommandé pour repartir d'un état propre) :

1. **Setup** — charge le `.env`, ouvre les connexions S3 et RDS
2. **OpenWeather** — définit les fonctions de récupération météo et de scoring
3. **Booking** — définit la fonction de scraping Selenium
4. **Exécution** — calcule le top 5 météo du jour, scrape les hôtels correspondants
5. **S3** — fusionne météo + hôtels, envoie le CSV enrichi
6. **RDS** — crée le schéma, recharge les tables
7. **Visualisation** — génère les deux cartes Plotly

## Structure du projet

```
.
├── Kayak_clean.ipynb          # notebook principal
├── villeKayak.csv              # géocodage des 35 villes (Nominatim)
├── requirements.txt
├── README.md
├── .env                        # non versionné — voir .gitignore
└── .gitignore
```

## Livrables

- [x] CSV enrichi (météo + hôtels) dans un bucket S3
- [x] Base SQL RDS avec les mêmes données nettoyées
- [x] Carte Top-5 destinations (Plotly)
- [x] Carte Top-20 hôtels (Plotly)

## Sécurité

Aucun identifiant (clé AWS, mot de passe RDS, clé API) ne doit apparaître en dur dans le code ou dans les outputs sauvegardés d'un notebook. Toutes les valeurs sensibles passent par le `.env`, exclu du versionning.
