# PubMed-Knowl-Graph
![demo](https://user-images.githubusercontent.com/70362842/151370302-779ae32c-5a78-44dc-8f11-792b96c47f16.gif)


##  Overview
本次Project是將20萬筆醫療相關文獻透過QA模型 將問題(Question) 和 答案(Answer)用 Knowledge Graph 去呈現

此Repo會教如何用Spark做資料前處理、QA模型設定 以及 用Neo4j Grpah去呈現最後成果

## 資料分析 與 前處理

1. 資料下載

   ```shell
   !wget https://raw.githubusercontent.com/Franck-Dernoncourt/pubmed-rct/master/PubMed_20k_RCT/train.txt
   ```

2. 資料觀察

   ```shell
   pubmed = sc.textFile("./train.txt")
   pubmed.count()
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151376340-ff4501ab-90a0-46ed-85da-5dd20d233828.png)
   總筆數大概21萬

  
   ```shell
   !head -n 20 train.txt
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151374192-770df96d-2db5-41a6-91ae-f9d1dcec2889.png)
   可以觀察到一個完整的句子 需要包含一個 abstract 和 sentence 
   
   因此在丟到QA模型前要排除 #數字 和 空白字串的資料 而且 將一筆資料分為 [Abstract,Sentence]
   
3. 資料清理
   
   ```shell
   def separate(content):
     try:
       abstract,sentence = content.split('\t')
       return abstract,sentence
     except:
       return "None"
    Real_Content = pubmed.map(lambda x : separate(x)).filter(lambda x : x != "None")
    Real_Content.count()
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151378698-9eed11cf-18e1-459d-8309-8f7f7d25c149.png)
   比清理前 少了3萬筆

   在觀察處理過後的20筆
   ```shell
   Real_Content.take(20)
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151378955-0eb5b5ad-6f17-4ed9-b1aa-be6967ddfb05.png)
   可以發現每筆都有包含一種Abstract和一段Sentence
   
   
## 建立Neo4j Graph

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


