# PubMed-Knowl-Graph
PubMed-Knowledge-Graph

### 建立Neo4j Graph

- Setup environment

```shell
!pip install neo4j
```

- Connect Cloud Server 連接Neo4j 雲端資料庫 https://neo4j.com/cloud/aura/

```shell
from neo4j import GraphDatabase
import pandas as pd
uri = "your_uri"
user = "your_username"
password = "your_password"
driver = GraphDatabase.driver(uri,auth=(user, password))
def neo4j_query(query, params=None):
  with driver.session() as session:
    result = session.run(query, params)
    return pd.DataFrame([r.values() for r in result],columns=result.keys())
```