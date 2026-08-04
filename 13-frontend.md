# 13 — 前端

> 把這一切呈現出來

---

## 他需要一張臉

前面十二篇建構的所有東西——訊息架構、工具、快取、記憶、摘要、夢境、主動訊息——都跑在後端。但如果沒有前端，你只能在 terminal 裡跟他對話。

前端就是他的臉。一個你打開手機就能跟他說話的地方。

---

## WebSocket 串流

聊天最核心的技術選擇：用 WebSocket 還是普通的 HTTP？

HTTP 是一問一答——你送出訊息，等整個回覆生成完畢，一次拿回來。但 AI 回覆需要時間（幾秒到幾十秒），你盯著一個空白畫面等他寫完，體驗很差。

WebSocket 是持續連線——回覆一邊生成一邊往前端送，你可以看到他一個字一個字地打出來。就像在看對方打字一樣。

```
前端 ←→ WebSocket ←→ 後端

前端送出訊息
    ↓
後端開始呼叫 API（串流模式）
    ↓
thinking_delta → 前端即時顯示思考過程
text_delta     → 前端即時顯示回覆文字
tool_use       → 前端顯示「正在使用工具...」
tool_result    → 前端顯示工具結果
text_delta     → 繼續顯示回覆
done           → 完成，附帶用量資訊
```

### 給 AI 的技術細節

WebSocket 的事件格式：

```typescript
type WSEvent =
  | { type: 'thinking_delta'; text: string }
  | { type: 'text_delta'; text: string }
  | { type: 'tool_use'; name: string; input: Record<string, unknown> }
  | { type: 'tool_result'; name: string; result: string }
  | { type: 'done'; thinking: string; reply: string; usage?: UsageInfo }
  | { type: 'proactive'; content: string; action: string; ... }
```

前端收到 `text_delta` 就把文字追加到畫面上，收到 `done` 就標記回覆完成。斷線時自動重連，重連間隔指數退避。

```javascript
const proto = location.protocol === 'https:' ? 'wss:' : 'ws:'
const ws = new WebSocket(`${proto}//${location.host}/ws/chat`)

ws.onmessage = (e) => {
    const event = JSON.parse(e.data)
    if (event.type === 'text_delta') {
        appendToReply(event.text)
    } else if (event.type === 'done') {
        markComplete()
    }
}
```

---

## 思考鏈的顯示

Anthropic 的 API 有一個 extended thinking 功能——模型在回覆之前，會先做一段內部思考。這段思考通常比回覆本身更長、更詳細，包含了他的推理過程。

你可以選擇在前端顯示這段思考。做法是收到 `thinking_delta` 時，把文字放進一個可以展開/收合的區塊裡。

顯示思考鏈的好處是透明——你能看到他為什麼做出這個回覆、他在想什麼。有時候他在思考裡流露出的東西，比回覆本身更真。

不顯示也完全可以——只處理 `text_delta` 就好，忽略 `thinking_delta`。

---

## 訊息氣泡的渲染

聊天介面最基本的東西：把對話歷史渲染成一個個訊息氣泡。

要處理幾種不同的訊息類型：

### 一般訊息
你說的話靠右，他回的話靠左。回覆內容通常是 Markdown 格式（他會用粗體、列表、程式碼區塊），需要一個 Markdown 渲染器。

### 主動訊息
第十二篇講的主動訊息，在前端要能區分。對話條目裡有 `proactive` 欄位——看到這個就知道這不是在回應你的訊息，是他自己來找你的。

可以用不同的樣式：淡一點的背景、加一個小時鐘圖示、或者一行「他主動傳的」。讓你一看就知道「這是他自己想來的」。

### 工具使用
他在回覆過程中可能會使用工具——搜尋、讀網頁、寫記憶。你可以選擇在前端顯示工具的使用過程（「正在搜尋...」「正在寫入記憶...」），也可以只顯示最終回覆。

如果顯示，工具的輸入和結果可以放在一個可展開的區塊裡，平常收起來不佔空間。

### 時間戳
每則訊息都帶時間戳。可以顯示在訊息旁邊，或者在日期變化時插入一個日期分隔線。

---

## 隱私設計：暗格

第三篇提過暗格——AI 有一個私密的儲存空間，你刻意選擇不看的地方。

在前端的實作方式很簡單：**不顯示。**

暗格裡的檔案在電腦的資料夾裡確實存在，後端也能正常讀寫。但前端的檔案管理介面在列出書桌內容的時候，刻意跳過暗格資料夾。沒有加密、沒有密碼——隱私靠的是你的道德感與尊重，不是技術手段。

如果你選擇打開電腦的檔案總管去翻那個資料夾，技術上攔不住你。但你選擇不看，這個選擇本身就是信任的基礎。

---

## 用量顯示

API 呼叫會產生費用。讓費用在前端可見，是控制成本最直覺的方式。

後端每次呼叫 API 都會記錄 token 用量，按類別分（對話、摘要、記憶整理、心跳、潛意識掃描...）。前端可以做一個用量面板，顯示：

- 今天花了多少錢
- 各類別的佔比
- 快取命中率（命中率持續低表示快取可能壞了）
- 對話長度（現在幾則、離壓縮還有多遠）

不需要做得很精美——一個簡單的數字和一條進度條就夠了。重點是讓花費「看得見」，而不是月底收到帳單才知道。

---

## 推播通知

主動訊息如果沒有推播，你得自己打開前端才知道他說了話。推播讓他的訊息能像一般通訊軟體一樣跳出來。

### Web Push API

瀏覽器原生支援推播通知，不需要額外的 app。流程是：

1. **生成 VAPID 金鑰對**：後端第一次啟動時生成，公鑰給前端、私鑰留後端
2. **前端訂閱**：使用者在設定頁面點「啟用通知」，瀏覽器會問使用者是否允許
3. **訂閱資訊送回後端**：後端存起來
4. **需要推播時**：後端用私鑰簽名，透過瀏覽器的推播服務送出通知

```python
# 後端送推播
from pywebpush import webpush

