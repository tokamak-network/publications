---
layout: post
title:   eth.events 활용하여 이더리움 블록체인 데이터 가져오기.
date:   2019-4-18 04:37:00 +09:00
author: "신진환(Jin)"
categories: ethereum general
tag: [ethereum, general]
youtubeId :
slideWebId :
---
![eth.events메인화면](/images/2019-04-18-eth-Event-dapp-1.png)

안녕하세요,

이번 글에서는 이더리움 블록체인 데이터를 더 쉽게 접근 할 수 있는 eth.events 라는 서비스를 하나 소개해 드리고자 합니다.
이더리움 블록체인 데이터를 가져오기 위해서는 보통 go-ethereum 이나 parity 노드에서 json-rpc를 통해 block, transaction, receipt 등의 정보를 가져오게 됩니다.
수시로 데이터를 분석하는 분들께서는 database를 직접 구축하여 노드로부터 데이터를 직접 insert하기도 하죠.
그러기엔 너무 많은 시간과 노력이 들어가게 됩니다. 사실 너무 귀찮지요.

소개해드릴 eth.events 에서는 이더리움 블록체인 데이터를 SQL 쿼리로 가져올 수 있습니다! 가입하고 API키를 받으면 postgresql database의 account를 받습니다. 무료로요. 사이트는 [https://eth.events/](https://eth.events/) 입니다.
심지어 ElasticSearch도 지원을 한답니다.

간단하게 사용법을 알아 볼까요?

## 가입 및 Key 얻기

아래처럼 다양한 인증방법으로 간단하게 가입을 하고...

![가입화면](/images/2019-04-18-eth-Event-dapp-2.png){: width="50%" height="50%"}

완료하면 바로 키를 발급 받을 수 있습니다. 참 쉽죠?

![가입완료](/images/2019-04-18-eth-Event-dapp-3.png){: width="80%" height="90%"}

`Get your free API key` 를 선택하여 들어가보면 아래처럼 나옵니다.

![API키정보](/images/2019-04-18-eth-Event-dapp-4.png)

이제 키를 생성하였으니, 데이터를 가져와 봅시다.

## 데이터 가져오기

API Key 아래 SQL Access에 있는 정보들이 이더리움 블록체인데이터가 담겨있는 Postgresql Database 접속 정보입니다. 자 이제 [eth.events](http://eth.events) 에서 제공하고 있는 데이터베이스에 접근을 해보겠습니다.

Python을 통해서 간단한 쿼리를 날려보도록 하겠습니다. 필요한 라이브러리는 psycopg2 와 pandas 를 이용하겠습니다. (psycopg2는 postgresql 드라이버, pandas는 데이터 분석용 라이브러리 입니다.)

```python
import psycopg2
import pandas as pd
import pandas.io.sql as sqlio

con = psycopg2.connect(database="ethereum_ethereum_mainnet",
                        user="6c...",
                        password="1c...",
                        host="sql.eth.events", port=45432)
```

접속정보를 잘 적어 놓습니다. SQL 구문이 친숙하신분들도 계시겠지만, 낮선분들을 위해서 링크를 하나 남겨드리겠습니다. [https://www.codecademy.com/learn/learn-sql/modules/learn-sql-queries](https://www.codecademy.com/learn/learn-sql/modules/learn-sql-queries)
queries 파트만 해보셔도 데이터를 가져오는데는 충분합니다^^.
아래 `query` 문을 잠시 살펴보자면, `tx` 라는 테이블에서 `0x89d24A6b4CcB1B6fAA2625fE562bDD9a23260359` 주소로 전송되는 transaction 들 중 2019년 1월 1일 이후 데이터를 모두 가져온다 라는 뜻입니다.
위 0x89로 시작하는 주소는 Dai 토큰 컨트랙트 주소입니다. Transaction 수만 12만번이 넘습니다!

[https://etherscan.io/address/0x89d24a6b4ccb1b6faa2625fe562bdd9a23260359](https://etherscan.io/address/0x89d24a6b4ccb1b6faa2625fe562bdd9a23260359) Dai 토큰의 etherscan 정보입니다.
```python
query = "SELECT * FROM tx WHERE tx.to = '0x89d24A6b4CcB1B6fAA2625fE562bDD9a23260359' AND timestamp > '2019-01-01 00:00:00'"
    
df_dai = sqlio.read_sql_query(query, con)
df_dai.head(1)
```

|---
| block_hash | id | block_number | from    | gas                                        | gas_price | hash        | creates                                           | public_key | input                                             || timestamp | v                      | r    | s                                                 | contract_address                                  | cumulative_gas_used | gas_used | logs_bloom | root                                              | status | 
|---------------------------------------------------|----|---------------------------------------------------|---------|--------------------------------------------|-----------|-------------|---------------------------------------------------|------------|---------------------------------------------------|---------------------------------------------------|-----------|------------------------|------|---------------------------------------------------|---------------------------------------------------|---------------------|----------|------------|---------------------------------------------------|--------| 
| 0xf6c3bf9408aa6dddf284a000eeceb229b209934417a9... | 0  | 0x18cbe9cd875c7b95b72beebbf5e607b6d6b0ba1c3630... | 7548484 | 0xDe3cb7b77Ce088fE787031C1F09c51e8089b371b | 61860     | 2500000000  | 0xf6c3bf9408aa6dddf284a000eeceb229b209934417a9... | None       | 0xc727c4790618be1f55ed34718a68bd2ee38dfd8e7dbc... | 0x095ea7b300000000000000000000000039755357759c... | ...       | "Apr 12, 2019 4:00 AM" | 0x26 | 0xa9ee69260f13f03db2edf737b90d95066ae0b4b4d760... | 0x46725004b78ec4a51728727857dceb07f6f7ab535516... | None 
|---
| 1 rows × 24 columns
|===

짜잔! pandas를 사용하면 테이블로 깔끔하게 데이터를 살펴볼수 있습니다.

## 사용후기 및 더 알아보기

이더리움 블록체인을 살펴볼 수 있는 [etherscan.io](http://etherscan.io) 와 [ethplorer.io](http://ethplorer.io)  있지만, 이렇게 SQL쿼리로 그리고 대량으로 데이터를 제공해주지는 않지요. ( etherscan.io의 API를 통해서는 한번에 10,000개씩 요청 할 수 있습니다.)

[https://docs.eth.events/en/latest/sql/authorization/index.html](https://docs.eth.events/en/latest/sql/authorization/index.html) 에서 명시하였듯, SQL은 아직 베타 상태입니다.

실제 사용하면서 다른 주소로 SQL 쿼리를 요청했었는데요, 몇개의 주소는 아예 데이터가 없거나 들어오지 않는 부분이 있었습니다. 서비스가 점차 안정화된다면, db 구축없이 빠르게 데이터를 가져올 수 있는 좋은 서비스임에 틀림 없네요

더 자세히 사용해보고, 알아보고싶으신분께서는 문서를 한번 보시면 좋을 것 같습니다.

[https://docs.eth.events/en/latest/index.html](https://docs.eth.events/en/latest/index.html)