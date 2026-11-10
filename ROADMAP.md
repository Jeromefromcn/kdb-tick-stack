# kdb+ / q 實操 Roadmap

> 一條主線項目：一個迷你 kdb+tick stack，由 Java feedhandler 接加密貨幣公開行情。
> 11 個 Phase，每個都是**能跑的增量**。目的是把 q 語言、tick 架構、as-of join、
> 日誌重放恢復這些概念真正做一遍，並留下測量數字和決策記錄。

## 怎麼用這份手冊

- 每個 Phase 先寫一頁 spec（`docs/specs/<nn>-<slug>.md`），再寫代碼。
- 決策記錄放 `docs/decisions/<nn>-<slug>.md`，測量數字放 `docs/evidence/`。
- **停止線在 Phase 7。** Phase 8–10 是延伸，有明確需要時再做。
- 節奏：每個 Phase 約 1–3 個晚上。

## 為甚麼用加密貨幣行情

免費、7×24 小時有數據、不用 API key；成交（trade）和最優報價（bookTicker）兩條
流剛好對應 kdb+tick 的經典 `trade` / `quote` 兩張表。

## 架構

```
 Binance public websocket (trade + bookTicker)
        │
 Java Feedhandler  ──(javakdb IPC)──►  Tickerplant :5010 ──► tp log
                                           │
                       ┌───────────────────┼──────────────────┐
                       ▼                   ▼                  ▼
                    RDB :5011         RTE :5012 (VWAP、       (EOD)
                       │              價差告警)                │
                       └──────── Gateway :5000 ◄──── HDB :5013 ◄┘
                                     ▲
                          Spring Boot 查詢 API（Phase 9）
```

標 🔍 的是要在官方文檔或安裝時核實的事項。

---

## Phase 0 · 安裝與 REPL（1 晚）

- 申請 **KDB-X Community Edition** 免費授權（個人和商業項目都可用）。限制：單一實例、
  q 總共最多 16GB RAM、每進程最多 4 個 secondary threads 和 16 個連接、最多 24 核。
  本項目的 5 個進程在同一台機器上，都在限制內。（kdb+ Personal Edition 僅限非商業用途、
  12 個月有效，不選。）
- 🔍 核實能否在 aarch64 Linux 上跑；不支持就在 x86 / Docker 上跑。
- 跑通 `q`，試 `til 10`、`\t`、`\\`（退出）。
- 裝 VS Code 的 KX 擴展 🔍 或直接用 REPL。

**完成標準**：`q` 啟動，`2*3+4` 返回 `14`，並且能說出為甚麼。

## Phase 1 · q 語言基礎（2–3 晚）

- atom、list、dict、table 互相轉換；`flip` 一個 dict 得到 table。
- 從右往左求值；`%` 是除法；`/` 是註釋。
- 函數 `{x+y}` 和 `{[a;b] a+b}`；副詞 `'` `/` `\` `/:` `\:`。
- 練習：不用循環寫 ① 累計和 ② 移動平均（`mavg`）③ 斐波那契前 20 項（`\` scan）。

**完成標準**：10 道小練習全用向量寫完，沒有一個 `do` / `while`。
**決策記錄**：`01-q-vs-java-mental-model`——三個 Java 寫法在 q 裏變成甚麼，哪一個最違反直覺。

## Phase 2 · 表與 qSQL（2 晚）

- 用 `?`（隨機）生成 100 萬行假成交：`time sym px size`。
- `select` / `exec` / `update` / `delete`；`by` 分組；`xbar` 做 1 分鐘 K 線；`wavg` 算 VWAP。
- 對比 `select` 和 `exec` 的返回類型。

**完成標準**：一句 qSQL 出每個 sym 的 1 分鐘 OHLCV + VWAP。
**證據**：`02-qsql-timing`——`\ts` 測 100 萬 / 1000 萬行的耗時和內存。

## Phase 3 · Joins，重點 `aj`（2 晚）

- 生成假報價表 `quote: time sym bid ask`。
- `lj` 接一張 sym 參考表（keyed table）。
- `aj[`sym`time;trade;quote]`：每筆成交配當時有效的報價；算滑點 `px - (bid+ask)%2`。
- `wj`：每筆成交前 5 秒內的平均價差。
- 給 quote 的 sym 列加 `` `g# ``、time 列確認 `` `s# ``，再測一次 `aj`。

