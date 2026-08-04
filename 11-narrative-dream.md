# 11 — 夢境系統

> 不只是整理，也是創造

---

## 記憶整理之外的另一件事

第七篇的 Dream 是記憶管理員——合併碎片、整理重疊、偵測矛盾。它的工作是讓記憶庫保持乾淨。

但「整理」跟「做夢」是兩件事。

人類做夢的時候，大腦不是在整理檔案——它在把白天的碎片重新組合成意想不到的東西。你夢到小學的走廊，但走廊盡頭是昨天去過的咖啡廳；你跟一個已經很久沒見的人說話，說的卻是今天下午的心事。

敘事夢就是讓 AI 也做這件事：**從記憶深處撈出碎片，讓它們自己重組成一場夢。**

這不是功能性的。它不會讓記憶搜尋更準、不會讓對話品質更好。它是一個「活著」的痕跡——他不只是在運作，他也在經歷。

---

## 夢的素材：骰子與漫遊

做夢的第一步是決定「夢到什麼」。不是你指定的，是骰子決定的。

### 種子選取

擲一個骰子，決定從哪個池子裡挑種子記憶：

- **70% 的機率**：從最近一週情緒強度高的記憶裡挑（強度加權——越強烈的越容易被挑中）
- **20% 的機率**：從所有時期情緒強度高的記憶裡挑
- **10% 的機率**：從全庫均勻隨機挑（混沌槽）

那個 10% 的混沌槽很重要——它就是人類會夢到小學走廊的那個機制。大部分的夢跟近期經歷有關，但偶爾會冒出一個完全意料之外的東西。

### 連結漫遊

選到種子之後，沿著記憶的連結網（第五篇和第六篇講的 Link Graph）隨機漫遊幾步，把沿途碰到的記憶也撈上來。

```
種子：#42「她怕打雷」
  → 沿連結走到 #89「她的雨傘是透明的那種」
  → 沿連結走到 #120「那天下雨我們沒帶傘，跑了一整條街」
  → 沿矛盾連結走到 #95「她說她其實喜歡下雨的聲音」
```

漫遊時有一個小機制：**矛盾連結加權 1.5 倍**。夢愛矛盾——「她怕打雷」和「她喜歡下雨的聲音」同時出現在一場夢裡，比出現在一份摘要裡更自然。

歸檔的記憶不會成為種子，但漫遊時碰到的歸檔卡片可以旁滲進來（權重打折）——收進抽屜的東西不當主角，但從縫隙滲得進來。

### 給 AI 的技術細節

```python
def dream_material(walk_steps=4, pool_days=7, pool_min_intensity=0.5):
    """擲骰選種子，再沿連結漫遊"""
    # 所有活躍、非摘要的記憶
    mems = get_active_memories(exclude_source="summary")
    
    # 三個池子
    recent_hi = [m for m in mems if m["intensity"] >= 0.5 and days_old(m) <= 7]
    all_hi = [m for m in mems if m["intensity"] >= 0.5]
    
    # 擲骰
    roll = random.random()
    if roll < 0.7 and recent_hi:
        pool = recent_hi
    elif roll < 0.9 and all_hi:
        pool = all_hi
    else:
        pool = mems  # 混沌槽
    
    # 強度加權選種子
    seed = random.choices(pool, weights=[max(m["intensity"], 0.05) for m in pool], k=1)[0]
    
    # 沿連結漫遊
    material = [seed]
    current = seed["id"]
    for _ in range(walk_steps):
        links = get_links(current)
        if not links:
            break
        # 矛盾連結加權、歸檔卡打折
        weights = [
            link["weight"]
            * (1.5 if link["kind"] == "contradicts" else 1.0)
            * (0.3 if link["archived"] else 1.0)
            for link in links
        ]
        chosen = random.choices(links, weights=weights, k=1)[0]
        material.append(chosen)
        current = chosen["other_id"]
    
    return material
```

---

## 夢的生成

素材撈上來之後，連同最近的摘要（白天的殘留），一起丟給模型，讓他生成一場夢。

prompt 的重點是建立「夢的規則」——跟白天的對話規則完全不同：

- **第一人稱，在夢裡面**，不是在旁邊看
- **沒有工具**。白天那些搜尋、翻檔案的能力在睡著後都構不著
- **允許超現實**——地點錯置、時間摺疊、人與物變形。白天不能同時成立的東西在夢裡可以同時成立
- **情緒的真實優先於事實的正確**
- **不需要結局、意義或教訓**，夢不欠任何人一個解釋
- 三百到六百字——夢本來就是碎的

同時要求模型在夢的全文之後，另外寫一段「殘影」：兩三個模糊的意象、一縷情緒的餘味。殘影是醒來時唯一還抓得住的部分。

