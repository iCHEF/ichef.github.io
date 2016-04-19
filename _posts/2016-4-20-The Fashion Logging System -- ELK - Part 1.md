# The Fashion Logging System -- ELK - Part 1

## 前言：
在做雲端服務的時候，有一個非常重要的環節，那就是logging system！！  
因為沒有logging system來幫你蒐集log並讓你容易search的話，先不論發生bug的時候不知道要從哪邊開始找起，就連伺服器掛掉狂噴error的時候，有可能客戶都會比你早發現，到時候就...  
在威脅完大家logging system的重要性之後，這系列文章主要會分成以下三個部分去說明，讓大家對於 ELK這套logging system有大致上的了解，並且可以架設出 ELK:
1. ***什麼是 ELk ？***
2. 用 EC2架設 Logstash，並搭配 AWS提供的 Elasticsearch與 Kibana來架構ELK
3. 將 Nginx的 Log藉由 Logstash送往 AWS Elasticsearch，並用 Kibana來監測 HTTP Status

## 1. 什麼是 ELK ?
ELK分別是下面這三種東西
- E: Elasticsearch
- L: Logstash
- K: Kibana

首先我們會先從 Elasticsearch說明起

#### Elasticsearch
Elasticsearch(以下簡稱 ES)是一個建基於 Apache Lecene的開源軟體([ES Github])，他的github上第一句就寫著 ***Elasticsearch is a distributed RESTful search engine***，所以大家應該能腦補出她是什麼東西了吧～  
我們先把重點放在 search engine上，Elasticsearch可以把他想像成是一個會自動幫你做全文index讓你可以輕鬆做全文搜尋的資料庫，而他跟一般的 SQL資料庫在結構上的比對如下  
SQL: Database -> Table -> Row  
ES:  Index    -> Type  -> Document  
你要放到 ES的資料就叫做 document，它可以是非結構化的一篇文章，這樣還是可以用關鍵字來搜尋到你想要的東西，不過使用在logging system上的話你還是會想要放像是 JSON這種結構化的資料，這樣不僅閱讀更容易，在搜尋的時候也可以用 ES提供的一些方法，來讓我們使用像是 NOSQL的搜尋方式來查 document，例如去查 JSON裡的 count這個欄位要大於5的資料，這種方式可以更方便我們找到想要的log，而 ES也提供了一種很棒棒的 search語法，叫做 [DSL]，不只可以 filter，更可以做到 aggregations呢！  
談完這麼強大的 search engine，我知道各位一定很想趕快知道怎麼塞資料進去，跟在哪裡做搜尋呢？這個時候我們就要來談 RESTful了！ES不只搜尋功能強大，更是好棒棒的直接提供你 REST API來做寫入 data跟 search data的功能啊，如果你想要 insert data到某個 index的某個 type裡只要像下面這樣
```
curl -XPOST 'localhost:9200/<index: restaurant>/<type: customer>/' -d '
{
  "name": "John"
}'
```
這樣就會在 restaurant的 index下的 customer type裡新增一筆資料 { "name": "John" } ，而 ES會自動提供這個 document一個 unique ID，並在response的時候回傳包含 ID在內的相關資訊，之後可以用 put跟這個 ID去更新該筆 document
```
curl -XPUT 'localhost:9200/<index: restaurant>/<type: customer>/<ID>' -d '
{
  "name": "John Doe"
}'
```
這樣該筆 document的 name就會被更新成 John Doe  
大家有發現嗎？我們沒有介紹到怎麼創建 ES的 Index！這就像SQL DB裡沒有創建 Database跟 Table一樣荒謬啊！不過 ES強大之處就在於即使原本沒有這個 Index，只要你用 POST的 API去新增 document時，他就會自動幫你新增好你 url上的 index跟 type，不過如果你真的真的就是很想要自己創立好所有的 Index的話，其實也只要執行下面的程式就可以了
```
curl -XPUT 'localhost:9200/<index: restaurant>
```
那講完怎麼放資料之後，就要來看怎麼搜尋資料啦，就簡單的就是用 REST API來 GET資料
```
curl 'localhost:9200/<index: restaurant>/<type: customer>/xxx' -d '
```
這樣就會獲得 restaurant下的 customer的 ID為 xxx的 document，不過上面有講了 ES搜尋功能如此強大，當然不是區區 REST API可以發揮的，而是要使用 DSL的語法
```
curl -XPOST 'localhost:9200/<index: restaurant>/_search' -d '
{
  "query": { "match": { "name": "John" } }
}'
```
這樣就會回傳 restaurant這個 index下所有符合 name是 John的 document，而詳細的 DSL用法可以參考 [DSL Document]  
Elasticsearch的介紹大概到這邊，想暸解更多可以直接到 [Elastic的官網]上看 doc  

