#  Analyse des dépenses marketing — SQL (MySQL)

Analyse de la performance de campagnes de marketing digital : coûts, revenus, entonnoir de conversion (impressions → clics → leads → commandes) et rentabilité par canal, par zone géographique et par période.


##  Objectifs

| # | Question métier | Indicateur |
|---|---|---|
| 1 | La campagne globale est-elle rentable ? | ROMI global |
| 2 | Quelles campagnes rapportent le plus ? | ROMI par campagne / catégorie |
| 3 | Quels jours ont été les plus coûteux, les plus rentables, les mieux convertis ? | Dépense, revenu, taux de conversion par date |
| 4 | Quel panier moyen par campagne ? | AOV (valeur moyenne de commande) |
| 5 | Les acheteurs sont-ils plus actifs en semaine ou le week-end ? | Commandes et revenu moyen par type de jour |
| 6 | Quel canal fonctionne le mieux ? | ROMI par catégorie (social, banner, influencer, search) |
| 7 | Faut-il cibler les villes tier 1 ou tier 2 ? | ROMI par zone géographique |

---

## 📁 Structure du projet

```
SQL-Project-Portfolio/
├── Marketing.csv      # Jeu de données brut
├── scripts/           # Requêtes SQL
└── README.md
```

---

##  Jeu de données

