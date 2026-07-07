# Issue Triage Platform

> Plateforme complète de **triage automatique des incidents** du projet Apache Spark, construite sur une **architecture en médaillon** (Bronze → Argent → Or) hébergée sur **Snowflake** et orchestrée par **dbt**, avec deux modèles de Machine Learning et deux canaux de restitution (Streamlit & Power BI).

**Projet de Fin d'Études (PFE)** — Filière Big Data, ENSA El Jadida
**Étudiante :** DRIHAM Siham · **Encadrant entreprise :** SQLI Rabat

![Snowflake](https://img.shields.io/badge/Snowflake-29B5E8?logo=snowflake&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-FF694B?logo=dbt&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.11+-3776AB?logo=python&logoColor=white)
![DeBERTa](https://img.shields.io/badge/NLP-DeBERTa--v3-8A2BE2)
![Streamlit](https://img.shields.io/badge/Streamlit-FF4B4B?logo=streamlit&logoColor=white)
![PowerBI](https://img.shields.io/badge/Power_BI-F2C811?logo=powerbi&logoColor=black)
![Tests](https://img.shields.io/badge/dbt_tests-66_PASS-brightgreen)

---

## 📖 Table des matières
- [Description](#-description)
- [Architecture](#️-architecture)
- [Démonstration](#-démonstration)
- [Résultats obtenus](#-résultats-obtenus)
- [Stack technique](#-stack-technique)
- [Jeu de données](#-jeu-de-données)
- [Prérequis](#-prérequis)
- [Mise en route](#-mise-en-route)
- [Structure du projet](#-structure-du-projet)
- [Documentation](#-documentation)
- [Tableau de bord Power BI](#-tableau-de-bord-power-bi)
- [Auteur](#-auteur)

---

## Description

Plateforme de triage automatique des incidents du projet **Apache Spark**, sur une architecture en médaillon hébergée sur Snowflake et orchestrée par dbt. Le dataset source est le dump public JIRA Apache Spark de Kaggle (mars 2025, **~49 832 tickets SPARK**).

> **Contexte / confidentialité :** la solution a été conçue pour un client de SQLI pendant le stage. Les données client étant confidentielles, **tous les résultats de ce dépôt sont une reproduction sur données publiques Apache Spark**, exécutée dans un **compte Snowflake d'essai personnel**.

Deux chemins de consommation sont exposés :

- **Chemin 1 — Inférence IA** : application **Streamlit-in-Snowflake** qui, pour un ticket donné, prédit le **type d'incident** (`microsoft/deberta-v3-base` fine-tuné, 4 classes : Bug, Improvement, Sub-task, Other — texte enrichi du signal `has_parent`) et la **résolution probable** (régression logistique hybride : embeddings `all-mpnet-base-v2` + features tabulaires). Elle produit une explication corrective **« Que faire ? »** générée par l'**API Google Gemini** (`gemini-2.0-flash`), avec repli sur un moteur de règles contextuel si l'API est indisponible.
- **Chemin 2 — Tableau de bord analytique Power BI** : dashboard connecté nativement aux tables `MARTS_ANALYTICS.*` de Snowflake (volumes mensuels, dynamique de résolution, charge des assignataires, liens entre tickets, versions).

---

## Architecture

L'architecture suit le pattern **médaillon** (Bronze → Argent → Or) : un **Data Lakehouse** sur Snowflake, orchestré par dbt.

<img width="1242" height="700" alt="architecture" src="https://github.com/user-attachments/assets/5f62b109-2cbd-4c73-9283-e0fea37b5959" />


| Couche | Schéma Snowflake | Rôle |
|---|---|---|
| 🥉 **Bronze** | `RAW` | Données brutes chargées telles quelles (VARCHAR), immuables. Filet de sécurité + historique. |
| 🥈 **Argent** | `STAGING` (4 vues) + `INTERMEDIATE` (tables) | Staging : filtre SPARK, renommage, cast, déduplication. Intermediate : nettoyage NLP + features (changelog, liens). |
| 🥇 **Or** | `MARTS_ML` + `MARTS_ANALYTICS` + `PREDICTIONS` | `MART_ML` (ML) · 7 tables analytiques (Power BI) · prédictions. |

**Pipeline dbt :** 20 modèles, 2 seeds — **66 tests de données : PASS=66, WARN=0, ERROR=0** (exécutés nativement sur Snowflake via l'intégration Git des Workspaces).

---

## Architecture des modèles de Machine Learning

La plateforme repose sur **deux modèles spécialisés** qui partent d'une source unique (`MART_ML`) et convergent dans l'application Streamlit. Chaque modèle reçoit une **représentation textuelle distincte** pour éviter toute fuite de données.

### Vue d'ensemble

<img width="1002" height="337" alt="general" src="https://github.com/user-attachments/assets/009141cf-b26f-43b5-9a26-2bc1aae1707f" />

| | Modèle 1 — Type d'incident | Modèle 2 — Résolution |
|---|---|---|
| **Cible** | 4 classes (Bug, Improvement, Sub-task, Other) | 7 classes (Fixed, Won't Fix…) |
| **Modèle** | DeBERTa-v3-base fine-tuné | Régression logistique hybride |
| **Entrée** | Texte enrichi de marqueurs | Embeddings 768d + features tabulaires |

### Modèle 1 — Classification du type (DeBERTa)

Le texte du ticket est **enrichi de marqueurs** encodant des signaux structurels (`[HAS-PARENT]`, signaux lexicaux, flags structurels), puis tokenisé par SentencePiece et passé à un DeBERTa-v3-base fine-tuné avec des class weights.

<img width="1727" height="505" alt="pipeline_deberta" src="https://github.com/user-attachments/assets/448f7589-7498-4727-9f6f-d984b402fa7a" />


### Modèle 2 — Prédiction de la résolution (hybride)

Le modèle combine deux branches : une **branche textuelle** (embeddings 768d via `all-mpnet-base-v2`) et une **branche tabulaire** (features numériques normalisées par StandardScaler). Les deux sont concaténées (785d) puis passées à une régression logistique.

<img width="1273" height="605" alt="resol" src="https://github.com/user-attachments/assets/db28b537-4f95-45ee-957c-fc63709b4158" />

---

## Démonstration

### Application Streamlit — triage en temps réel

https://github.com/user-attachments/assets/b5ec09de-8760-4190-828e-69463c01c0d8

### Tableau de bord Power BI

https://github.com/user-attachments/assets/71c3131a-ffe4-4e34-8fd5-8c7c87647cf1

---

## Résultats obtenus
*(jeu de validation, 3 809 tickets)*

| Métrique | Modèle | Valeur | Seuil cible |
|----------|--------|--------|-------------|
| Accuracy issuetype (4 classes) | DeBERTa-v3-base fine-tuné | **79,6 %** | > 70 % |
| Macro-F1 issuetype (4 classes) | DeBERTa-v3-base fine-tuné | **73,63 %** | — |
| Accuracy résolution (7 classes) | LogisticRegression (all-mpnet-base-v2 + features) | 81,3 % | > 75 % |
| Macro-F1 résolution (7 classes) | LogisticRegression (all-mpnet-base-v2 + features) | 26,9 % | — |

> **Lecture honnête de la résolution :** le seuil d'accuracy de 75 % est franchi (81,3 %), mais sur une cible aussi déséquilibrée (*Fixed* ≈ 90 % de la validation) l'accuracy est trompeuse. La métrique honnête est le **macro-F1 (26,9 %)** ; ce volet est présenté comme un **résultat partiel assumé**, pas comme une réussite.

**Résultats par classe issuetype :**

| Classe | Precision | Recall | F1 | Support |
|--------|-----------|--------|----|---------|
| Bug | 0,72 | 0,69 | 0,71 | 671 |
| Improvement | 0,66 | 0,72 | 0,69 | 1 012 |
| Other | 0,57 | 0,49 | 0,53 | 572 |
| Sub-task | 0,99 | 1,00 | **1,00** | 1 554 |

> Le Sub-task atteint F1 = 1,00 grâce au signal **`has_parent`** (100 % des Sub-tasks ont un ticket parent JIRA, 0 % pour Bug/Improvement). Signal récupéré via l'API JIRA (`spark_parent_keys.csv`) et préfixé dans le texte sous la forme `[HAS-PARENT]`. C'est un signal quasi-étiquette : l'essentiel de la compréhension textuelle se mesure sur Bug/Improvement/Other (F1 0,53–0,71).

> **Note sur les features de résolution :** le modèle déployé a été entraîné sur 19 features tabulaires ; le rapport en documente **17**, après exclusion de deux variables constituant une fuite de la cible (`resolution_days`, `n_resolution_changes`). Cf. `docs/decisions_log.md`.

---

## Stack technique

| Domaine | Technologies |
|---|---|
| **Data Lakehouse** | Snowflake |
| **Transformation & qualité** | dbt (medallion, 66 tests, docs) |
| **Langage & ML** | Python 3.11+, scikit-learn, PyTorch, Hugging Face Transformers |
| **NLP** | `microsoft/deberta-v3-base` (fine-tuné), `all-mpnet-base-v2` (embeddings) |
| **Explication générative** | API Google Gemini (`gemini-2.0-flash`) + repli moteur de règles |
| **Application** | Streamlit-in-Snowflake |
| **Visualisation** | Power BI (connecteur natif Snowflake) |

---

## Jeu de données

| Table | Lignes |
|-------|--------|
| RAW.ISSUES | 1 149 321 |
| RAW.COMMENTS | 5 047 714 |
| RAW.CHANGELOG | 9 653 526 |
| RAW.ISSUELINKS | 390 063 |
| Tickets SPARK filtrés | 49 832 |
| MARTS_ML.MART_ML (train + val) | 42 083 |
| Entraînement / validation | 38 274 / 3 809 |

---

## Prérequis

- Python 3.11+ (chargement + pipeline ML) ; Python 3.12 pour le venv dbt
- Un compte Snowflake (trial ou payant — voir note Cortex)
- dbt-snowflake 1.11+ (installé via `uv`)
- Les 4 fichiers CSV source dans `data/`
- *(optionnel)* une clé `GEMINI_API_KEY` pour l'explication « Que faire ? »

> **Note Snowflake Cortex :** les fonctions `SNOWFLAKE.CORTEX.COMPLETE` et `EMBED_TEXT_1024` sont bloquées sur les comptes trial. Le pipeline utilise donc `sentence-transformers` (gratuit, local) et le DeBERTa v3 fine-tuné ; l'explication « Que faire ? » passe par l'API Gemini.

---

## Mise en route

L'installation pas-à-pas complète est détaillée dans **[SETUP.md](SETUP.md)**. En résumé :

```bash
git clone https://github.com/[user]/[repo].git
cd [repo]
cp .env.example .env          # renseigner SNOWFLAKE_* (+ GEMINI_API_KEY optionnel)

python load/write_profiles.py # profil dbt
python load/run_phase1.py     # DB PFE_SPARK + 7 schémas + warehouse PFE_WH
python load/03_put_files.py   # PUT des CSV vers @RAW.CSV_STAGE
python load/run_phase4.py     # COPY INTO tables brutes

cd dbt_project
dbt deps && dbt seed && dbt run && dbt test   # 20 modèles, 66 tests

python fetch_parent_keys.py        # has_parent via l'API JIRA -> spark_parent_keys.csv
python migrate_to_snowflake.py     # upload + reconstruction de mart_ml
python load/run_ml_pipeline.py     # embeddings + prédictions + évaluation
```

### Déploiement (Streamlit-in-Snowflake)

```bash
python load/train_save_resolution_model.py   # génère results/sklearn_models/
python deploy_streamlit_snowflake.py          # déploie l'app + les modèles
python deploy_notebooks.py                    # déploie les 2 notebooks
```

Accès dans Snowsight : **Streamlit → SPARK_TRIAGE_APP** et **Projects → Notebooks**.

---

## Structure du projet

```
DataLakeHouse_PFE/
├── fetch_parent_keys.py          # has_parent via l'API JIRA Apache
├── migrate_to_snowflake.py       # upload parent keys + reconstruction mart_ml
├── deploy_streamlit_snowflake.py # déploie l'app Streamlit-in-Snowflake + modèles
├── deploy_notebooks.py           # déploie les 2 notebooks dans Snowflake
│
├── data/                         # CSVs source (non versionnés, ~8 GB)
│
├── load/                         # Chargement Bronze + pipeline ML
│   ├── run_phase1.py             # crée la base + schémas + warehouse
│   ├── 03_put_files.py           # PUT vers le stage
│   ├── run_phase4.py             # COPY INTO + vérification
│   ├── run_ml_pipeline.py        # embeddings + prédictions + évaluation
│   └── train_save_resolution_model.py
│
├── dbt_project/                  # Transformations Argent + Or
│   ├── models/
│   │   ├── staging/              # 4 vues (1:1 avec les sources)
│   │   ├── intermediate/         # tables NLP, features, split, analytics
│   │   └── marts/
│   │       ├── ml/               # MART_ML (42 083 lignes)
│   │       └── analytics/        # 7 tables MART_ANALYTICS_*
│   ├── seeds/                    # issuetype_mapping.csv, resolution_mapping.csv
│   └── macros/                   # clean_jira_text, generate_schema_name
│
├── apps/
│   └── inference/                # Application Streamlit-in-Snowflake
│
├── results/                      # Artefacts (embeddings, modèles, notebooks)
│   ├── deberta_v3_parent.zip     # modèle DeBERTa fine-tuné (599 MB)
│   ├── sklearn_models/           # clf_resolution.pkl, scaler.pkl, meta.json
│   └── notebooks/                # deberta_fine_tuning_v3.ipynb, sklearn_resolution_training.ipynb
│
└── docs/
    ├── architecture.md
    ├── data_dictionary.md
    └── decisions_log.md
```

---

## Documentation

| Document | Contenu |
|----------|---------|
| [SETUP.md](SETUP.md) | Guide d'installation pas-à-pas (Snowflake → dbt → ML → déploiement) |
| [docs/architecture.md](docs/architecture.md) | Schéma médaillon et description des couches |
| [docs/data_dictionary.md](docs/data_dictionary.md) | Description de chaque table/colonne |
| [docs/decisions_log.md](docs/decisions_log.md) | Choix architecturaux figés et leur justification |

---

## Tableau de bord Power BI

Le dashboard analytique (tables `MARTS_ANALYTICS.*`) est développé sous Power BI Desktop et connecté nativement à Snowflake.

<!-- Ajoute ici quelques captures d'écran de tes pages de dashboard -->
<img width="882" alt="Dashboard Power BI" src="docs/images/dashboard_powerbi.png" />

> Fichier `.pbix` disponible sur SharePoint (requiert Power BI Desktop et un accès autorisé).

---

## Auteur

**DRIHAM Siham**
Ingénieure d'État en Big Data — ENSA El Jadida
🔗 [LinkedIn]([https://linkedin.com/in/[ton-profil]](https://www.linkedin.com/in/siham-driham-955838238/)) · 📧 [sihamdriham@gmail.com]

Projet de Fin d'Études réalisé au sein du département **Data & IA de SQLI Rabat**.

---

