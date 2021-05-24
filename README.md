# lineBOT

## 注意事項
### 1.當開發者架設LINE Messaging API的Webhook伺服器時，只能使用 https 協定  
```
a.  HTTPS伺服器所使用的根憑證（Root CA）必須是在LINE平台的白名單列表中，否則LINE平台會拒絕傳送訊息。
    在白名單列表中大多數的憑證都需要付費申請，但是LINE平台也支援常用的免費憑證，例如Let’s Encrypt。
b.  請勿使用已知具有安全性漏洞的協定（例如SSL v2或SSL v3）或Cipher Suite（例如SWEET32或CVE-2016-2183）。
c.  請務必正確設定中繼憑證（Intermediate certificate），以避免無法對應到根憑證而發生錯誤。這是最常見的問題
    通報狀況，請在設定HTTPS伺服器時多加留意。
```
#### 測試用工具
> [ualys SSL Labs](https://www.ssllabs.com/ssltest/analyze.html)   
> [testssl.sh](https://testssl.sh/)  
> [DigiCert SSL Installation Diagnostics Tool](https://www.digicert.com/help/)  

### 2.驗證訊息來源
標準的驗證方式是檢查所收到HTTP請求標頭（HTTP request header）中的數位簽章。如果該HTTP POST訊息是來自LINE
平台，在HTTP請求標頭中一定會包括X-Line-Signature項目，該項目的內容值是即為數位簽章。例如：
```
POST /callback HTTP/1.1
X-Line-Signature: j9p1sXsb0yCyIEE8cEsmHbp9eHc85P2DQBVH1RKiQLk=
Content-Type: application/json;charset=UTF-8
Content-Length: 223
Host: callback.sample.com
Accept: */*
User-Agent: LineBotWebhook/1.0
{"events":[{"type":"message","replyToken":"1234567890abcdef1234567890abcdef","source":{"userId":"U1234567890abcdef1234567890abcdef","type":"user"},"timestamp":1500052665000,"message":{"type":"audio","id":"1234567890123"}}]}
```
> 以Channel secret作為密鑰（Secret key），使用HMAC-SHA256演算法取得HTTP請求本體（HTTP request body）的文摘值（Digest value）。  

> 將上述文摘值以Base64編碼，比對編碼後的內容與X-Line-Signature項目內容值是否相同；若是相同，表示該事件訊息是來自LINE平台，否則拒絕處理該事件訊息。  

### 3.盡快回覆LINE平台正確的HTTP狀態碼
> a.LINE平台在傳送事件訊息到開發者Webhook伺服器之後，若是等待1秒鐘沒有得到任何HTTP狀態碼的回覆，就會發生逾時（Timeout）錯誤，LINE平台會關閉該次HTTP連線並認為該次傳送結果失敗

> b.若是一直發生傳送失敗的狀況，LINE平台可能會將該Webhook伺服器封鎖或進行其他處置

較好的處理方式:

1.在Webhook收到事件訊息的程序上，先回覆LINE平台HTTP狀態碼200並關閉連線，然後以原程序直接處理。
![image](https://i.imgur.com/H7kZq2n.png)

2.Webhook收到事件訊息的程序先將事件內容儲存到一個佇列（Queue）或資料庫中，然後回覆LINE平台HTTP狀態碼200並關閉連線，結束程序。再以另外一個或多個程序依序讀取佇列或資料庫中的事件內容逐一處理。
![image](https://i.imgur.com/rjBYx4i.png)
### 4.LINE平台所傳送的事件是一個陣列
LINE平台傳送給Webhook伺服器的HTTP請求本體是包括一個或多個Webhook事件物件的JSON格式物件，ex:
```
{
  "events": [
    {
      "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
      "type": "message",
      "timestamp": 1462629479859,
      "source": {
        "type": "user",
        "userId": "U206d25c2ea6bd87c17655609a1c37cb8"
      },
      "message": {
        "id": "325708",
        "type": "text",
        "text": "Hello, world"
      }
    },
    {
      "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
      "type": "follow",
      "timestamp": 1462629479859,
      "source": {
        "type": "user",
        "userId": "U206d25c2ea6bd87c17655609a1c37cb8"
      }
    }
  ]
}
```
> #在處理LINE平台的事件請求時，必須以迴圈方式逐一讀取與處理每一個Webhook事件物件，而不是在處理完第一筆資料後便停止了。
### 5.瞭解用戶識別碼
```
a. 每一個LINE用戶帳號都有一個專屬的內部識別碼，稱為User ID。
b. User ID與LINE用戶自訂的LINE ID的格式與用途完全不同。
c. User ID的格式為33個字元的英數字字串，ex: U206d25c2ea6bd87c17655609a1c37cb8。
```
```
d. 如果開發者想要驗證一個字串是否為正確的User ID格式:

        可以使用 "正規表示式"（Regular Expression）「^U[0-9a-f]{32}$」來測試。
```
> #同一個公司或組織的User ID都有相同的(公司／組織)專屬的識別碼  

![image](https://i.imgur.com/PFF8c0j.png)

### 6.使用Reply Token的注意事項


## reference
> [開發LINE聊天機器人不可不知的十件事](https://engineering.linecorp.com/zh-hant/blog/line-device-10/)  