Source : [Kaggle — Analyze the marketing spending](https://www.kaggle.com/datasets/sinderpreet/analyze-the-marketing-spending)

| Colonne | Description |
|---|---|
| `c_date` | Date de la dépense |
| `campaign_name` | Nom de la campagne (canal + zone, ex. `facebook_tier1`) |
| `category` | Type de canal : `social`, `banner`, `influencer`, `search` |
| `campaign_id` | Identifiant unique de la campagne |
| `impressions` | Nombre d'affichages de l'annonce |
| `mark_spent` | Budget dépensé ce jour-là |
| `clicks` | Nombre de clics (= visites du site) |
| `leads` | Nombre d'inscriptions |
| `orders` | Nombre de commandes payées |
| `revenue` | Chiffre d'affaires généré |

---

##  Installation

**Prérequis** : MySQL 8+, MySQL Workbench (ou VS Code + extension SQLTools).


```sql
CREATE DATABASE IF NOT EXISTS marketing_spending;
USE marketing_spending;
```

Importer `Marketing.csv` dans une table `marketing_analysis` via **Table Data Import Wizard** (Workbench). Importer `c_date` en `VARCHAR`, elle sera convertie ensuite.

---

##  Nettoyage des données (Data Cleaning)

### 1. Séparer le canal et la zone géographique

`campaign_name` contient deux informations (`facebook_tier1` → `facebook` + `tier1`).

```sql
ALTER TABLE marketing_analysis ADD COLUMN geo_location VARCHAR(20);

UPDATE marketing_analysis
SET geo_location  = SUBSTRING_INDEX(campaign_name, '_', -1),
    campaign_name = SUBSTRING_INDEX(campaign_name, '_', 1);
```

### 2. Harmoniser la casse

Certaines valeurs sont mal écrites (ex. `facebOOK`).

```sql
UPDATE marketing_analysis
SET campaign_name = LOWER(TRIM(campaign_name)),
    geo_location  = LOWER(TRIM(geo_location));
```

### 3. Convertir les dates

Format source `mm/dd/yyyy` → type `DATE`.

```sql
UPDATE marketing_analysis
SET c_date = STR_TO_DATE(c_date, '%m/%d/%Y');

ALTER TABLE marketing_analysis MODIFY c_date DATE;
```

### 4. Ajouter le type de jour

```sql
ALTER TABLE marketing_analysis ADD COLUMN day_type VARCHAR(10);

UPDATE marketing_analysis
SET day_type = CASE
    WHEN DAYOFWEEK(c_date) IN (1, 7) THEN 'weekend'   -- 1 = dimanche, 7 = samedi
    ELSE 'weekday'
END;
```

### 5. Vérifications rapides

```sql
-- Valeurs nulles
SELECT
    SUM(c_date IS NULL)     AS null_dates,
    SUM(revenue IS NULL)    AS null_revenue,
    SUM(mark_spent IS NULL) AS null_spent
FROM marketing_analysis;

-- Valeurs distinctes après nettoyage
SELECT DISTINCT campaign_name, geo_location, category FROM marketing_analysis;
```

---

## Définitions des indicateurs

| Indicateur | Formule |
|---|---|
| **Profit** | `revenue − mark_spent` |
| **ROMI** | `(revenue − mark_spent) / mark_spent × 100` |
| **CTR** | `clicks / impressions × 100` |
| **Taux de conversion** | `orders / clicks × 100` |
| **AOV** (panier moyen) | `revenue / orders` |
| **CPO** (coût par commande) | `mark_spent / orders` |

> ⚠️ Toujours agréger avant de diviser (`SUM(a) / SUM(b)`) et non faire la moyenne de ratios ligne par ligne. `NULLIF(x, 0)` évite les divisions par zéro.

---

##  Analyses

### 1. ROMI global

```sql
SELECT
    SUM(mark_spent)                                         AS total_spent,
    SUM(revenue)                                            AS total_revenue,
    SUM(revenue) - SUM(mark_spent)                          AS profit,
    ROUND((SUM(revenue) - SUM(mark_spent)) / SUM(mark_spent) * 100, 2) AS romi_pct
FROM marketing_analysis;
```

### 2. ROMI par campagne

```sql
SELECT
    campaign_name,
    SUM(mark_spent) AS spent,
    SUM(revenue)    AS revenue,
    ROUND((SUM(revenue) - SUM(mark_spent)) / NULLIF(SUM(mark_spent), 0) * 100, 2) AS romi_pct
FROM marketing_analysis
GROUP BY campaign_name
ORDER BY romi_pct DESC;
```

### 3. Performance par date

```sql
-- Vue journalière complète
SELECT
    c_date,
    SUM(mark_spent) AS spent,
    SUM(revenue)    AS revenue,
    ROUND(SUM(orders) / NULLIF(SUM(clicks), 0) * 100, 2) AS conversion_pct
FROM marketing_analysis
GROUP BY c_date
ORDER BY c_date;
```

```sql
-- Jour avec la plus grosse dépense
SELECT c_date, SUM(mark_spent) AS spent
FROM marketing_analysis GROUP BY c_date
ORDER BY spent DESC LIMIT 1;

-- Jour avec le plus gros revenu
SELECT c_date, SUM(revenue) AS revenue
FROM marketing_analysis GROUP BY c_date
ORDER BY revenue DESC LIMIT 1;

-- Jours avec la conversion la plus haute / la plus basse
SELECT c_date, ROUND(SUM(orders) / NULLIF(SUM(clicks), 0) * 100, 2) AS conversion_pct
FROM marketing_analysis GROUP BY c_date
ORDER BY conversion_pct DESC LIMIT 1;

SELECT c_date, ROUND(SUM(orders) / NULLIF(SUM(clicks), 0) * 100, 2) AS conversion_pct
FROM marketing_analysis GROUP BY c_date
ORDER BY conversion_pct ASC LIMIT 1;
```

### 4. Panier moyen (AOV) par campagne

```sql
SELECT
    campaign_name,
    SUM(orders) AS orders,
    ROUND(SUM(revenue) / NULLIF(SUM(orders), 0), 2) AS aov
FROM marketing_analysis
GROUP BY campaign_name
ORDER BY aov DESC;
```

### 5. Semaine vs week-end

On compare des moyennes **par jour**, car il y a 5 jours de semaine pour 2 jours de week-end.

```sql
WITH daily AS (
    SELECT c_date, day_type,
           SUM(orders)  AS orders,
           SUM(revenue) AS revenue
    FROM marketing_analysis
    GROUP BY c_date, day_type
)
SELECT
    day_type,
    COUNT(*)                AS nb_days,
    ROUND(AVG(orders), 1)   AS avg_orders_per_day,
    ROUND(AVG(revenue), 2)  AS avg_revenue_per_day
FROM daily
GROUP BY day_type;
```

### 6. Meilleur type de canal

```sql
SELECT
    category,
    SUM(mark_spent) AS spent,
    SUM(revenue)    AS revenue,
    ROUND((SUM(revenue) - SUM(mark_spent)) / NULLIF(SUM(mark_spent), 0) * 100, 2) AS romi_pct,
    ROUND(SUM(clicks) / NULLIF(SUM(impressions), 0) * 100, 2) AS ctr_pct,
    ROUND(SUM(orders) / NULLIF(SUM(clicks), 0) * 100, 2)      AS conversion_pct,
    ROUND(SUM(mark_spent) / NULLIF(SUM(orders), 0), 2)        AS cpo
FROM marketing_analysis
GROUP BY category
ORDER BY romi_pct DESC;
```

### 7. Tier 1 vs tier 2

```sql
SELECT
    geo_location,
    SUM(mark_spent) AS spent,
    SUM(revenue)    AS revenue,
    ROUND((SUM(revenue) - SUM(mark_spent)) / NULLIF(SUM(mark_spent), 0) * 100, 2) AS romi_pct,
    ROUND(SUM(orders) / NULLIF(SUM(clicks), 0) * 100, 2) AS conversion_pct
FROM marketing_analysis
GROUP BY geo_location
ORDER BY romi_pct DESC;
```

-


## 🛠️ Stack

- **MySQL 8** — stockage et requêtes
- **MySQL Workbench / VS Code (SQLTools)** — exécution

## Sources

- Données : [Kaggle — sinderpreet](https://www.kaggle.com/datasets/sinderpreet/analyze-the-marketing-spending)
