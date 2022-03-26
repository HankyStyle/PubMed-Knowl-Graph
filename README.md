# PubMed-Knowl-Graph
![demo](https://user-images.githubusercontent.com/70362842/151370302-779ae32c-5a78-44dc-8f11-792b96c47f16.gif)


##  Overview
本次Project是將20萬筆醫療相關文獻透過QA模型 將問題(Question) 和 答案(Answer)用 Knowledge Graph 去呈現

此Repo會教如何用Spark做資料前處理、QA模型設定 以及 用Neo4j Grpah去呈現最後成果

![image](https://user-images.githubusercontent.com/70362842/151416580-575b6519-d719-48fe-a4e7-57eb4a9b53c0.png)


## 資料分析 與 前處理

1. 資料下載
   
   有關 PubMed 200k RCT dataset 作者的介紹 可以參考 : https://github.com/Franck-Dernoncourt/pubmed-rct
   
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
   
   
   可以觀察到一個完整的資料 需要包含一個 Abstract 和 Sentence
   
   Abstract 代表 後面句子 是屬於 ['OBJECTIVE', 'METHODS', 'RESULTS', 'CONCLUSIONS', 'BACKGROUND'] 哪種性質
   
   因此在丟到QA模型前要排除 #數字 和 空白字串的資料 而且 將一筆資料分為 [Abstract,Sentence]
   
3. 資料清理
   
   ```shell
   def separate(content):
     try:
       label,sentence = content.split('\t')
       return label,sentence
     except:
       return "None"
    Real_Content = pubmed.map(lambda x : separate(x)).filter(lambda x : x != "None")
    Real_Content.count()
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151378698-9eed11cf-18e1-459d-8309-8f7f7d25c149.png)
   
   比清理前 少了3萬筆

   在觀察處理過後的資料
   ```shell
   Real_Content.take(20)
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151379194-4c7fceab-2ef4-49d6-b32f-69fb38474830.png)
   
   可以發現每筆都有包含一個Label和一段Sentence
   資料處理完後可以開始進入到QA模型的步驟

## QA模型 Setting
 本次專案是用roberta-base-squad2的閱讀理解模型 
 
 QA模型的Input是給定一段問句 和 一篇文章或句子 Output會回傳對應問句的答案
 
 簡單的Example:
 
   + Question : what is the better therapy for HIV?
   
   + Context  : In patients with advanced HIV disease , zidovudine appears to be more effective than didanosine as initial therapy ; however , some patients with advanced HIV disease may benefit from a change to didanosine therapy after as little as 8 to 16 weeks of therapy with zidovudine
   
       ![image](https://user-images.githubusercontent.com/70362842/151392874-b4ba9e14-ec00-478c-bd7c-b2463baea0f1.png)

      Model Output : **zidovudine**
   
       ![image](https://user-images.githubusercontent.com/70362842/151392795-d8c41f81-3b7a-4431-83ee-753a497b527a.png)

   從範例當中可以了解到 如果給定與問題相關的文章或句子 
   
   模型有辦法回答正確回答 問題的答案
   
   因此我們初步的想法是先挑選 有包含問句相關詞彙的文章 再和問句 一起當作Model的Input
   
   再用模型回答的Output(答案) 與 Question(問句)做成Knowledge Graph
   
  + 假定今天問 what is the therapy for the HIV?
   
   1. 先挑選 有關 HIV 與 therapy 的資料 當作 Input的Content
   ![image](https://user-images.githubusercontent.com/70362842/151400687-407119c3-2ae0-49bf-9de0-20a74072c70c.png)
   
   2. 將 Input(挑選的資料與問句) 分別 丟入模型  並將 回答的Output 、參考的文章、文章的Label 存成DataFrame的個格式 方便做Knowledge Graph的資料庫
   ![image](https://user-images.githubusercontent.com/70362842/160242589-933c1353-7e08-4e7c-af7e-6862bef487e3.png)
   
   3. 答案 與 文章的Label
   
       ![image](https://user-images.githubusercontent.com/70362842/151406170-a91545e1-d877-4d04-b800-1483d5da17bf.png)

  
   
   可以觀察到有些相同的答案 參考文章的Label也不同 
   
   之後可以根據 Label 與 問句的關係 來挑選要參考文章 說不定有時候找Label是Conclusion的文章 會比 Method來的好 (有可能Method只有提到Experiment的想法並不能當答案)
   
   
   
## 建立Neo4j Knowledge Graph

### Neo4j 介紹
Neo4j 是目前圖型化資料庫中最受歡迎的 在DB-ENGINES ranking 中長期名列前矛 近年台灣也有政府機關及企業團體慢慢導入Neo4j的技術 

如果之前有碰過SQL類似的語法 那後面的教學會很輕易的上手

以下會介紹簡單的安裝 與 資料庫語法

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
- Create Node 建立Node 

   建立Question Node
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

   建立Answer Node
   ```shell
   #create Answer
   neo4j_query("""
   UNWIND $data as item
   MERGE (a:Answer {id:item["tokens"]})
   SET a.label = item["label"]
   SET a.sentence = item["sentence"]
   RETURN count(a)
   """,{"data":paper})
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151409412-d5097fca-b0b0-460f-aaf0-6f00a255a19c.png)

   建立Label Node
   ```shell
   #create types
   types = ['OBJECTIVE', 'METHODS', 'RESULTS', 'CONCLUSIONS', 'BACKGROUND']
   neo4j_query("""
   UNWIND $data as item
   MERGE (a:LABEL {id:item})
   SET a.label = item
   RETURN count(a)
   """,{"data":types})
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151410095-95010984-2a8c-4e21-8238-a020c55e8bba.png)



- Create Relation Between Node 建立Node之間的關係

   有相同的Label 的Node 相連
   ```shell
   # Match sentence with their label
   neo4j_query("""
   MATCH (a:Answer)
   WITH a
   UNWIND a.label as type
   MATCH (types:LABEL) where types.label = type
   MERGE (a)-[:屬於]->(types)
   """)
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151410384-340ef35e-23d1-4bdf-82a8-4ac94af07899.png)


   建立Question 與 Answer 的關係
   ```shell
   # Match Question with Answer
   neo4j_query("""
   MATCH (q:Question)
   WITH q
   MATCH (ans:Answer)
   MERGE (q)-[:答案為]->(ans)
   """)
   ```
   ![image](https://user-images.githubusercontent.com/70362842/151410969-21ccaf8b-9a0b-4176-ae5e-266513deed75.png)




  有關其他Neo4j語法 可以參考我的Notion筆記 : https://alpine-friction-207.notion.site/Neo4j-983d4798e63d417bba635c089f81a0e1
  
##  Conclusion
可以發現有此專案還有蠻多地方值得去研究 

### 第1點

   比如說在挑選與問句相關的文章時 

   可以考慮用 **NER** 或是 **Word2Vec** 去做更多的文章挑選

   因為跟問句有關的文章 不一定要包含當中的字詞

   比如說 **what is the therapy for the HIV?** 也可以用 **what is the treatment for the HIV?** 來代替

   用 **treatment** 來檢索更多文章應該是值得去嘗試的
   
   
### 第2點

   可以用**Label** 篩選文章

   可能 **Method** 性質的文章 並不是正確的 文章提及的只是一種醫療實驗的方法 
   
   並沒有解答到問題 但很適合研究類型的問句 如 **what are the possible treatment for the HIV?**

   反過來 **Conclusion** 性質的文章 較適合拿來當作一般問句的Input Content

   
   
### 第3點

   Knowledge Graph 內 Node 的 Properties 和 之間的 Relation 可以更細部的設定

   以 Answer Node 來說 有 **答案** **參考文章** **參考文章的性質** 3種Key

   Question Node 來說 只有 **內容** 1種Key

   2個Node之間 可以再多增加Key 或是 更明確的關係

   像是 問句可以分等級 **what is the best therapy for the HIV?** 跟 **what is the therapy for the HIV?**

   Answer 與 Question 的關係 也能分 **答案一定是** 跟 **答案有可能是**

   或是 Answer Node **參考文章**的Value 能夠在track至其他有用文章 給模型去當作Input Content




### 以上是這次專案的介紹 如果對於這個Project有疑問的地方 或是 想與我更進一步的討論 歡迎寄信至 nchu_hank@smail.nchu.edu.tw

## Credit

- [HankyStyle](https://github.com/HankyStyle)
- 范耀中教授
- 參考網站 : https://www.kdnuggets.com/2021/12/analyzing-scientific-articles-finetuned-scibert-ner-model-neo4j.html?fbclid=IwAR2bfOlJdOMPBdwTzWNh0s4Q2rCqvx3rcB2ni_ZQYgQqWDRwgrVRpSMQK0c 、 https://neo4j.com/