#### Kibana
Kibana簡單來說就是讓你更容易讀取 ES資料的圖像化工具，將複雜的 DSL包裝成可以直接用介面選擇的方式就完成 query同時用 query的結果畫出圖表呈現給別人看，可以說是沒有 Kibana的話，ES就像是失去雙手一樣不方便，不過當然你可以選擇自己做或用坊間的 Library去對 ES做讀取，其實不同語言都有一些 implement甚至有語言有做 stream的功能，這方面就讓大家自行摸索了，這邊主要介紹 Kibana的結構，主要分成三層，分別是：discover, visualize, dashboard  
- discover  
discover其實就是 search的語法，例如 name = John(當然也可以使用複雜的 DSL Filter)，而 discover就是負責把這些語法存起來，每次使用不同的 discover，在 discover的頁面上就會有不同的 raw document顯示在上面，而這些存起來的 discover也是丟給 visualize用的東西
- visualize  
visualize主要的功能就是依據 discover filter出來的資料來畫出圖表，你可以先選定要畫的圖表類型，接著選擇要使用的 discover，最後再根據你選的圖表類型去選擇 x軸要用 discover filter出來的 documents的哪個 field，以及該 field要經過什麼運算才顯示在 x軸，這樣就可以完成一張圖表，並且你可以將這張圖表儲存起來，要特別注意的是，圖表儲存的其實是 schema而已，也就是使用哪個 discover以及哪個 field這樣的資訊，所以你每次看該圖表的時候都有可能隨著資料變多而改變圖表的樣貌
- dashboard  
最後是 dashboard，dashboard的功能就是將 visualize畫出來的圖表組合起來，一起顯示在同一個頁面，你一樣可以把做好的 dashboard儲存起來，方便之後直接 load出來使用  

以上大概就是 Kibana的大致介紹，有興趣的人一樣可以到 [Elastic的 Kibana網頁]上觀看 Document

#### Logstash
既然介紹了怎麼用 Kibana從 ES裡讀取資料，那總是要想個辦法把資料丟到 ES裡啊！當然我們可以直接使用 ES提供的 REST API去 POST Data，不過往往我們的 Log其實都是非結構化的資料，直接丟進去的話即使用 Kibana也畫不出什麼東西，所以勢必要有一個服務可以幫我們做 formatter的工作，同時當我們的 Log來源有好多個，要送去的 ES也不同，甚至是有其他儲存 log的地方的時候，統一由一個服務來做蒐集、format、發送的工作，會更容易管理，而這就是 Logstash的場子了！  
Logstash基本上就像上面說的，負責蒐集之後做一些處理，接著把資料送到我們想要的地方，而 Logstash的設定方式也很單純，就是安裝好之後給他一個 .conf檔之後start就可以了，.conf檔的結構分成三個部分: input、filter、output，而資料處理的流程則是inputs → filters → outputs  

- input  
input就是負責蒐集資料的section，Logstash的input方式提供了許多種，你可以讓他監測 file，當 file內容有變更的時候他就會將變更的內容抓進來，你可以指定 HTTP，設定好 port之後就可以直接用 HTTP將資料送進 Logstash，甚至是你可以讓 Logstash去抓twitter裡的訊息，範例如下
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
這樣 Logstash就可以從 /tmp/access_log的檔案裡讀取內容，同時從 HTTP的8000 port讀取資料  
更多 input plugin請參考 [Logstash input plugin說明]

- filter  
把資料抓進來後，接著就是要對資料做處理，而 filter就是負責這部分的，直接舉例說明，假設今天從 input那邊吃進來了 Apache的 Log，他一定長得醜醜一大串文字，像是：
```
127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] "GET /xampp/status.php HTTP/1.1" 200 3891 "http://cadenza/xampp/navi.php" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0"
```
我們需要把它整理成一份 JSON的格式的 document，像是 {"http_method":"GET"}，這樣子才比較容易做搜尋，而 Logstash也提供的 filter的 plugin叫過 grok，grok可以自動幫你把 Apache格式的 raw data轉成結構化的資料格式
```
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    # message 是 raw data的 key name
  }
}
```
經過 filter的轉換後，資料會變成下面這樣
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
是不是美多了！Logstash一樣有提供許多 filter的 plugin，有興趣的人一樣可以看[Logstash Filter Plugin]

- output  
把資料處理好後，當然就是要把它儲存起來啦，output跟 input一樣容易，就只需要指定你想要送去的地方，不只有 ELk必備的 Elasticsearch，更可以把 log直接送到 s3封存，範例如下
```
output {
  elasticsearch {
    hosts => ["localhost:9200"]
  }
}
```
沒錯！就是這麼簡單就會幫你送去 Elasticsearch裡，Logstash一樣佛心的提供許多 output plugin，請參考 [Logstash Output Plugin]

## 小結
Part 1的文章大概介紹了 ELK的概念，希望大家不要得了名詞恐懼症！其實 ELK就是讓你可以從各個 server裡將 log吐到 logstash，接著從 logstash送到 Elasticsearch，最後從 Kibana裡去分析 log，重點就是讓傳統上分散在各個 server裡的 log，能夠處理後集中觀看，對於現在越來越大的雲端架構，以及越來越多的服務，能夠更容易地找出問題點在哪裡！  
我相信大家對於實際上要怎麼安裝環境，以及怎麼去把這三個東西接在一起會有疑問，這個部分就讓我們在接下來的系列文章裡繼續看下去～













[ES Github]: <https://github.com/elastic/elasticsearch>
[DSL]: <https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html>
[DSL Document]: <https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html>
[Elastic的官網]: <https://www.elastic.co/guide/en/elasticsearch/reference/2.0/getting-started.html>
[Elastic的 Kibana網頁]: <https://www.elastic.co/guide/en/kibana/current/index.html>
[Logstash input plugin說明]: <https://www.elastic.co/guide/en/logstash/current/input-plugins.html>
[Logstash Filter Plugin]: <https://www.elastic.co/guide/en/logstash/current/filter-plugins.html>
[Logstash Output Plugin]: <https://www.elastic.co/guide/en/logstash/current/output-plugins.html>


