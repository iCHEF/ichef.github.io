(暫時紀錄一下，還有些 python-mysql 的情形也可以分享)
# Story

我們都知道 MySQL 的 index 很重要，會影響 query 的速度，設定 index 的時候，通常直覺會直接對 columns(fields) 直接下index。
這樣的做法在一般情形下當然是ok，速度也都還可以接受，但是當資料成長到千萬筆以上的時候，就開始出現問題了，MySQL 在index 的選擇並不會很聰明地將 index 合併在一起。

### 舉個例子
> 我有username 和 timestamp 的index，當 sql 語法下 where 的時候，MySQL 會根據你的資料量來選擇你適合的 index

> 假設以 username 撈出來會有200萬筆，timestamp 撈出來會有500萬筆，這時候 MySQL 會選擇使用 username 來做 index 的 key

> 問題來了，若符合 username 以及 timestamp 的資料可能才3萬筆，每次都要先撈200萬筆以後再去篩選嗎？這樣很慢吧！

> _*所以 MySQL 又有一種 indexing 的方法是可以合併多個 columns 當 key 的，可以把 username 以及 timestamp 當做 key，這樣 MySQL 排序篩選的時候，會加快許多*_

每個 framework 的設定方法也不一樣，這裡用 Django 當作例子

[Django Reference](https://docs.djangoproject.com/en/1.9/ref/models/options/#index-together)
```
class Meta:
  index_together = [
      ["username", "timestamp"],
  ]
```
這樣即可設定完成

# Reference
indexing 的方法有兩種，也有些差異，可以參考
[這裡](https://www.percona.com/blog/2014/01/03/multiple-column-index-vs-multiple-indexes-with-mysql-56/)
