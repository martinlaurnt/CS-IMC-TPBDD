# Travaux Pratiques Bases de Données Relationnelles et Graphes
Dans ce TP, nous allons travailler sur des jeux de données publics issus des [datasets IMDB](https://datasets.imdbws.com/)). Ces données sont disponibles dès le démarrage du TP dans une base de données relationelle [Azure SQL Database](https://docs.microsoft.com/fr-fr/azure/azure-sql/database/sql-database-paas-overview).

Les objectifs du TP sont les suivants:

1. Se familiariser avec les différences de paradigme entre le requêtage relationnel et graphe
2. Transformer les données pour les exporter depuis la base de données relationelle SQL, vers une base de données graphe Neo4j
3. Implémenter un ensemble de requêtes:
    - en SQL sur la base de donnée relationelle [Azure SQL Database](https://docs.microsoft.com/fr-fr/azure/azure-sql/database/sql-database-paas-overview)
    - en [Cypher](https://neo4j.com/developer/cypher/) sur la base de données graphe [Neo4j](https://neo4j.com/)
    - en [Apache Gremlin](https://tinkerpop.apache.org/docs/3.3.2/reference/#graph-traversal-steps) sur une base de données graphe répartie [Cosmos Db](https://docs.microsoft.com/en-us/azure/cosmos-db/graph/graph-introduction)
4. (optionnel) Encapsuler ces requêtes dans des API serverless (Azure Functions) et mesurer les performances

## Prérequis
### GitHub Codespaces et récupération du code
L'installation des prérequis à ce TP étant fastidieuse, nous vous avons préconfiguré un [GitHub Codespace](https://github.com/features/codespaces). Un compte GitHub sera nécessaire, sachant que les étudiants bénéficient de certains avantages via le programme [GitHub Student Developer Pack](https://education.github.com/pack).

1. Effectuez un [fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo) privé de ce repository. Vous travaillerez désormais uniquement à partir de ce fork.
   ![image](https://github.com/user-attachments/assets/cb2263fe-19f9-4005-98fe-fcad811a0d5e)

3. Assurez-vous d'être dans votre fork (notamment dans l'URL du navigateur)

4. Créez votre codespace en cliquant sur *Create a codespace on main*. Choisissez la plus petite machine possible: le TP ne demande pas beaucoup de ressource car les tâches de calcul sont surtout effectuées par les bases de données.
![image](https://github.com/user-attachments/assets/337199dd-4223-4326-9e7d-4180c0792082)

6. Créez un fichier `.env` à la racine de votre repository avec le contenu suivant (certaines valeurs seront fournies par l'enseignant):
```bash
export TPBDD_SERVER=tpbdd-sqlserver.database.windows.net
export TPBDD_DB=tpbdd-sql
export TPBDD_USERNAME=sqluser-read
export TPBDD_PASSWORD=###A_REMPLIR###
export ODBC_DRIVER={ODBC Driver 18 for SQL Server}

export TPBDD_NEO4J_SERVER=bolt://...
export TPBDD_NEO4J_USER=###A_REMPLIR###
export TPBDD_NEO4J_PASSWORD=###A_REMPLIR###

# Just the account name, no https:// or URL suffix
export TPBDD_COSMOSDB_ACCOUNT=tpbdd-movies-cdb
export TPBDD_COSMOSDB_GRAPH=tpbdd-movies-graph
export TPBDD_COSMOSDB_PASSWORD=##A_REMPLIR##

```
5. Démarrez un *nouveau* terminal et exécutez la commande suivante:
```bash
python3 pyodbc-py2neo-test.py
```
6. Si elle fonctionne, alors votre environnement est valide!

⚠️ Le nombre d'heure gratuites octroyées par GitHub est limité. Lorsque votre session est terminée, veillez absolument à:
    1. Commiter et pusher (sync) votre code dans votre repository
    2. Eteindre votre Codespace depuis le [tableau de bord Codespaces](https://github.com/codespaces)

### Outillage
1. Il est conseiller de passer l'interface du portail Azure en **anglais** afin de suivre plus facilement les instructions du TP.
2. Si vous le pouvez, installez [Azure Data Studio](https://docs.microsoft.com/fr-fr/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15) pour faciliter la création de vos requêtes SQL. Si ce n'est pas possible, vous pourrez également effectuer les requêtes depuis votre navigateur via [portail Azure](https://portal.azure.com), en ouvrant la page correspondant à la base de données SQL `tpbdd-sqlserver/tpbdd-sql`

# Partie 1 - Base de données relationnelle - Azure SQL Database
Ecrire les requêtes ci-dessous, et les expliquer en deux ou trois phrases maximum en français ou en anglais.

## Requêtes SQL

**Exercice 0**: Décrivez les tables et les attributs.

**Exercice 1** (¼ pt): Visualisez l'année de naissance de l'artiste `Brad Pitt`.

**Exercice 2** (¼ pt): Comptez le nombre d'artistes présents dans la base de donnee. 

**Exercice 3** (¼ pt): Trouvez les noms des artistes nés en `1960`, affichez ensuite leur nombre.

**Exercice 4** (1 pt): Trouvez l'année de naissance la plus représentée parmi les acteurs (sauf 0!), et combien d'acteurs sont nés cette année là.

**Exercice 5** (½ pt): Trouvez les artistes ayant joué dans plus d'un film

**Exercice 6** (½ pt): Trouvez les artistes ayant eu plusieurs responsabilités au cours de leur carrière (acteur, directeur, producteur...).

**Exercice 7** (¾ pt): Trouver le nom du ou des film(s) ayant le plus d'acteurs (i.e. uniquement *acted in*).

**Exercice 8** (1 pt): Montrez les artistes ayant eu plusieurs responsabilités dans un même film (ex: à la fois acteur et directeur, ou toute autre combinaison) et les titres de ces films.

# Partie 2 - Base de données graphe - Neo4j
Neo4j est une base de données graphe. Les données sont représentées par des nœuds, des relations entre les nœuds et des propriétés:
1. Les nœuds et les relations contiennent des propriétés
2. Les *relations* connectent les nœuds
3. Les *propriétés* sont des paires clé-valeur pouvant être associées à des nœuds ou des relations
4. Les relations ont des *directions*: unidirectionnelles et bidirectionnelles

## Création d'une base de données Neo4j
Pour la suite du TP, nous aurons besoin de créer une base de données Neo4j en mode "bac à sable" (Sandbox). Pour cela:
1. Créez une base de données  [Neo4j Sandbox](https://neo4j.com/sandbox/). Utilisez les informations que vous souhaitez pour la création de compte.
2. Parmi les templates, créez une base vierge (*Blank sandbox*)
3. Dans le portail Neo4j (un lien vous a été envoyé par email), dépliez la ligne correspondant à votre sandbox puis allez dans l'onglet **Connection details** pour noter ces informations de connexion
![image](https://user-images.githubusercontent.com/22498922/147907013-ae0f0d32-7982-464b-969a-576646407c9c.png)
4. ⚠️ Modifiez votre fichier `.env` les variables d'environnement nécessaires à la connexion à votre base Neo4j
    ```sh
    export TPBDD_NEO4J_SERVER=
    export TPBDD_NEO4J_USER=
    export TPBDD_NEO4J_PASSWORD=
    ```
    Définissez ces variables d'environnement dans votre shell ou redémarrez-le - sans quoi vous pointerez vers la base de données de test qui sera supprimée 1j après le TP.

## Cypher 
Le langage de requêtage utilisé par Neo4j est [Cypher](https://neo4j.com/developer/cypher/). Vous trouverez sa [documentation officielle](https://neo4j.com/docs/cypher-refcard/current/) sur le site de Neo4j.

Cypher est basé sur des **patterns**. Un pattern décrit les données et permet d'interroger le graphe par le biais d'une syntaxe visuelle très similaire à la façon dont sont généralement illustrées sur papier les données dans un graphe.

On utilise des:
* Cercles `()` pour représenter les **nœuds**
* Flèches `-->` pour représenter les **relations**

**Patterns** :
1. nœud a : `(a)`
2. une relation entre deux nœuds : `(a)–>(b)` 
3. un chemin : `(a)–>(b)<–(c)`
4. un label (ou étiquette) : `(a:label)`

Les patterns sont utilisés pour le requêtage et visent à surtout réprésenter la structure des liens entre les nœuds, comme on peut le voir dans les exemples suivants:
1. `(a)–(b)`
2. `(a)-[r]- >(b) `
3. `(a)-[r:REL_TYPE]->(b) `
4. `(a)-[:REL_TYPE]->(b) `
5. `(a)-[*]->(b)`

### Création de noeuds
Le statement `CREATE` nous permet de créer des noeuds selon la structure suivante :  
```
CREATE (nodePseudoVariable:nodeLabel { nodePropertyName: nodePropertyValue, .. })
```

Supposons que nous voulions créer deux noeuds: `Alice` et `Bob` de type `Person` avec la propriéte `name`, les requêtes seraient les suivantes: 
```
query = ("CREATE (Alice:Person { name: 'Alice' }) "
query = ("CREATE (Bob:Person { name: 'Bob' }) "
```

Nous pouvons aussi lier les noeuds par des relations, comme par exemple en indiquant que `Bob` et `Alice` se connaissent:

```
MATCH
  (a:Person),
  (b:Person)
WHERE a.name = 'Alice' AND b.name = 'Bob'
CREATE (a)-[r:KNOWS]->(b)
RETURN type(r)
```

### Création de relations
Le statement `MATCH` permet également de créer des relations entre des noeuds préalablement identifiés (*matchés*) en fonction de conditions:
```
MATCH (a:NodeLabel),(b:NodeLabel)
WHERE .....
CREATE (a)-[r:RELTYPE]->(b)
RETURN type(r)
```

Pour sélectionner tous les noeuds, utilisez la requête suivante:
```
MATCH ( n ) 
RETURN n
```

Autre exemple, pour sélectionner tous les noeuds disposant de la propriété `label`

```
MATCH (n:label) 
RETURN n
```

Pour affiner encore d'avantage, voici comment sélectionner tous les noeuds ayant une propriété `label` dont la valeur vaut `value` :


```
MATCH ( n )
WHERE n.label = 'value'
RETURN n
```

Pour supprimer des noeuds, utilisez le statement `DELETE`.

# Partie 3 - Export des données vers un modèle graphe
1. Dans le dossier `tp`, complétez le programme [export-neo4j.py](TP-Bdd-src/export-neo4j.py) aux endroits notés `A COMPLETER`. N'hésitez pas à déboguer en ajoutant des `print`, créer des programmes de test etc. Utilisez les fonctions [`create_nodes` et `create_relationships`](https://py2neo.org/2021.0/bulk/index.html) de **py2neo**.
2. Effectuez l'export vers votre base Neo4j Sandbox
3. (optionnel, 2 points bonus) Décrivez de quelle manière l'environnement de développement a été préconfiguré pour vous:
    - Environnement Python
    - Informations de connexion aux bases de données

# Partie 4 - Requêtes graphe (Cypher)
Ecrire les requêtes ci-dessous, et les expliquer en deux ou trois phrases maximum en français ou en anglais.

**Exercice 1** (¼ pt): Ajoutez une personne ayant votre prénom et votre nom dans le graphe. Verifiez qui le noeud a bien éte crée. 

**Exercice 2** (¼ pt): Ajoutez un film nommé `L'histoire de mon 20 au cours Infrastructure de donnees`

**Exercice 3** (½ pt): Ajoutez la relation `ACTED_IN` qui modélise votre participation à ce film en tant qu'acteur/actrice

**Exercice 4** (½ pt): Ajoutez deux de vos professeurs/enseignants comme réalisateurs/réalisatrices de ce film.

**Exercice 5** (½ pt): Affichez le noeud représentant l'actrice nommée `Nicole Kidman`, et visualisez son année de naissance.

**Exercice 6** (½ pt): Visualisez l'ensemble des films.

**Exercice 7** (½ pt): Trouvez les noms des artistes nés en `1963`, affichez ensuite leur nombre.

**Exercice 8** (1 pt): Trouver l'ensemble des acteurs (sans entrées doublons) qui ont joué dans plus d'un film.

**Exercice 9** (1 pt): Trouvez les artistes ayant eu plusieurs responsabilités au cours de leur carrière (acteur, directeur, producteur...).

**Exercice 10** (1 pt): Montrez les artistes ayant eu plusieurs responsabilités dans un même film (ex: à la fois acteur et directeur, ou toute autre combinaison) et les titres de ces films.

**Exercice 11** (2 pt): Trouver le nom du ou des film(s) ayant le plus d'acteurs.

## Requêtes graphe (Gremlin)
Une autre base de données graphe a été créée pour ce TP.  Elle utilise la technologie Cosmos DB et utilise un langage de requêtage différent: [Apache Gremlin](https://tinkerpop.apache.org/docs/3.3.2/reference/#graph-traversal-steps).

Vous pourrez trouver cette base, déjà préremplie, dans le portail Azure sous le nom `tpbdd-movies-cdb`. Vous pourrez y effectuer des requêtes en utilisant l'onglet **Data Explorer**.

**Exercices**: Effectuez toutes les requêtes de la section précédentes en utilisant ce langage de requêtage.

Barême: [1-¼]  [2-¼] [3-¼] [4-¼] [5-¼] [6-¼] [7-½] [8-1,5] [9-2] [10-2] [11-5 (bonus)]

## (optionnel, 3 points bonus) Encapsulation dans des APIs serverless
Implémentez quelques unes des requêtes du TP sous forme d'API Serverless avec Azure Function ou AWS Lambda. Mesurez les performances.

