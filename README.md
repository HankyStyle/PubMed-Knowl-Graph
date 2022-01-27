# PubMed-Knowl-Graph
![demo](https://user-images.githubusercontent.com/70362842/151370302-779ae32c-5a78-44dc-8f11-792b96c47f16.gif)


##  Overview
本次Project是將2萬筆醫療相關文獻透過QA模型 將問題(Question) 和 答案(Answer)用 Knowledge Graph 去呈現
此Repo會教如何做資料前處理、QA模型設定 以及 用Neo4j Grpah去呈現最後成果

##  Datacleaning

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
- Create Node 

```shell
#create Question
neo4j_query("""
UNWIND $data as item
MERGE (a:Question {id:item})
SET a.text = item
RETURN count(a)
""",{"data":Q})
```
![image](https://user-images.githubusercontent.com/70362842/151332064-3834e610-e601-4a96-8762-0c87d240a683.png) ![image](https://user-images.githubusercontent.com/70362842/151332176-b13c69b2-c38a-4d84-a59c-8a739fee1283.png)



  有關其他Neo4j語法 可以參考我的Notion筆記 : https://alpine-friction-207.notion.site/Neo4j-983d4798e63d417bba635c089f81a0e1


