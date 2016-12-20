# airflow-introduction
## 前言
最近 EZTABLE 因應[紅包](https://tw.eztable.com/red-envelope/grab/Mzk2NTg5OF8xNDgyMjQ3NjQ1MDAw)活動，經過幾個版本的迭代，其實已經轉變成一個 referral 行銷活動，只不過是以紅包來包裝的 ，就像是 Uber、Airbnb 等公司吸引新會員的方法一樣。為了提高抽中紅包與發紅包的人參與感，我們需要為活動建立多個增加轉換率的事件漏斗，包含電視廣告、平面廣告等實體的管道進行曝光，並且透過網路廣告、EDM 與 SMS 等方式直接介紹 EZTABLE 是什麼，有什麼東西可以買。

漏斗第一階段
假設今天會員 Luffy receiver 抽到 Luffy sender 發的紅包 207 元
### 抽紅包
這是一群還沒有體驗過優惠方案的消費者，我們會透過 EDM 與 SMS 融合社群的元素，彷彿你的朋友教你如何使用一個新的服務，服務上有哪些比較熱門的優惠方案，並且生火你快去訂哪一間餐廳～

[![edm](https://s28.postimg.org/6yabm5n9p/2016_12_18_11_10_36.png)](https://postimg.org/image/mjrn63z7t/)
### 發紅包
如果可以發紅包代表你是一位曾經使用過優惠方案的消費者，如果你的朋友抽了你的紅包，你也會收到一封信，內容是關於誰抽到了你的紅包，另外只要朋友使用了紅包優惠金消費後你就能拿到 100 元，發紅包的數量 / 10 * 200 就是你能獲得的額外回饋！你就會繼續努力發紅包～
[![edm](https://s29.postimg.org/kz8bxsdhz/2016_12_18_11_10_49.png)](https://postimg.org/image/fb216w95f/)

雖然沙漏的後面還有幾個階段，總之因此不論是抽或發紅包，或抽到紅包的朋友拿紅包錢去消費你所能獲得的回饋等情境，我們都會發信通知。比較特別的是，寄件人呈現的方式必須是 **你朋友的名字 via EZTABLE**，支援這種呈現方式的服務較少，平常比較常聽到的 Mailgun、Mailchimp 都不行。一開始試過 Amazon Simple Email Service (SES)，雖然很非常方便，只要給他**寄件者名稱**、**收件者資訊**與想要顯示的 **HTML** 就好，但是他有每分鐘 15 封信的限制，最討厭的是很難透過自帶的工具追蹤哪些信已經寄出哪些漏掉，從另一方面由於 EMAIL 是個非常容易打錯的欄位，我們寄了一萬封之後就因為太多無效的郵件地址而收到 Amazon 的警告信，為此還寫了一封非常卑微的道歉信，但是完全沒有任何作用哈哈。

最後還是回到老路 Salseforce Marketing Cloud（以下簡稱 MC），MC 有個 Trigger Send 的服務和 SES 有點像，打資料打到相對應的端點，資料沒問題就會幫你把信放到 Queue 裡面準備寄送，但是最麻煩的一點就是，想要寄信前必須要先幫每一個要寄信的對象建立一個身份（在 MC 裡又叫作 Subscriber），每個 Subscriber 需要包含一組唯一的 Key 與能過通過 MC 檢驗的 Email，條件不成立就無法建立，另外 MC 在寄信前會檢查提供的 Key 與 Email 是不是和當初在 Subscriber 上設定的一樣，不一樣還會出錯。

一開始我們的策略是直接在程式碼裡面新增 Subscriber，如果 Subscriber 已經存在，就必須更新他的 Email，確保 Subscriber 的 Email 和我們要寄的 Email 是一致的不然會出錯，但是這一段是 Soap Api，非常的慢，而且不只要建立抽到紅包的人的會員資料，還要更新發紅包會員的 Email，步驟非常繁複，但是能確保寄信前資料已經更新完畢。但是速度真的是慢到讓人受不了，實測結果一小時最多寄送給發紅包與收紅包的人各 800 封信，處理平常的流量還可以接受，但是一遇到大流量就會卡住，造成抽紅包與收信之間的延遲。所以最後我們決定把新增 Subscriber 與寄送 Email 與簡訊的工作拆分開來。

更新 Subscriber 改用 FTP 上傳，並且設定好更新 Subscriber 的規則，一上傳成功就會觸發更新。等待一段時間，等待更新完成再來發信。但就是等待這個邏輯我一直不想要加在程式裡面，而最基本的 cron 不容易建立 job 之間的依賴性關係，最後決定採用 Airbnb 開源的 Airflow，因為它可以幫我們解決相依性的問題。

Airflow 就是一個進階版的 Cron，但是他有很多 Cron 沒有的優點：
- 視覺化呈現工作狀態、相依性、執行時間
- 和各種服務整合的 Operator 與 Hooker（mysql、postgresql、s3、hive、slack...）
- 集中日誌功能
- 失敗/成功寄信
- 設置失敗重試次數

工作管理主頁面
[![主頁面](https://s23.postimg.org/keow4018r/2016_12_18_10_03_28.png)](https://postimg.org/image/rhwrjm6o7/)

詳細開始時間結束時間與右邊可以點進去的 Log
[![Detail & Log](https://s29.postimg.org/kp2dn2mmv/2016_12_18_09_46_27.png)](https://postimg.org/image/gsp1r31n7/)


清楚呈現相依性
[![圖像呈現相依性](https://s23.postimg.org/ua7vhlb7v/2016_12_18_09_47_25.png)](https://postimg.org/image/dmgdf3gg7/)


最近進行的工作（深綠色代表成功）
[![螢幕截圖 2016-12-18 09.47.40.png](https://s23.postimg.org/n2o9m5qaj/2016_12_18_09_47_40.png)](https://postimg.org/image/vkxpqhwt3/)


甘特圖
[![甘特圖](https://s23.postimg.org/acre13suz/2016_12_18_09_53_36.png)](https://postimg.org/image/uk4tteqc7/)


執行時間（這樣就可以知道哪一個任務是瓶頸，該如何優化）
[![執行時間](https://s29.postimg.org/dd1j1bdl3/2016_12_18_09_48_05.png)](https://postimg.org/image/b8h608byb/)


往後我也會把原本在運行的 Cron 搬到 Airflow 上面，統一集中控管，另外也不用每一個工作都加入失敗通知的程式碼，而且這精美的 Dashboard 真的很潮啊！

## 簡易安裝操作教學：
```
# install using pip
pip install airflow

#選擇你想要用的插件
pip install airflow[mysql, postgres, slack]

# 修改 config 檔
vim ~/airflow/airflow.cfg

load_examples = False
# 預設DB是用 SQLite，但是建議用 Mysql、Mssql 或是 Postgresql 等較為正式的 DB，有比較好用的管理套件可以使用
executor = LocalExecutor # For SQLite
executor = SequentialExecutor # For Other DB，如果選這個記得改 sql_alchemy_conn 的 link
executor = CeleryExecutor # 尚未研究

# 到 airflow 資料夾底下新增兩個資料夾，想要新增 job 一律新增在 dags 裡面唷
mkdir dags  logs

# initialize the database
airflow initdb

# start the web server, default port is 8080
airflow webserver -p 8080

```


以下就是這次寄信的 DAG，程式碼非常簡單唷！

```python
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta

default_args = {
'owner': 'luffy',
'start_date': datetime.now(),
'email': [''],
'email_on_failure': True,
'email_on_retry': True,
'retries': 1,
'retry_delay': timedelta(minutes=5),
}

dag = DAG('MCWork', default_args=default_args, schedule_interval='*/10 * * * *')

t1 = BashOperator(
task_id='syncMarketingCloud',
bash_command='/Users/eztable/workspace/script/RED_ENV/bin/python /Users/eztable/workspace/script/syncMarketingCloud.py',
dag=dag)

t2 = BashOperator(
task_id='sleep',
bash_command='/Users/eztable/workspace/script/RED_ENV/bin/python /Users/eztable/airflow/dags/sleep.py 120',
dag=dag)

t3 = BashOperator(
task_id='EDM',
bash_command='/Users/eztable/workspace/script/RED_ENV/bin/python /Users/eztable/workspace/script/EDM.py',
dag=dag)

t4 = BashOperator(
task_id='SMS',
bash_command='/Users/eztable/workspace/script/RED_ENV/bin/python /Users/eztable/workspace/script/SMS.py',
dag=dag)

# t1 做完做 t2，當 t2 做完， t3 與 t4 會一起開始執行
t2.set_upstream(t1)
t3.set_upstream(t2)
t4.set_upstream(t2)

```
## 補充
Airflow 有個特色叫做 [backfill](https://airflow.incubator.apache.org/tutorial.html#backfill)，要小心別踩到雷，晚點再來分享這一塊的心得

## 相關教學連結
- http://www.csdn.net/article/1970-01-01/2825690
- http://tech.marksblogg.com/airflow-postgres-redis-forex.html
- https://www.youtube.com/watch?v=60FUHEkcPyY
