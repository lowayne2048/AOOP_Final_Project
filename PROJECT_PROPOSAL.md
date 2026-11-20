# 等廁所模擬器｜遊戲設計與開發計畫

> Toilet Waiting Simulator · AOOP 個人專題 · 企劃階段
>
> 技術：Python、Pygame、pytest、Git/GitHub｜人數：1 人｜預計開發時間：約 4 週

## 1. 遊戲概念

玩家突然有強烈便意，必須在進度條達到 **100%** 之前找到能使用的廁所隔間。看似簡單，但隔間可能被占用、故障、清潔中，門口也可能有人排隊。玩家可以繼續等待、換間廁所或去其他樓層；上下樓梯會讓便意增加得更快，因此遠處的空廁所不一定比較划算。

遊戲採 **2D 俯視角像素風**。玩家以 WASD 移動、E 鍵互動，在建築物裡探索。成功找到合法且可用的隔間即通關；便意達到上限則失敗。未來可在自由探索畫面之外，增加廁所或 NPC 的專屬互動畫面（A → C），但第一版先不實作。

### 有趣的地方

- **每次排隊都是一場賭注。** 某個隔間看起來快空了，裡面的人卻可能正在滑手機、抽菸或做其他私人活動，遲遲不出來。
- **換樓層有成本。** 玩家知道其他樓層可能更空，但移動需要時間，上樓梯還會加快便意累積。
- **NPC 也在搶有限的隔間資源。** NPC 會自主選擇廁所、排隊、離開或改去別處；玩家與 NPC 的選擇會互相影響。
- **NPC 並非全知。** 他們知道廁所的位置，卻必須親自接近才能知道當下的排隊狀況，離開後只能依據舊記憶決策。
- **環境會變動。** 清潔、故障、維修，以及誤闖不適當廁所而遭驅離，都可能打亂原本的計畫。

## 2. 核心遊戲機制

| 系統 | 遊戲中的功能 | 實作重點 |
|---|---|---|
| 地圖與移動 | 自由走動、碰撞、與廁所互動；未來跨樓層 | Pygame、Tile 地圖、場景切換 |
| 便意 | 隨時間上升；移動與樓梯影響增長；100% 失敗 | 統一的數值更新與邊界判定 |
| 隔間 | 空閒、使用中、故障、清潔中 | 隔間狀態轉換與占用互斥 |
| 排隊 | 玩家／NPC 入隊、離隊、輪流使用隔間 | FIFO 與資源分配，不可重複入隊 |
| NPC 行為 | 閒逛→找廁所→前往→觀察→排隊→使用→離開 | FSM（有限狀態機） |
| 感知與記憶 | 靠近廁所才能觀察隊伍，記住上次看到的資訊 | 觀察範圍、時間戳、舊資訊 |
| 決策與尋路 | NPC 依距離、排隊資訊與個性選擇目的地並走過去 | Strategy Pattern + A* |
| 事件 | 長時間占用、故障、清潔、驅離 | 可重現的事件觸發與狀態影響 |
| UI | 便意條、互動提示、隔間狀態、成功／失敗 | 畫面與核心邏輯分離 |

**NPC 決策例子：** NPC 到達一樓廁所，看到前面有四人排隊。他可以選擇等待，或憑記憶前往二樓。到了二樓才知道實際排隊人數；若情報過期，可能做出錯誤決定。為避免兩間廁所間無限往返，會設計換目標冷卻時間。

**三種技術各司其職：** FSM 決定 NPC 目前做什麼；Strategy Pattern 決定要去哪間廁所；A* 負責找到可通行的移動路徑。NPC 的觀察與記憶提供策略所需資訊。

## 3. 預計類別與主要方法

以下是**設計草案**，方法名稱與參數會隨實作修訂。不同模組透過明確介面合作，不把所有判斷塞進 `Game` 或 `NPC.update()`。

### 遊戲與玩家

| Class / Interface | 職責 | 預計 Methods |
|---|---|---|
| `Game` | 遊戲主迴圈、狀態更新與勝負 | `run()`、`update(dt)`、`check_result()` |
| `SceneManager` | 管理探索畫面與未來互動畫面 | `change_scene(scene)`、`update(dt)`、`render(screen)` |
| `ExplorationScene` | 地圖、角色與可見場景 | `handle_input(event)`、`update(dt)`、`render(screen)` |
| `Player` | 移動、碰撞、互動與隊伍操作 | `move(direction, dt)`、`interact(target)`、`join_queue(queue)`、`leave_queue()` |
| `UrgencySystem` | 便意增長與失敗判定 | `advance(dt, activity)`、`get_level()`、`is_failed()` |
| `InteractionController` | 判斷互動是否合法並執行 | `get_actions(target)`、`execute(action)` |

### 廁所與共享資源

| Class / Interface | 職責 | 預計 Methods |
|---|---|---|
| `Building` / `Floor` | 管理樓層、廁所與樓梯 | `get_floor(id)`、`get_restrooms()`、`get_stairs()` |
| `Restroom` | 管理所屬隔間與隊伍 | `get_available_stalls()`、`request_entry(actor)` |
| `Stall` | 隔間占用與狀態轉換 | `can_enter()`、`occupy(actor)`、`release()`、`set_out_of_order()`、`start_cleaning()` |
| `QueueManager` | 維持隊伍順序與分配資格 | `enqueue(actor)`、`leave(actor)`、`peek_next()`、`dequeue_next()` |
| `StallActivity` | NPC 在隔間內活動的占用時間規則 | `get_duration(npc, rng)`、`on_finish()` |

