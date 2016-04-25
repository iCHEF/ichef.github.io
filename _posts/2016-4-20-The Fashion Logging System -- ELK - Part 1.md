# The Fashion Logging System -- ELK - Part 1

## 前言：
在做雲端服務的時候，有一個非常重要的環節，那就是 logging system！！  
因為沒有 logging system 來幫你蒐集 log 並讓你容易 search 的話，先不論發生 bug 的時候不知道要從哪邊開始找起，就連伺服器掛掉狂噴 error 的時候，有可能客戶都會比你早發現，到時候就...  
在威脅完大家 logging system 的重要性之後，這系列文章主要會分成以下三個部分去說明，讓大家對於 ELK 這套 logging system 有大致上的了解，並且可以架設出 ELK:
  
1. ***什麼是 ELK ？***
2. 用 EC2 架設 Logstash，並搭配 AWS 提供的 Elasticsearch 與 Kibana 來架構 ELK
3. 將 Nginx 的 Log 藉由 Logstash 送往 AWS Elasticsearch，並用 Kibana 來監測 HTTP Status  

而本篇文章將針對 ***什麼是 ELK ？*** 來開始講起  

ELK 分別是下面這三種東西
- E: Elasticsearch
- L: Logstash
- K: Kibana

首先我們會先從 Elasticsearch 說明起

## Elasticsearch
Elasticsearch(以下簡稱 ES)是一個建基於 Apache Lecene 的開源軟體([ES Github])，它的 github 上第一句就寫著 ***Elasticsearch is a distributed RESTful search engine***，所以大家應該能腦補出它是什麼東西了吧～  
我們先把重點放在 search engine 上，Elasticsearch 可以把它想像成是一個會自動幫你做全文 index 讓你可以輕鬆做全文搜尋的資料庫，而它跟一般的 SQL 資料庫在結構上的比對如下  

SQL Terms | ES Terms
--------- | --------
Database  | Index
Table     | Type
Document  | Row
你要放到 ES 的資料就叫做 document，它可以是非結構化的一篇文章，這樣還是可以用關鍵字來搜尋到你想要的東西，不過使用在 logging system 上的話你還是會想要放像是 JSON 這種結構化的資料，這樣不僅閱讀更容易，在搜尋的時候也可以用 ES 提供的一些方法，來讓我們使用像是 NOSQL 的搜尋方式來查 document，例如去查 JSON 裡的 count 這個欄位要大於5的資料，這種方式可以更方便我們找到想要的 log，而 ES 也提供了一種很棒棒的 search 語法，叫做 [DSL]，不只可以 filter，更可以做到 aggregations 呢！  
談完這麼強大的 search engine，我知道各位一定很想趕快知道怎麼塞資料進去，跟在哪裡做搜尋呢？這個時候我們就要來談 RESTful 了！ES 不只搜尋功能強大，更是好棒棒的直接提供你 REST API 來做寫入 data 跟 search data 的功能啊，如果你想要 insert data 到某個 index 的某個 type 裡只要像下面這樣
```
curl -XPOST 'localhost:9200/<index: restaurant>/<type: customer>/' -d '
{
  "name": "John"
}'
```
這樣就會在 restaurant 的 index 下的 customer type 裡新增一筆資料  ` { "name": "John" } ` ，而 ES 會自動提供這個 document 一個 unique ID，並在 response 的時候回傳包含 ID 在內的相關資訊，之後可以用 put 跟這個 ID 去更新該筆 document
```
curl -XPUT 'localhost:9200/<index: restaurant>/<type: customer>/<ID>' -d '
{
  "name": "John Doe"
}'
```
這樣該筆 document 的 name 就會被更新成 John Doe  
大家有發現嗎？我們沒有介紹到怎麼創建 ES 的 Index！這就像 SQL DB 裡沒有創建 Database 跟 Table 一樣荒謬啊！不過 ES強大之處就在於即使原本沒有這個 Index，只要你用 POST 的 API 去新增 document 時，它就會自動幫你新增好你 url 上的 index 跟 type，不過如果你真的真的就是很想要自己創立好所有的 Index 的話，其實也只要執行下面的程式就可以了
```
curl -XPUT 'localhost:9200/<index: restaurant>
```
那講完怎麼放資料之後，就要來看怎麼搜尋資料啦，就簡單的就是用 REST API 來 GET 資料
```
curl 'localhost:9200/<index: restaurant>/<type: customer>/xxx' -d '
```
這樣就會獲得 restaurant 下的 customer 的 ID 為 xxx 的 document，不過上面有講了 ES 搜尋功能如此強大，當然不是區區 REST API 可以發揮的，而是要使用 DSL 的語法
```
curl -XPOST 'localhost:9200/<index: restaurant>/_search' -d '
{
  "query": { "match": { "name": "John" } }
}'
```
這樣就會回傳 restaurant 這個 index 下所有符合 name 是 John 的 document，而詳細的 DSL 用法可以參考 [DSL Document]  
Elasticsearch 的介紹大概到這邊，想暸解更多可以直接到 [Elastic 的官網上看 Document]  

