# Audit d'intégrité inter-SGBD — TP 4

**Big Data Engineering — Master 1** · Fédération multi-SGBD avec PySpark
Auteur : Ndeye Khadidiatou Lo

## Contexte

Les quatre tables de la plateforme e-commerce sont réparties sur quatre SGBD
(MySQL, PostgreSQL, Oracle XE, SQL Server). **Aucune clé étrangère ne traverse
ces systèmes** : l'intégrité référentielle a donc été vérifiée côté Spark, par
anti-jointures (`LEFT ANTI JOIN`) sur les vues temporaires (partie 7 du notebook).

## Anomalies détectées

| # | Identifiant | Table / SGBD source | Référence cassée | Montant |
|---|---|---|---|---|
| 1 | `CMD21` | `commandes` — Oracle XE (schéma `SPARK_USER`) | client `C999` absent de `clients` (MySQL) | 15 000 FCFA |
| 2 | `CMD22` | `commandes` — Oracle XE (schéma `SPARK_USER`) | produit `P999` absent de `produits` (PostgreSQL) | 50 000 FCFA |
| 3 | `PAY27` | `paiements` — SQL Server (`dbo.paiements`) | commande `CMD99` absente de `commandes` (Oracle) | 99 000 FCFA |

### 1. `CMD21` — commande sans client (client fantôme `C999`)

- **Requête** : `commandes LEFT ANTI JOIN clients ON client_id`
- **Impact métier** : une vente livrée et payée (15 000 FCFA, `PAY18`) ne peut être
  rattachée à aucun client : elle disparaît du CA par ville et fausse le suivi
  client (fidélité, SAV, facturation).

### 2. `CMD22` — commande sans produit (produit fantôme `P999`)

- **Requête** : `commandes LEFT ANTI JOIN produits ON produit_id`
- **Impact métier** : 50 000 FCFA encaissés (`PAY19`) pour un article absent du
  catalogue : le CA par catégorie est sous-estimé et le stock ne peut pas être
  mis à jour.

### 3. `PAY27` — paiement orphelin (commande inconnue `CMD99`)

- **Requête** : `paiements LEFT ANTI JOIN commandes ON commande_id`
- **Impact métier** : 99 000 FCFA encaissés via Wave sans commande correspondante
  côté Ventes : risque de fraude, de double encaissement ou de remboursement dû
  au client — c'est l'anomalie la plus grave.
- **Responsable proposé** : la **Finance** (propriétaire de `paiements`) pilote le
  rapprochement avec l'équipe **Ventes**.

### Contrôle complémentaire — livrées sans paiement réussi (ex. 8)

**0 ligne** : toutes les commandes livrées ont un paiement réussi. Ce résultat vide
prouve la cohérence livraison / encaissement.

## Effet sur la vue fédérée

| Étape | Lignes | Cause |
|---|---:|---|
| Commandes (Oracle) | 30 | point de départ |
| + clients (MySQL) | 29 | `CMD21` exclue (client `C999` inconnu) |
| + produits (PostgreSQL) | 28 | `CMD22` exclue (produit `P999` inconnu) |
| + paiements (SQL Server) + filtres | 22 | 3 annulées + 3 en cours exclues |

Les jointures internes **masquent silencieusement** 65 000 FCFA de CA réellement
encaissé (`CMD21` + `CMD22`) et ignorent les 99 000 FCFA de `PAY27`.

## Tableau de relevés

| Mesure | Valeur |
|---|---:|
| nb_clients (MySQL) | 12 |
| nb_produits (PostgreSQL) | 10 |
| nb_commandes (Oracle) | 30 |
| nb_paiements (SQL Server) | 27 |
| jointure commandes-clients | 29 |
| jointure 3 tables | 28 |
| vue federee (4 SGBD) | 22 |
| commandes sans client | 1 |
| commandes sans produit | 1 |
| paiements orphelins | 1 |
| livrees sans paiement reussi | 0 |
| CA federe total (FCFA) | 1 452 000 |

### Analyses consolidées (ex. 9)

| Regroupement | Tête de classement | CA (FCFA) |
|---|---|---:|
| Par ville | Dakar | 723 000 |
| Par catégorie | Informatique | 740 000 |
| Par moyen de paiement | Wave | 537 000 |

## Recommandations

1. Mettre en place un **contrôle d'intégrité automatisé** (anti-jointures) à
   chaque chargement, avec alerte vers les data owners.
2. Utiliser des **jointures externes** (`LEFT JOIN`) dans les rapports de CA pour
   ne pas perdre de lignes en silence.
3. Partager un **référentiel commun** (clients, produits) entre les équipes.

## Notes d'environnement (Windows)

- Ajout d'une cellule installant `winutils.exe` / `hadoop.dll` et définissant
  `HADOOP_HOME` : sans elle, la SparkSession échoue (`HADOOP_HOME and
  hadoop.home.dir are unset`).
- Java 17 (64 bits) imposé via `JAVA_HOME` : le Java 8 32 bits du poste était
  incompatible avec `ojdbc11` et `mssql-jdbc`.
- SQL Server 2025 Express utilisé (au lieu de 2022) : même port 1433, aucun impact.
- Types Oracle lus : `decimal(5,0)` et `decimal(12,0)` (et non `decimal(38,10)`),
  car les colonnes sont déclarées `NUMBER(5)` et `NUMBER(12)`.