### NPC、資訊與事件

| Class / Interface | 職責 | 預計 Methods |
|---|---|---|
| `NPC` | 持有狀態、位置、目標、記憶及策略 | `update(dt)`、`observe_restroom(restroom)`、`follow_path()` |
| `NPCBehavior` | FSM 狀態轉換 | `decide_next_state(npc)`、`transition(new_state)` |
| `ObservationSystem` | 判斷能否看到廁所隊伍 | `can_observe(npc, restroom)`、`observe_restroom(npc, restroom)` |
| `RestroomMemory` | 保存曾觀察的隊伍與觀察時間 | `observe(id, queue_len, time)`、`get_last_observation(id)`、`is_stale(id, now)` |
| `RestroomSelectionStrategy` | NPC 選擇目的廁所的共通介面 | `choose(npc, known_restrooms, candidates)` |
| `NearestStrategy` | 優先選最近廁所 | `choose(...)` |
| `ExpectedWaitStrategy` | 參考已知排隊資訊與移動成本 | `choose(...)` |
| `ImpatientStrategy` | 等太久會改換目標 | `choose(...)` |
| `AStarPathfinder` | 在網格上找可走的路 | `find_path(start, goal, grid)`、`heuristic(a, b)` |
| `EventManager` | 排定、更新、結束事件 | `schedule(event)`、`update(dt)`、`resolve(event)` |
| `GameEvent` | 故障、清潔等事件的共通介面 | `apply(world)`、`is_active()` |

**OOP 設計原則：** 隔間與隊伍封裝自己的狀態；NPC 使用組合取得策略、記憶和行為；選廁所策略可以替換；場景介面與 Pygame 呈現分開，為日後 A+C 模式保留擴充空間。FSM 初期可先用 `Enum` 與轉換函式，不強制為每種狀態建立一個類別。

### 架構圖與流程圖

- [場景架構與未來互動擴充](docs/diagrams/01_scene_and_extension.png)
- [核心領域類別：玩家、隔間、隊伍](docs/diagrams/02_domain_classes.png)
- [NPC 決策、觀察、記憶與尋路](docs/diagrams/03_npc_decision_observation.png)
- [遊戲主要流程](docs/diagrams/04_gameplay_flow.png)
- [NPC 狀態流程](docs/diagrams/05_npc_state_flow.png)

## 4. 預計進度（約一個月）

| 時間 | 主要工作 | 可驗收成果 |
|---|---|---|
| 第 1 週 | 建立 Repo、Pygame 視窗、地圖與玩家移動；便意條、隔間狀態 | 能走到可用隔間通關，便意滿會失敗 |
| 第 2 週 | `QueueManager`、NPC 基本 FSM、隔間占用與釋放 | NPC 可以排隊、使用、離開；排隊與占用測試通過 |
| 第 3 週 | A*、`ObservationSystem`、`RestroomMemory`、至少一種選擇策略 | NPC 可移動至廁所，靠近後更新情報並作決策 |
| 第 4 週 | 模組整合、除錯、測試、畫面整理、README、展示 | 有一個穩定可玩的版本及可重現的測試紀錄 |

### 功能優先順序

- **必做（MVP）：** 一層地圖、一間廁所與三個隔間、便意系統、玩家移動、NPC 基本 FSM、排隊、基本 A*、成功／失敗判定。
- **優先擴充：** NPC 有限資訊觀察與記憶、兩種以上廁所選擇策略、第二間廁所（使策略差異有實際意義）。
- **有餘力再做：** 多樓層、清潔／故障事件、NPC 特殊占用活動、進入不適當廁所遭驅離，以及 C 模式的專屬互動畫面。

## 5. 測試與專案管理

課程允許 AI 協助開發，但程式正確性由開發者負責。將核心邏輯與 Pygame 畫面分離，用 `pytest` 驗證：

- 同一隔間不會同時被兩人占用；故障／清潔中無法使用。
- NPC 不會重複入隊；離隊與釋放隔間後的隊伍順序正確。
- FSM 不會在排隊中同時移動或使用另一隔間。
- A* 不會穿牆，無路可走時能回報失敗。
- NPC 不會讀取未曾觀察的遠方即時排隊資訊。
- 便意不超出設定範圍，勝敗判斷正確。

使用 GitHub Issues 記錄需求和 Bug，將各週目標設成 Milestones，重要功能分支開發並保留 commit／PR 與測試結果。每週至少保留一個可執行版本。若進度落後，優先完成穩定的 MVP，不為增加事件而犧牲正確性。

## 6. 技術與待決事項

- **技術：** Python、Pygame、pytest、Git/GitHub；地圖可使用 Tiled，像素素材可使用 LibreSprite／Aseprite。
- **尚未決定：** 正式地圖大小、NPC 數量、便意增長數值、事件頻率、美術素材與關卡配置。
- **未來方向：** 更多樓層與廁所、複雜 NPC 活動、不同建築地圖，以及從方案 A 擴充至 A+C 的互動模式。

> 本文件是設計與開發規劃，不代表上述進階功能已完成。
