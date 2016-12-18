# airflow-introduction
## 前言
最近 EZTABLE 因應紅包活動，希望提高抽紅包的人與發紅包的人參與感，因此不論是抽到紅包、發出紅包，或是更進階朋友拿紅包錢去消費你所能獲得100元的回饋諸如此類的情境，都會發信通知。由於希望寄件人呈現的方式是 ooxx via EZTABLE，支援客製化寄件人名稱的服務相對較少，平常比較常聽到的 Mailgun、Mailchimp 就被排除。一開始我們採用的是 Amazon Simple Email Service (SES)，雖然真的非常方便，但是每分鐘只能寄出 14 封信，而且很難追蹤哪些信已經寄出了，最慘的是由於 email 是個非常容易打錯的欄位，我們寄了一萬封之後就因為 email 地址錯誤太多而收到 Amazon 的警告信，為此我還寫了一封非常卑微的道歉信，但是完全沒有任何作用哈哈。

最後還是回到老路 Salseforce Marketing Cloud（以下簡稱 MC），MC 有個 Trigger Send 的服務和 SES 有點像，打資料打到相對應的端點，資料沒問題就會幫你把信放到 Queue 裡面準備寄送，但是最麻煩的一點就是，想要寄信前必須要先幫每一個要寄信的對象建立 Subscriber，每個 Subscriber 需要包含一組唯一的 Key 與正確的 Email，如果 Email 沒有通過 MC 的檢驗就無法正確建立 Subscriber，MC 在寄信前會檢查提供的 Key 與 Email 是不是和當初在 Subscriber 上設定的一樣，不一樣就會出錯。

假設今天會員 EZ 抽到 TABLE 發的紅包 250 元
我必須要跟他說：親愛的 EZ 恭喜你好手氣抽到 TABLE 送的紅包 $250！訂餐廳吃飯可直接用！
外加一封 welcome 的簡訊

並且通知 TABLE：你的朋友 EZ 抽到你發的 $250 EZCASH 紅包！只要他消費你就能得到 $100 回饋！

一開始我們的策略是直接在程式碼裡面新增 Subscriber，如果 Subscriber 已經存在，就必須更新他的 Email，確保 Subscriber 的 Email 和我們要寄的 Email 是一致的不然會出錯，但是這一段是 Soap Api，非常的慢，而且不只要建立 EZ 的會員資料，還要更新 TABLE 的 Email，步驟非常繁複，但是能確保寄信前資料已經更新完畢。但是速度真的是慢到讓人受不了，實測結果一小時最多寄送給發紅包與收紅包的人各 800 封信，處理平常的流量還可以接受，但是一遇到大流量就會卡住，造成抽紅包與收信之間的延遲。所以最後我們決定把新增 Subscriber 與寄送 Email 與簡訊的工作拆分開來。

更新 Subscriber 改用 FTP 上傳，並且設定好更新 Subscriber 的規則，一上傳成功就會觸發更新。等待更新完成再來發信。但就是等待更新這件事情讓我很感冒，非常不想要在程式碼裡面加入等待的機制，而最基本的 cron 不容易建立 job 與 job 之間的依賴性關係，最後決定採用 Airbnb 開源的 Airflow。

Airflow 有很多很棒的優點：
視覺化呈現工作狀態、相依性、執行時間
和各種服務整合（mysql、postgresql、s3、hive、slack...）
集中日誌功能
失敗/成功寄信
設置失敗重試次數

```python
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'luffy',
    'start_date': datetime.now(),
    'email': ['luffy.tsai@eztable.com'],
    'email_on_failure': False,
    'email_on_retry': False,
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


t2.set_upstream(t1)
t3.set_upstream(t2)
t4.set_upstream(t2)

```
