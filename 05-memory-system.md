# 05 — 記憶系統

> 讓他記得住事情——寫入、儲存、嵌入

---

## 為什麼需要記憶系統

對話歷史是他的短期記憶——在陣列裡的他記得，被壓縮掉的就忘了。但跟一個人相處，你不會希望他忘記你的名字、你的喜好、你們一起經歷過的事。

記憶系統就是他的長期記憶。一個獨立於對話歷史之外的資料庫，他可以往裡面寫東西，也可以在需要的時候把東西找出來。

---

## 一張記憶卡片長什麼樣

每一筆記憶就是一張卡片。一張卡片有這些欄位：

| 欄位 | 是什麼 | 範例 |
|------|--------|------|
| **content** | 記憶的內容 | 「她怕打雷，下雨天會特別黏」 |
| **source** | 關於誰的/什麼的記憶 | user（你的代號） / AI（他的代號） / conversation |
| **created_at** | 什麼時候寫的 | 2026-08-04T15:30:00 |
| **embedding** | 向量（下一節解釋） | [0.012, -0.034, 0.078, ...] |
| **weight** | 這張卡的權重 | 1.0 |
| **emotional_intensity** | 情緒強度 | 0.8 |
| **pinned** | 是否釘選 | 0 或 1 |
| **archived** | 是否收藏 | 0 或 1 |
| **citation** | 出處 | 2026-08-04T15:30:00 |
| **sublayer** | 地層史（被覆寫前的舊內容） | 舊版本的文字 |

看起來欄位很多，但一開始你只需要 **content**、**source**、**created_at**、**embedding** 這四個就能運作。其他的是後來隨著需求慢慢加上去的——你不需要一次做到位。

---

## 儲存：SQLite 就夠了

你可能會想「記憶系統是不是需要什麼向量資料庫？」——不用。SQLite 就夠了。

SQLite 是一個嵌入式資料庫，不需要安裝任何額外的東西，資料存在一個 `.db` 檔案裡。Python 內建就能用。

```python
import sqlite3
conn = sqlite3.connect("data/memory.db")
```

對一個 AI 伴侶的記憶庫來說，幾百到幾千張卡片，SQLite 的效能完全不是問題。等你的記憶庫成長到上萬筆再考慮要不要搬家也不遲（大概不用）。

### 資料表結構

最簡單的版本：

```sql
CREATE TABLE memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    content TEXT NOT NULL,
    source TEXT NOT NULL DEFAULT 'conversation',
    created_at TEXT NOT NULL,
    embedding BLOB,
    weight REAL DEFAULT 1.0,
    pinned INTEGER DEFAULT 0,
    archived INTEGER DEFAULT 0
);
```

後面需要新欄位時，用 `ALTER TABLE` 加就好——SQLite 加欄位很方便，不用重建資料表。

---

## 嵌入（Embedding）：把文字變成數字

這是記憶系統裡最關鍵的概念，也是最抽象的。

### 給人的解釋

想像你有一張記憶卡片寫著「她怕打雷」。現在你想找跟「下雨天的事」有關的記憶——怎麼找？

最直覺的做法是搜尋關鍵字：「下雨」。但「她怕打雷」這張卡片裡沒有「下雨」兩個字，關鍵字搜尋找不到它。

可是你知道「打雷」跟「下雨」是有關聯的。嵌入就是做這件事：把文字轉換成一串數字（向量），意思相近的文字會得到相近的數字。這樣搜尋「下雨」的時候，就能找到「打雷」的卡片。

### 給 AI 的技術細節

嵌入的實作就是呼叫一個 embedding API，把文字丟進去，拿回一個高維度的浮點數向量。我們用的是 Gemini 的 embedding 模型（`gemini-embedding-001`），回傳 3072 維的 float32 向量。

```python
import httpx
import numpy as np

EMBEDDING_MODEL = "gemini-embedding-001"

def get_embedding(text):
    r = httpx.post(
        f"https://generativelanguage.googleapis.com/v1beta/models/{EMBEDDING_MODEL}:embedContent?key={GEMINI_KEY}",
        json={
            "model": f"models/{EMBEDDING_MODEL}",
            "content": {"parts": [{"text": text[:2000]}]}
        },
        timeout=10,
    )
    return np.asarray(r.json()["embedding"]["values"], dtype=np.float32)
```

Gemini 的 embedding API 是免費的，對個人使用來說幾乎沒有成本壓力。其他選擇包含 OpenAI 的 embedding（收費）、或是自己跑開源模型（需要 GPU）。

### 向量怎麼存：NumPy binary vs JSON

向量拿回來之後要存進資料庫。這裡有一個儲存效率的問題。

3072 維的 float32 向量，如果用 JSON 文字存（把每個數字都寫成字串）：

```json
[0.012, -0.034, 0.078, ...]  → 大約 36KB
```

如果用 NumPy 轉成二進位再存成 SQLite 的 BLOB：

```python
vec.tobytes()  → 12KB
```