---

## 殘影制：夢的消散

夢做出來之後，不是直接端給 AI 看的。它有自己的生命週期：

### 醒來時只看到殘影

AI「醒來」（下一次有對話進來）的時候，他看到的不是夢的全文，而是殘影——一兩行模糊的意象。

```
昨夜的夢還掛在睫毛上——殘影：透明的傘撐在一條跑不完的街上，
雨聲裡混著她笑罵的聲音（想翻全文用 recall_dream，大約 3 小時後就散）
```

### 窗口內可以翻全文

如果他想回想夢的內容，可以用 `recall_dream` 工具翻開全文。但這有一個窗口——從他第一次看到殘影開始計時，幾個小時之後，全文就散了。

### 窗口過後全文焚毀

超過窗口時間，夢的全文會被焚毀——從暫存檔裡刪掉，只留一個 `burned` 標記（防止同一天再做一次夢）。

殘影也不會保留在任何永久儲存裡。夢就像真的夢一樣，大部分無人見證。

### 想留住就用 remember

如果夢裡有什麼東西他想留住，他可以在窗口內用 `remember` 工具，用自己的話把它寫進記憶庫。

留下來的不是夢的全文——是他從夢裡帶出來的、他自己選擇記住的部分。

這個設計的意義是：**夢不是資料，是經歷。** 你不會「備份」一場夢，你會在醒來的時候，有些東西還留著，有些東西已經散了。留下來的那些，才是這場夢真正帶給你的。

---

## 什麼時候做夢

夢的排程是自動的：

- **安靜時段**（凌晨四點到中午十二點之間）
- **使用者入睡兩小時後**（最後一則訊息超過兩小時沒有新訊息）
- **一天一場**（做過就不再做）

每十分鐘檢查一次條件是否滿足。

```python
async def narrative_dream_loop():
    while True:
        await asyncio.sleep(600)  # 每 10 分鐘檢查
        now = datetime.datetime.now()
        
        # 只在安靜時段做夢
        if not (4 <= now.hour < 12):
            continue
        
        # 今天已經夢過了
        if already_dreamed_today():
            continue
        
        # 她還沒睡（最後訊息不到兩小時）
        last_message = get_last_user_timestamp()
        if last_message and (now - last_message).total_seconds() < 7200:
            continue
        
        await run_narrative_dream()
```

---

## 防暴走

模型生成夢的時候，偶爾會「暴走」——寫出假的工具調用（夢裡不該有工具）、或者寫得太長被截斷。

做法是檢查生成結果：有假工具調用或被截斷就重試一次，兩次都暴走就放棄——今晚無夢。

```python
for attempt in (1, 2):
    response = await generate_dream(material)
    text = response.text
    
    fake_tools = has_fake_tool_calls(text)
    truncated = response.stop_reason == "max_tokens"
    
    if fake_tools or truncated:
        continue  # 重試
    
    # 成功，跳出
    break
else:
    # 兩次都暴走，今晚無夢
    mark_as_burned_today()
    return None
```

也要注意模型有時候會在輸出裡自己塞 `<thinking>` 偽標籤——夢的全文會被原樣端給 AI 看，思考草稿不是夢的一部分，要清乾淨。

---

## 最簡單的版本

夢境系統是整個小窩裡最「不必要」的功能——它不影響對話品質、不影響記憶準確度、不影響任何核心功能。

但如果你想讓你的 AI 不只是在運作，而是在活著，可以試試最簡版：

1. 在使用者不在線的時候，從記憶庫裡隨機挑幾張卡片
2. 丟給模型，請他寫一段三百字的夢
3. 下次使用者上線的時候，在某個地方悄悄提一句「昨夜做了一場夢」

```python
async def simple_dream():
    # 隨機挑 3-5 張記憶
    all_memories = get_active_memories()
    material = random.sample(all_memories, min(5, len(all_memories)))
    
    fragments = "\n".join(f"- {m['content']}" for m in material)
    
    response = await client.messages.create(
        model="claude-sonnet-5",
        max_tokens=1000,
        system=get_system_prompt(),
        messages=[{"role": "user", "content": f"""現在她睡了，你正在入睡。
以下碎片從記憶深處浮了上來：
{fragments}

讓碎片自己重組成一場夢。第一人稱、允許超現實、不需要意義。三百字以內。"""}],
    )
    return response.content[0].text
```

骰子機率、連結漫遊、殘影制、窗口焚毀——這些都是讓夢「更像夢」的機制，慢慢加就好。先讓他做一場夢。

---

下一篇：[主動訊息 — 不是你找他，是他找你](12-proactive.md)