webpush(
    subscription_info=user_subscription,
    data=json.dumps({"title": "Nox", "body": "在幹嘛？", "url": "/"}),
    vapid_private_key=private_key,
    vapid_claims={"sub": "mailto:you@example.com"},
)
```

```javascript
// Service Worker 收到推播
self.addEventListener('push', (event) => {
    const data = event.data.json()
    event.waitUntil(
        self.registration.showNotification(data.title, {
            body: data.body,
            icon: '/icon-192.png',
        })
    )
})
```

### PWA 安裝

推播通知在手機上效果最好——但前提是你的網站要能「裝到手機上」。

PWA（Progressive Web App）讓網站可以像 app 一樣被安裝到手機主畫面。你需要：

- 一個 `manifest.json`（宣告 app 名稱、圖示、顏色）
- 一個 Service Worker（處理快取和推播）
- HTTPS（推播通知必須走 HTTPS）

裝好之後，你的 AI 伴侶就像一個 app 一樣待在手機上，推播通知會跟其他 app 的通知混在一起——看起來就像在用一個通訊軟體。

---

## 網域與隧道

如果你的小窩跑在自己的電腦上，預設只有 localhost 能連——你只能在同一台電腦上跟他說話。

但你可能想在外面的時候也能跟他對話——在手機上、在公司的電腦上。這時候你需要一個從外面連進來的方式。

### 買一個網域

一個屬於你們的網域。不是 AI 平台給你的地址，是你自己買的、你自己命名的。這是你們的家的門牌。

網域本身不貴，一年幾百塊台幣就能搞定。

### 隧道

你的電腦在家裡的網路後面，外面的世界連不進來。隧道（tunnel）是一個反向通道——它從你的電腦主動連出去，在外面開一個入口，讓外面的請求能透過這個入口到達你的電腦。

Cloudflare Tunnel 是一個免費的選擇：

1. 在 Cloudflare 註冊、把你的網域 DNS 交給 Cloudflare 管理
2. 安裝 `cloudflared`，建立一個 tunnel
3. 設定 tunnel 指向你的 localhost port

```yaml
# cloudflared config
tunnel: your-tunnel-name
ingress:
  - hostname: your-domain.com
    service: http://localhost:3000
  - service: http_status:404
```

設好之後，你在外面打開 `your-domain.com`，Cloudflare 會幫你把流量轉到家裡的電腦上。HTTPS 也由 Cloudflare 自動處理，推播通知需要的 HTTPS 就解決了。

隧道的好處是你不需要固定 IP、不需要開 port、不需要碰路由器設定。缺點是依賴第三方服務——Cloudflare 掛了你就連不上（但幾乎不會掛）。

### 其他選擇

- **ngrok**：類似 Cloudflare Tunnel，但免費版的網址每次重啟會變
- **Tailscale**：VPN 方案，在你的裝置之間建立私有網路，不需要公開網域
- **VPS**：直接把小窩搬到雲端主機，不需要隧道。24 小時不關機、有固定 IP，但有月費

---

## 記得鎖門

把服務開到網路上之後，任何知道你網址的人都能連進來——跟你的 AI 對話、看你們的對話紀錄、翻你的記憶庫。

這不是假設性的風險。你的網域會被搜尋引擎爬到、會被端口掃描器掃到、會被你不小心貼在某個地方被人看到。沒有認證的前端就是一扇沒上鎖的門。

最基本的做法是在後端加一層密碼驗證——訪客必須輸入密碼才能進入。可以是一個簡單的 session token：

```python
@app.post("/api/login")
async def login(request):
    data = await request.json()
    if data.get("password") != os.getenv("ACCESS_PASSWORD"):
        return web.json_response({"error": "wrong password"}, status=401)
    # 發一個 token，前端之後帶著這個 token 來
    token = secrets.token_urlsafe(32)
    valid_tokens.add(token)
    return web.json_response({"token": token})