## Kibana
Kibana 簡單來說就是讓你更容易讀取 ES 資料的圖像化工具，將複雜的 DSL 包裝成可以直接用介面選擇的方式就完成 query 同時用 query 的結果畫出圖表呈現給別人看，可以說是沒有 Kibana 的話，ES 就像是失去雙手一樣不方便，不過當然你可以選擇自己做或用坊間的 Library 去對 ES 做讀取，其實不同語言都有一些 implement 甚至有語言有做 stream 的功能，這方面就讓大家自行摸索了，這邊主要介紹 Kibana 的結構，主要分成三層，分別是：discover, visualize, dashboard  
- discover  
discover 其實就是 search 的語法，例如 name = John(當然也可以使用複雜的 DSL Filter)，而 discover 就是負責把這些語法存起來，每次使用不同的 discover，在 discover 的頁面上就會有不同的 raw document 顯示在上面，而這些存起來的 discover 也是丟給 visualize 用的東西
- visualize  
visualize 主要的功能就是依據 discover filter 出來的資料來畫出圖表，你可以先選定要畫的圖表類型，接著選擇要使用的 discover，最後再根據你選的圖表類型去選擇 x 軸要用 discover filter 出來的 documents 的哪個 field ，以及該 field 要經過什麼運算才顯示在 x 軸，這樣就可以完成一張圖表，並且你可以將這張圖表儲存起來，要特別注意的是，圖表儲存的其實是 schema 而已，也就是使用哪個 discover 以及哪個 field 這樣的資訊，所以你每次看該圖表的時候都有可能隨著資料變多而改變圖表的樣貌
- dashboard  
最後是 dashboard，dashboard 的功能就是將 visualize 畫出來的圖表組合起來，一起顯示在同一個頁面，你一樣可以把做好的 dashboard 儲存起來，方便之後直接 load 出來使用  

以上大概就是 Kibana 的大致介紹，有興趣的人一樣可以到 [Elastic 的 Kibana 網頁上觀看 Document]  

## Logstash
既然介紹了怎麼用 Kibana 從 ES 裡讀取資料，那總是要想個辦法把資料丟到 ES 裡啊！當然我們可以直接使用 ES 提供的 REST API 去 POST Data，不過往往我們的 Log 其實都是非結構化的資料，直接丟進去的話即使用 Kibana 也畫不出什麼東西，所以勢必要有一個服務可以幫我們做 formatter 的工作，同時當我們的 Log 來源有好多個，要送去的 ES 也不同，甚至是有其它儲存 log 的地方的時候，統一由一個服務來做蒐集、format、發送的工作，會更容易管理，而這就是 Logstash 的場子了！  
Logstash 基本上就像上面說的，負責蒐集之後做一些處理，接著把資料送到我們想要的地方，而 Logstash 的設定方式也很單純，就是安裝好之後給它一個 .conf 檔之後 start 就可以了，.conf 檔的結構分成三個部分: input、filter、output，而資料處理的流程則是 inputs → filters → outputs  

