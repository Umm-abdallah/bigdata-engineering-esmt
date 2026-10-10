# Big Data Engineering — ESMT

Dépôt des travaux pratiques du cours Big Data Engineering (Spark, Kafka, Delta Lake, Airflow).
Auteur : Ndeye Khadidiatou Lo

## Structure

| Dossier | Contenu |
|---|---|
| `notebooks/` | Notebooks Jupyter des TP, exécutés avec leurs sorties |
| `docs/` | Rapports et audits associés aux TP |

## TP 4 — Fédération de données multi-SGBD avec PySpark

Jointure de 4 tables hébergées dans 4 SGBD différents (MySQL, PostgreSQL,
Oracle XE, SQL Server) via une seule SparkSession et 4 pilotes JDBC.

- Notebook : [`notebooks/TP4_federation_multi_sgbd.ipynb`](notebooks/TP4_federation_multi_sgbd.ipynb)
- Audit d'intégrité : [`docs/AUDIT_TP4.md`](docs/AUDIT_TP4.md)

**Résultats clés** : vue fédérée de 22 lignes, CA consolidé de 1 452 000 FCFA,
3 anomalies d'intégrité détectées (CMD21, CMD22, PAY27), résultat écrit dans
l'entrepôt PostgreSQL `analytics.ventes_federees`.

**Environnement** : Windows, Python 3.13, PySpark 3.5.1, Java 17, MySQL 8,
PostgreSQL 16, Oracle XE 21c, SQL Server 2025 Express.# bigdata-engineering-esmt
