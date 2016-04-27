# The Fashion Logging System -- ELK - Part 2  


## 前言:  
繼[上一篇介紹什麼是 ELK ]後，這一篇我們將開始進入實作的部分，用 EC2 架設 Logstash 並搭配 AWS 提供的 Elasticsearch 與 Kibana 來架構 ELK，聽起來是不是有些小興奮呢  
首先我們會先講怎麼在 AWS 上開 ES 以及設定他，接著我們會講在 EC2 上安裝 Logstash 的步驟，最後會講怎麼樣從我們的 Logstash 送東西給 AWS 的 ES  

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
















[上一篇介紹什麼是 ELK ]: <http://ichef.github.io/The%20Fashion%20Logging%20System%20--%20ELK%20-%20Part%201/>
[AWS 的 access policy document]: <http://docs.aws.amazon.com/AmazonS3/latest/dev/access-policy-language-overview.html>