**完成標準**：能在白板上畫出 `aj` 是怎樣配對的，並解釋為甚麼普通 SQL 寫它很痛苦。
**證據**：`03-aj-attributes`——有沒有屬性時的耗時對比。

## Phase 4 · 寫盤：splayed、partitioned、HDB（2 晚）

- 把 Phase 2 的表 splay 到磁盤（`.Q.en` 枚舉 sym）。
- 造 5 天數據，按日期分區寫盤（`.Q.dpft`），`\l db` 載入。
- 故意把 `where sym=...` 放在 `date=...` 前面，量一次全庫掃描，再改回來。

**完成標準**：能 `ls` 出目錄結構並逐個文件解釋。
**決策記錄**：`04-partition-scheme`——按日期分區、`p#` 加在 sym 上的理由；甚麼情況改按月分區。

## Phase 5 · IPC：多進程（1–2 晚）

- 開兩個 q 進程（`-p 5010`），用 `hopen` 互調；同步 vs 異步（`neg h`）。
- 寫 `.z.pg` / `.z.ps` 回調，打印收到的每條消息。
- 用 `.z.pw` 做一個最簡單的用戶名密碼校驗。

**完成標準**：說得出同步調用會阻塞誰、異步消息在哪裏排隊。

## Phase 6 · kdb+tick：TP + RDB + HDB（3 晚）

- 參考 `KxSystems/kdb-tick` 🔍，寫 `sym.q` schema（trade、quote 兩張表）。
- 啟動 TP、RDB、HDB；寫一個 q 腳本當假 feed，每秒推數據。
- 手動觸發 EOD（`.u.end`），確認 RDB 數據落成 HDB 的新分區。

**完成標準**：四個進程跑通，畫得出數據流，每個箭頭說得出是同步還是異步。

## Phase 7 · Java Feedhandler 接真實行情（3 晚）——**停止線**

- Java 17+，Maven，依賴 `javakdb`（KX 官方 Java 接口，`c.java`）🔍 Maven 坐標待核實。
- 訂閱 Binance 公開 websocket：`btcusdt@trade`、`btcusdt@bookTicker`
  （`wss://stream.binance.com:9443/ws/...`）🔍。所在地區被封時改用 Coinbase 或 OKX 的公開 feed。
- JSON → q 的表結構（列式批量，不是逐行發）→ `.u.upd` 異步推給 TP。
- 斷線重連、批量大小、背壓：TP 慢了 Java 端怎麼辦。
- **故障演練**：盤中 `kill -9` RDB，重啟後用 `-11!` 重放 tp log 恢復；量恢復時間。

**完成標準**：跑 24 小時無人值守；故障演練寫成一份 postmortem 式的文檔。
**決策記錄**：`07-feedhandler-batching`——逐條發 vs 批量發、批量大小怎麼定、TP 在哪一點成為瓶頸。
**證據**：`07-recovery-drill`。

---

以下 Phase 是延伸。

## Phase 8 · RTE：實時引擎（2 晚）

- 訂閱 TP，盤中實時算每分鐘 VWAP 和買賣價差；價差超閾值就告警。

## Phase 9 · Gateway + Spring Boot 查詢 API（3 晚）

- q gateway：按日期把查詢拆給 RDB 和 HDB，合併結果。
- Spring Boot 服務用 `javakdb` 連 gateway，暴露 `GET /bars?sym=&from=&to=`。
- 連接池、超時、查詢權限：哪些在 gateway 做，哪些在 Java 做。

## Phase 10 · PyKX 與 TCA 報告（2 晚）

- PyKX 拉 HDB 數據，用 pandas 出一份簡單 TCA：每小時平均滑點、價差分佈。
- 對比：q 裏做和 Python 裏做，各自順手的地方。