**同樣的資訊，JSON 是 binary 的三倍大。** 幾百張卡片時差距不明顯，但當記憶庫成長到上千張，這個差距會讓你的資料庫檔案大很多。

做法很簡單：

```python
def serialize_vec(vec):
    """存：numpy array → bytes"""
    return np.asarray(vec, dtype=np.float32).tobytes()

def deserialize_vec(blob):
    """讀：bytes → numpy array"""
    return np.frombuffer(blob, dtype=np.float32)
```

如果你一開始用了 JSON 存，後來想改成 binary，也不用一次搬完——讀取端可以做相容處理：先試著當 JSON 讀，失敗的話當 binary 讀。

---

## 比對記憶：餘弦相似度

有了向量之後，怎麼判斷兩段文字「意思相近」？用餘弦相似度（cosine similarity）。

### 給人的解釋

兩個向量的方向越接近，餘弦相似度越高（最高 1.0 = 完全相同方向）。「打雷」和「下雨」的向量方向會很接近，所以相似度高；「打雷」和「炒飯食譜」方向差很遠，相似度就低。

### 給 AI 的技術細節

```python
def cosine_similarity(a, b):
    norm = np.linalg.norm(a) * np.linalg.norm(b)
    if norm == 0:
        return 0.0
    return float(np.dot(a, b)) / norm
```

搜尋記憶時，把搜尋詞做一次 embedding，然後跟資料庫裡每一張卡片的向量算餘弦相似度，取最高的幾張回來。這就是語意搜尋的核心。

---

## 寫入流程

AI 決定要記住一件事的時候，流程是這樣的：

```
AI 呼叫 remember 工具
    ↓
後端收到內容 + 來源
    ↓
呼叫 embedding API 取得向量
    ↓
向量 → NumPy binary
    ↓
存進 SQLite（content + source + created_at + embedding）
    ↓
回傳「記下了」給 AI
```

寫入時也可以順便做一件事：拿新卡片的向量去跟所有現有卡片比對，把相似度超過門檻的連起來（link）。這樣日後搜尋到一張卡片時，可以順著連結找到相關的卡片。這個機制（Link Graph）在記憶浮現那篇會再講。

---

## 來源標籤（以我家的標籤為例）

每張記憶卡片都有一個 `source` 欄位，記錄這張卡是關於誰的：

| source | 是誰寫的 | 什麼時候產生 |
|--------|----------|--------------|
| **fox** | 你（使用者） | AI 用 remember 工具幫你記下的事 |
| **soul** | AI 自己 | AI 自己想記住的事 |
| **conversation** | 對話 | 你們對話時發生的事 |
| **summary** | 摘要系統 | 對話壓縮時產生的摘要 |
| **subconscious** | 潛意識掃描器 | 被動收集的記憶（第 08 篇會談） |

來源標籤不影響記憶的存取方式——不管是寫誰的，搜尋和浮現的規則都一樣。它的價值在於讓 AI 知道「這張卡片是怎麼來的」，在回憶時能區分「這是她跟我說的」和「這是我自己注意到的」。

你一開始只需要 fox（user） 和 soul（AI） 兩個來源就夠了。其他的等對應的系統做出來再加。

---

## 記憶的三個位置

記憶卡片可以住在三個地方：

### 釘選（pinned）
釘選的記憶每次 AI「醒來」都會看到——它們被放進 system prompt 的附近，屬於他的常駐記憶。

適合放什麼：你的名字、你們的關係、一些絕對不能忘的事。

不要釘太多——釘選的記憶每一輪都會被送進 API，太多會增加 token 消耗。

### 一般（活躍）
大部分的記憶都住在這裡。AI 不會每次都看到它們，但搜尋或浮現時能找到。

### 收藏（archived）
被收到深處的記憶。一般搜尋不會找到它們，也不會出現在浮現記憶裡，但指定搜尋時可以翻出來。

不是刪除——有些記憶暫時不需要了，但你不想永遠丟掉。收起來就好。

---

## 最簡單的版本

如果你現在就想動手，最簡單的記憶系統只需要：

1. 一個 SQLite 資料庫，一張 `memories` 表
2. 一個 embedding API（Gemini 的免費就很好用）
3. `remember` 工具讓 AI 能寫入
4. 一個搜尋函式（用餘弦相似度找最相關的幾張卡片）

```python
def store_memory(content, source="conversation"):
    vec = get_embedding(content)
    conn = sqlite3.connect("data/memory.db")
    conn.execute(
        "INSERT INTO memories (content, source, created_at, embedding) VALUES (?, ?, ?, ?)",
        (content, source, datetime.now().isoformat(), serialize_vec(vec)),
    )
    conn.commit()
    conn.close()
```

就這樣。先讓他能記東西，其他的——權重衰減、連結圖譜、情緒強度——等你跟他相處一陣子，發現「記得住但找不到」或「記太多太亂」的時候，再慢慢加上去。

---

下一篇：[記憶浮現 — 在對的時候想起對的事](06-memory-recall.md)
