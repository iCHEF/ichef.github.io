# The Fashion Logging System -- ELK - Part 2  


## 前言:  
繼[上一篇介紹什麼是 ELK ]後，這一篇我們將開始進入實作的部分，用 EC2 架設 Logstash 並搭配 AWS 提供的 Elasticsearch 與 Kibana 來架構 ELK，聽起來是不是有些小興奮呢  
首先我們會先講怎麼在 AWS 上開 ES 以及設定他，接著我們會講在 EC2 上安裝 Logstash 的步驟，最後會講怎麼樣從我們的 Logstash 送東西給 AWS 的 ES，並在 kibana 上看到那些資料  

## 設定 AWS 的 Elasticsearch Service
首先盡到你的 AWS console，接著在所有服務的 Analytics 區塊裡可以看到 Elasticsearch Service，按下去之後你就會來到 ES 的  Dashboard 裡了！接著按 `Create a new domain` ，這個 domain 可以想像成是一個一個的 ES 資料庫，可以讓你來放跟不同類型的資料，進去創立 domain 的頁面後，先輸入你想要的 domain name，接著選擇符合你的使用量的 instance，接著設定 access policy，來決定這台 ES 可以讓誰 access，這邊我們為了方便，只設定兩個簡單的 rule，第一個是讓 Logstash 的 server 可以做 HTTP POST，這樣就可以新增資料，接著是讓所有人都可以對 ES 做 HTTP GET，讓大家都可以從 ES 裡獲取資訊，以下是 access policy範例：
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:ESHttpPost",
      "Resource": "arn:aws:es:ap-northeast-1:209570776318:domain/ichef/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "<Logstash Server IP>"
        }
      }
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:ESHttpGet",
      "Resource": "arn:aws:es:ap-northeast-1:209570776318:domain/ichef/*"
    }
  ]
}
```
更安全詳細的 access policy 設定可以參考 [AWS 的 access policy document]  
最後按下 `Confirm and create` 就完成開啟 ES 的任務了！
回到 Dashboard 後可以在側邊欄看到你新創立的 domain，點下去後就會開心地看到 Domain status 等等的相關資訊了，重點是那個 Endpoint，那是讓你的 Logstash 可以送資料進去的 url

## 在 EC2 上架設 Logstash  
在開啟 ES 之後當然就是要來架設 Logstash 了，只要簡單的三步驟就可以完成囉！  

### step 1: 開啟 EC2
基本上就是到你的 AWS EC2 Console 裡面，按 create instance 之後照著做就好了，唯一要記得的是 AMI 的部分記得選 ubuntu 或是 centos，選你自己的 AMI 也可以，不要用 windows 就好了！  
記得配一個 Elastic IP 給這台 EC2，因為我們上個步驟的 ES 有指定只有哪個 IP 的 server 可以送資料進去，然後要回 ES 的 access policy 裡面把 Logstash 這台 EC2 的 IP 加進去
然後 ssh 進到這台 EC2，準備進行接下來的步驟！

### step 2: Install Logstash
首先要下載官網裡的 Logstash package
```
[root@logstash ~] wget https://download.elasticsearch.org/logstash/logstash/packages/centos/logstash-2.2.1.noarch.rpm
```
接著把它安裝起來
```
[root@logstash ~] yum install logstash-2.2.1.noarch.rpm
```
然後就完成了  
沒錯就是這麼簡單的讓我們進入第三步

### step 3: Configuration and start
安裝完了之後呢，就是要按照我們上一篇所講的 Logstash 的設定方式來設定了，只要編輯下面的 .conf 檔即可
```
[root@logstash ~]  vi /etc/logstash/conf.d/logstash.conf
```
這個就是 Logstash 主要的設定檔了，把上次講的設定一古腦放進去就對了！
```
input {
  file {
    path => "/tmp/access_log"
  }
}
output {
  elasticsearch {
    hosts => ["<ES Endpoint>:80"]
  }
  file {
    path => "/opt/logstash/logstash_log"
  }
}
```
接著
```
[root@logstash ~] service logstash restart
```
你的 Logstash 就開始動了！你可以用下面的指令來檢查狀態
```
[root@logstash ~] service logstash status
```
要確定它出來的狀態是有啟動的，如果狀態是 not running 的話，你要到下面的位置看一下 error log
```
[root@logstash ~] cd /var/log/logstash/
```
到這資料夾下找到該日期的 log，就可以看到啟動不起來的原因了，雖然大多數時候都是 config 檔的格式寫錯啦...

### 測試
我們上面寫的 conf 檔主要是來簡單的測試，當我們往 `/tmp/access_log` 寫一點東西進去的時候，Logstash 會幫我們把 access_log 裡的文字丟到 ES 裡同時也會丟到 `/opt/logstash/logstash_log` 這個檔案裡，讓我們來試試看吧
```
[root@logstash ~] echo "Hello" > /tmp/access_log
```
這時你到 `/opt/logstash/logstash_log` 這個檔案裡應該會看到一個 `Hello`，這樣 Logstash 就算設置完成了！ 
而送到 ES 的資料怎麼看我們會在下面 kibana 的部分說明  

***備註1：*** 很多網路上說明 ELK 的文章裡，往往 Logstash output 到 ES 的設定，它的 hosts 的 port 都是9200，這是因為如果 ES 是你自己架設的話，官網的說明也都是放在 9200 port，但是因為我們是使用 AWS 所提供的 ES service，所以 AWS 是直接使用 HTTP 的 80 port，而且要注意的是這個 80 port 一定要指定，不然 Logstash 對 ES 的 output，預設都是 9200 port！  
***備註2：*** 幫大家複習一下 [Logstash 的 config 設定]  

## 用 kibana 來觀看 ES 內的資料
首先讓我們回到 AWS ES 的 Dashboard，在側邊欄選擇你的 domain，你可以看到有一個 Kibana 的連結，點下去就對了！  
因為是第一次進來 kibana，所以你會來到 kibana 的設定頁面，它會叫你 Configure an index pattern，這個意思就是說這次你想要看哪個 Index，白話翻譯就是要看哪個 DB，不過我們剛剛根本沒有設定 Logstash 要送到哪個 Index 啊！這樣我哪知道要看哪個 Index 啊！  
沒問題！智慧的 Logstash 在你沒有額外設定的情況下，它會依照日期送到不同的 Index，舉個例假設今天是 2016年 4月 15號，那 Logstash 就會自動把資料送到 name 為 `logstash-2016.04.15` 的 Index 裡，所以你只要在輸入框裡輸入 `logstash-2016.04.15`，再按下下面的 Create，就可以在 kibana 的 discover 裡看到我們剛剛送進來的 `Hello` 了！這樣我們的 Logstash 送資料到 ES 就算成功了！  
但是如果你想要看所有從 Logstash 送進來的資料要怎麼辦呢？總不能一天一天看啊？沒問題！kibana 在設定的標題上都說了要用 index pattern 了，所以你只要輸入 `logstash-*` 就可以看到所有名稱最前面是 logstash 的 Index 了，是不是非常方便呢，而這功能最重要的是在實際使用時，雖然資料都是從 logstash 送過來，但是可能有不同類型的資料，例如 nginx 的 log，app 的 log 等等，你就可以用 `logstash-nginx-log-*` 這樣的 prefix 來當作該種類型資料的 Index 名稱，方便你搜尋也方便你只刪除某些比較不重要的類型的資料

## 小結
上面大概說明了怎麼利用 AWS 提供的 ES Service 搭配自己架設的 Logstash 來完成 ELK 的系統架構，在實際使用的時候還有 permission 的問題得要好好思考怎麼設定才安全，以及 Logstash 要怎麼設定 filter，讓送進去 ES 的資料可供搜尋與分析，下一篇文章會說明怎麼利用這套架構來實作監控 nginx server 健康狀態的 Dashboard，敬請期待！

## 後話
其實對 cluster 的概念掌握得好的話，會建議 ES 可以自己架設，雖然 AWS 提供了很方便的服務讓你按一按就架起服務了，可是其實 AWS 的 ES service 算是半殘版本，ES 本身提供了多種自我管理的 API，只要在 ES 的 domain後面加上 `/_<API Name>`，就可以使用了，像是可以看到 ES 狀態的 API，只要訪問 `https://<ES domain>/_status/` 就可以看到了，而 AWS 提供的是的 API 都列在這篇 [AWS ES Operations]裡了，不過可以看到的是並不像 ES 官方所提供的那麼完整  
另外最可惜的一件事是 ***plugin！*** 使用 ES 的 plugin 也很簡單，就是在 ES domain 後面加上 `/_plugin/<plugin name>/`，可以看到上面的 kibana 的網址就是 `https://<ES domain>/_plugin/kibana/`，而 AWS 只提供了一些 ES 內建的 plugin，但是這世界上有好多善心人士開發了許多的 ES plugin，像是幫 ES 加上 websocket 的功能，這樣就可以容易的 stream 出資料，做到像 [papertrial] 一樣的事情了！不過 AWS 的 ES service 並不提供自行安裝 plugin 的功能，所以這世界上的 plugin 都跟你無關了...  
這大概就是方便使用的代價吧













[上一篇介紹什麼是 ELK ]: <http://ichef.github.io/The%20Fashion%20Logging%20System%20--%20ELK%20-%20Part%201/>
[AWS 的 access policy document]: <http://docs.aws.amazon.com/AmazonS3/latest/dev/access-policy-language-overview.html>
[Logstash 的 config 設定]: <https://www.elastic.co/guide/en/logstash/current/configuration.html>
[AWS ES Operations]: <http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-gsg-supported-operations.html>
[papertrial]: <https://papertrailapp.com/>