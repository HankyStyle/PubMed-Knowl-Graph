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
- Create Node 有關其他Neo4j語法 可以參考我的Notion筆記 
 
  https://alpine-friction-207.notion.site/Neo4j-983d4798e63d417bba635c089f81a0e1

```shell
#create Question
neo4j_query("""
UNWIND $data as item
MERGE (a:Question {id:item})
SET a.text = item
RETURN count(a)
""",{"data":Q})
```