- input  
input 就是負責蒐集資料的 section，Logstash 的 input 方式提供了許多種，你可以讓它監測 file，當 file 內容有變更的時候它就會將變更的內容抓進來，你可以指定 HTTP，設定好 port 之後就可以直接用 HTTP 將資料送進 Logstash，甚至是你可以讓 Logstash 去抓 twitter 裡的訊息，範例如下
```
# logstash.conf
input {
  file {
    path => "/tmp/access_log"
  }
  http {
    port => "8000"
  }
}
```
這樣 Logstash 就可以從 /tmp/access_log 的檔案裡讀取內容，同時從 HTTP 的 8000 port 讀取資料  
更多 input plugin 請參考 [Logstash input plugin說明]

- filter  
把資料抓進來後，接著就是要對資料做處理，而 filter 就是負責這部分的，直接舉例說明，假設今天從 input 那邊吃進來了 Apache 的 Log，它一定長得醜醜一大串文字，像是：
```
127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] "GET /xampp/status.php HTTP/1.1" 200 3891 "http://cadenza/xampp/navi.php" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0"
```
我們需要把它整理成一份 JSON 的格式的 document，像是  ` {"http_method":"GET"} ` ，這樣子才比較容易做搜尋，而 Logstash 也提供的 filter 的 plugin叫過 grok，grok 可以自動幫你把 Apache 格式的 raw data 轉成結構化的資料格式
```
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    # message 是 raw data的 key name
  }
}
```
經過 filter 的轉換後，資料會變成下面這樣
```
{
        "message" => "127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] \"GET /xampp/status.php HTTP/1.1\" 200 3891 \"http://cadenza/xampp/navi.php\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0\"",
     "@timestamp" => "2013-12-11T08:01:45.000Z",
       "@version" => "1",
           "host" => "cadenza",
       "clientip" => "127.0.0.1",
          "ident" => "-",
           "auth" => "-",
      "timestamp" => "11/Dec/2013:00:01:45 -0800",
           "verb" => "GET",
        "request" => "/xampp/status.php",
    "httpversion" => "1.1",
       "response" => "200",
          "bytes" => "3891",
       "referrer" => "\"http://cadenza/xampp/navi.php\"",
          "agent" => "\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0\""
}
```
是不是美多了！Logstash 一樣有提供許多 filter 的 plugin，有興趣的人一樣可以看 [Logstash Filter Plugin]

- output  
把資料處理好後，當然就是要把它儲存起來啦，output 跟 input 一樣容易，就只需要指定你想要送去的地方，不只有 ELk 必備的 Elasticsearch，更可以把 log 直接送到 s3 封存，範例如下
```
output {
  elasticsearch {
    hosts => ["localhost:9200"]
  }
}
```
沒錯！就是這麼簡單就會幫你送去 Elasticsearch 裡，Logstash 一樣佛心的提供許多 output plugin，請參考 [Logstash Output Plugin]

## 小結
Part 1的文章大概介紹了 ELK 的概念，希望大家不要得了名詞恐懼症！其實 ELK 就是讓你可以從各個 server 裡將 log 吐到 logstash，接著從 logstash 送到 Elasticsearch，最後從 Kibana 裡去分析 log，重點就是讓傳統上分散在各個 server 裡的 log，能夠處理後集中觀看，對於現在越來越大的雲端架構，以及越來越多的服務，能夠更容易地找出問題點在哪裡！  
我相信大家對於實際上要怎麼安裝環境以及怎麼去把這三個東西接在一起會有疑問，這個部分就讓我們在接下來的系列文章裡繼續看下去～













[ES Github]: <https://github.com/elastic/elasticsearch>
[DSL]: <https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html>
[DSL Document]: <https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html>
[Elastic 的官網上看 Document]: <https://www.elastic.co/guide/en/elasticsearch/reference/2.0/getting-started.html>
[Elastic 的 Kibana 網頁上觀看 Document]: <https://www.elastic.co/guide/en/kibana/current/index.html>
[Logstash input plugin說明]: <https://www.elastic.co/guide/en/logstash/current/input-plugins.html>
[Logstash Filter Plugin]: <https://www.elastic.co/guide/en/logstash/current/filter-plugins.html>
[Logstash Output Plugin]: <https://www.elastic.co/guide/en/logstash/current/output-plugins.html>