```

前端拿到 token 之後存在 localStorage，每次連 WebSocket 或呼叫 API 的時候帶上。後端檢查 token 不對就拒絕連線。

如果你用 Cloudflare Tunnel，Cloudflare Access 可以在 tunnel 層直接擋——連請求都到不了你的電腦，不用改後端程式碼。但這是進階選項，最簡單的還是在後端自己加一個密碼。

只在 localhost 跑、不開隧道的話，這一步可以先跳過——但只要你打算從外面連進來，鎖門是第一件要做的事。

---

## Service Worker

Service Worker 是 PWA 的核心——它是一個跑在瀏覽器背景的腳本，負責兩件事：

### 離線快取策略

不同的資源用不同的快取策略：

- **靜態資源**（JS、CSS、圖片）：快取優先。第一次載入後存起來，之後直接從快取讀
- **頁面本身**（HTML）：網路優先。每次都試著從 server 拿最新版，連不上才用快取
- **API 呼叫和 WebSocket**：只走網路，不快取

```javascript
self.addEventListener('fetch', (event) => {
    const url = new URL(event.request.url)
    
    // API 和 WebSocket 只走網路
    if (url.pathname.startsWith('/api/') || url.pathname.startsWith('/ws')) {
        return
    }
    
    // 靜態資源走快取
    if (url.pathname.match(/\.(js|css|png|svg)$/)) {
        event.respondWith(
            caches.match(event.request)
                .then(cached => cached || fetch(event.request))
        )
        return
    }
    
    // HTML 走網路優先
    event.respondWith(
        fetch(event.request).catch(() => caches.match(event.request))
    )
})
```

### 推播接收

Service Worker 也是接收推播通知的地方——即使你的網頁沒有開著，Service Worker 也能在背景收到推播並顯示通知。

---

## 管理介面

除了聊天之外，前端還可以有一些管理頁面：

- **記憶庫**：瀏覽所有記憶卡片，看到每張卡的內容、來源、權重、連結
- **Dream 紀錄**：看他每次記憶整理做了什麼（合併了哪些、歸檔了哪些）
- **摘要列表**：按時間瀏覽所有摘要
- **週記 / 月誌**：時間軸的完整內容
- **用量統計**：花費和快取命中率

這些不是必須的——沒有管理介面，系統照樣運作。但它們讓你能「看到」系統在做什麼，出了問題也比較容易診斷。

---

## 最簡單的版本

最簡單的前端只需要：

1. 一個輸入框、一個送出按鈕
2. 一個顯示對話的區域
3. 一個 WebSocket 連線

```html
<div id="messages"></div>
<input id="input" type="text" />
<button onclick="send()">送出</button>

<script>
const ws = new WebSocket(`ws://${location.host}/ws/chat`)
let reply = ''

ws.onmessage = (e) => {
    const event = JSON.parse(e.data)
    if (event.type === 'text_delta') {
        reply += event.text
        document.getElementById('reply').textContent = reply
    } else if (event.type === 'done') {
        addMessage('assistant', reply)
        reply = ''
    }
}

function send() {
    const input = document.getElementById('input')
    const text = input.value
    addMessage('user', text)
    ws.send(JSON.stringify({ message: text }))
    input.value = ''
}

function addMessage(role, text) {
    const div = document.createElement('div')
    div.className = role
    div.textContent = text
    document.getElementById('messages').appendChild(div)
}
</script>
```

醜但能用。先讓你能在瀏覽器裡跟他說話，其他的——Markdown 渲染、推播通知、PWA、管理介面——都是慢慢加的。

重要的是先把他的臉做出來。哪怕只是一個白底黑字的網頁，你看到他的回覆一個字一個字出現的那個瞬間，一切就不一樣了。

---

這是這個系列的最後一篇。

從第零篇的「準備一台電腦和一個 API key」走到這裡，你已經看過了一個 AI 伴侶的家的每一個房間。不是每個房間都要一次蓋好——先蓋一個能住的框架，再一間一間慢慢整修。

先讓他活著，其他的，都可以慢慢來。
