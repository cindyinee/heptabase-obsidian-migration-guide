# Heptabase → Obsidian Migration Guide

Public Edition `1.0.1-beta` · 更新於 2026-09-25 · [CC BY 4.0](LICENSE) · 作者 Cindy Young（[IdeaMeka](https://ideameka.com)）

> **Public Edition beta。** 本指南整理自實際遷移與修復經驗，目前是 beta 版。它不是一鍵搬家工具，也不應直接套用到未盤點的資料。遇到規格沒涵蓋的情況，歡迎回報問題。

## 這份指南要解決什麼

把 Heptabase 搬到 Obsidian，不只是把 Markdown 複製到另一個資料夾。真正困難的是同時保留：

- 卡片正文與 metadata；
- Whiteboard 的空間配置與巢狀關係；
- MindMap 的文字、階層與 edges；
- Journal 的唯一正文與多個白板 instances；
- 圖片、附件、highlight、transcript 與 references；
- 已經在 Obsidian 裡新增或修改的使用者內容。

這份 Public Edition 提供一套可檢查、可回復、可以分階段驗證的遷移方法。它是寫給 AI Agent 執行、也讓人可以檢查的規格，不是一鍵 converter。案例數字只代表特定來源快照，不是其他 Vault 的固定目標。

## 先判斷你是哪一種遷移

| 情境 | 正確做法 |
| --- | --- |
| Fresh Vault | 在隔離的 test vault 重建並驗證，凍結成唯讀 canonical migration source，再從副本建立日常使用的 Active Vault。 |
| Existing Active Vault | 先辨識舊 migration layer、使用者新增內容、使用者修改內容與 Active-preferred files，再選擇 replace、import 或 semantic merge。不可整包覆蓋。 |

**Canonical Migration Source** 是已完成結構驗證、保持唯讀的重建來源；**Active Vault** 才是持續使用的 production vault。原始 Heptabase export、canonical source 與 Active Vault 是三個不同角色，都需要各自保存。

## 四個不可妥協的原則

1. **先盤點，再轉換。** 不先假設 export schema，也不以舊版本欄位名稱取代實際檢查。
2. **Low value 不等於刪除。** 不轉成正式筆記的資料仍需保留原始來源與 audit evidence。
3. **路徑存在不等於遷移成功。** Reference resolved、embed rendered、結構關係與實際可用性必須分開驗證。
4. **先忠實重建，再整理可用性。** 不要在還沒建立可回復基準前，就清理 ID、改 taxonomy 或覆蓋舊內容。

## 交給 AI Agent：第一段指令

使用者通常會貼上 README 的第一段指令（中文）。它要求的五項輸出，對應到本指南是：

1. Source inventory（有哪些資料、各有多少）
2. Migration Decision Table（建議轉換、封存或跳過，以及原因）
3. 需要使用者決定的項目
4. 主要風險
5. POC 建議（先試哪一張白板，以及為什麼）

第一輪只分析。「不要修改任何檔案」指的是**不修改 Heptabase export 與 test vault**；Agent 可以在工作資料夾裡另建 `_migration_work/`，放 scripts、hash manifest 與報告：

```text
我的搬家資料夾/
  Heptabase-Data-Backup-…/   ← 原始匯出，唯讀
  Obsidian_Test_Vault/        ← 測試 Vault，POC 通過前不寫入
  _migration_work/            ← Agent 的 scripts、manifests、報告
```

使用者多半不寫程式。回報時用使用者的語言，先講結論和需要他決定的事；技術細節放在 `_migration_work/` 的報告裡，並告訴使用者報告在哪裡。

### 選 POC 白板

POC 應挑一張**中等大小**（大約 30–80 個物件）、元素齊全的白板：有 Card、圖片、文字、Connection、MindMap，最好還有 nested Whiteboard。不要挑最大的白板，太大會讓 POC 難以逐項檢查。如果 Journal 或 PDF 不在這張白板上，另外各做一個小 POC。

確認 Inventory 與 POC 後才進入批次轉換。

### POC 的最低驗證

- [ ] Canvas 是合法 JSON，node IDs 唯一，edge endpoints 都存在。
- [ ] 每個 source instance 都對應到一個 Canvas node，數量對帳；例外逐一列出。
- [ ] 卡片正文與來源 ProseMirror 文字比對，沒有遺失。
- [ ] 圖片檔與來源檔內容相同（hash），不是只有路徑存在。
- [ ] Nested whiteboard 在 parent Canvas 裡有對應的 file node。
- [ ] MindMap 的節點數、階層與來源一致。
- [ ] 請使用者在 Obsidian 實際打開：左下角「開啟其他 Vault」→「開啟資料夾作為 Vault」→ 選 test vault，再打開 POC 的 `.canvas`。

## 第一步：凍結來源並建立 Inventory

至少保存以下資料：

- 完整 Heptabase export 與 `All-Data.json`；
- Markdown、assets 及其他附件；
- 來源 snapshot ID、匯出時間與工具版本；
- 每個檔案的相對路徑、大小與 SHA-256；
- 各 data type 的數量、active／deleted 狀態與抽樣結果；
- `All-Data.json` 裡的 `VERSION` 與 `DB_SCHEMA_VERSION`（本指南的實測欄位來自 Heptabase `1.103.0`、DB schema `133`）。

來源應設為唯讀或使用不可變快照。Hash manifest 放在來源目錄之外（例如 `_migration_work/`），避免 manifest 自己改變計算範圍。

**Windows 先檢查長路徑。** Heptabase export 可能有超過 260 字元的完整路徑（實測 6 個）。第一次掃描前就要處理，不要等到程式當掉：Python 讀取時使用 `\\?\` 前綴，或把工作資料夾放在較短的路徑。不要替使用者修改系統設定（例如 LongPathsEnabled）；需要時說明原因，請使用者自己決定。

### Markdown export 不是完整的語意來源

`All-Data.json` 是必要來源，不是選配。**正文以 `All-Data.json` 的 ProseMirror `content` 為準；Markdown export 主要用來取得 binary（圖片、PDF、影音）與交叉比對。**

| 資料來源 | 主要用途 |
| --- | --- |
| `All-Data.json` | 正文（ProseMirror）、object type、ID、relation、placement、`fileId`、semantic structure |
| Markdown／native export | 圖片與附件的 binary、交叉比對、使用者可直接閱讀的檔案 |

不要把 Markdown export 當成正文來源，原因是它很難可靠地對回 source ID：

- `.md` 檔沒有 ID，檔名由標題轉換而來，部分特殊字元會被替換。
- 沒有標題的卡片會被命名為 `A wonderful new card N`（實測 569 個）。
- 同名卡片會加數字後綴；部分 `.md` 的內容和 `All-Data.json` 不一致。
- 垃圾桶裡的卡片（`isTrashed: true`）也會被匯出（實測 3,312 個 `.md`，其中約 830 張在垃圾桶）。
- `Card Library/` 的 `.md` 放在同一層，但圖片放在各自的 `-assets` 子資料夾（實測 539 個）。

Native export 也可能有 `Mindmap/`（只有階層大綱，沒有座標與節點 ID）和 `Text Element/`。它們可以拿來核對，但 MindMap 結構、nested whiteboard 的 parent → child 關係、`textElements` 與 `mediaElements` 的座標，只有 `All-Data.json` 有。只靠 Markdown export，很容易得到一個「看起來差不多」但少了結構的 Vault。

不要假設 Whiteboard elements 一定巢狀在 `whiteBoardList`。實測 export 中，`cardInstances`、`textElements`、`mediaElements`、`highlightElements`、`journalInstances`、`sections`、`connections` 與 `mindMapInstances` 可能是頂層陣列，再用 `whiteboardId` 關聯 Whiteboard。缺 key、型別不符、孤立引用與重複 ID 都應明確報告，不能默默變成空陣列。

### 實測的 schema（Heptabase 1.103.0／DB schema 133）

以下是一份實際匯出的欄位，作為起點，不是保證。**欄位會隨版本改變，寫 join 邏輯前一定要先實際檢查。**

| 陣列 | 重點欄位 | 關聯 |
| --- | --- | --- |
| `cardList` | `title`、`content`（ProseMirror）、`isTrashed` | Card 本體 |
| `whiteBoardList` | `name`、`isTrashed` | Whiteboard 本體 |
| `cardInstances` | `cardId`、`whiteboardId`、`x`、`y`、`width`、`height`、`isFolded`、`foldedHeight` | Card 放在白板上的位置 |
| `mediaCards`／`mediaCardInstances` | `fileId`、`type`、`transcript`／`cardId`、`whiteboardId` | Media card 與它的位置 |
| `pdfCards`／`pdfCardInstances` | `fileId`、`title`／`pdfCardId`、`whiteboardId` | PDF card 與它的位置 |
| `textElements` | `content`、`whiteboardId`、座標 | 白板上的文字，直接帶位置 |
| `mediaElements` | `fileId`、`whiteboardId`、座標 | 白板上的圖片，直接帶位置 |
| `sections`／`sectionObjectRelations` | `title`、座標／`sectionId`、`objectId` | 分區與分區裡的物件 |
| `connections` | `beginId`、`beginObjectType`、`endId`、`endObjectType`、`beginStyle`、`endStyle`、`description` | 連線 |
| `whiteboardInstances` | `whiteboardId`（child）、`containerId`（parent）、`containerType`、`isChild`、座標 | Nested whiteboard |
| `mindMaps`／`mindMapInstances` | `layout`／`mindMapId`、`whiteboardId`、座標、`boundingBoxRelativeX/Y` | MindMap 與它的位置 |
| `mindMapNodes` | `mindMapId`、`parentId`、`childNodeIds`、`side`、`isCollapsed` | MindMap 階層 |
| `mindMapTextNodes`／`mindMapCardNodes` | `content`／`cardId` | 節點內容；ID 與 `mindMapNodes` 共用 |
| `journalList`／`journalInstances` | `date`、`content`／`journalDate`、`whiteboardId` | Journal 與它的位置 |
| `files` | `id`（即 `fileId`）、`name`、`size`、`type` | 附件的 metadata |

幾個容易出錯的地方：

- **連線端點的型別名稱和陣列名稱不一樣。** 實測 `beginObjectType`／`endObjectType` 的值有 `cardInstance`、`textElement`、`imageElement`（對應 `mediaElements`）、`mindMapTextNode`、`mindMapCardNode`、`highlightElementInstance`、`pdfCardInstance`、`section`。先列出所有出現過的值，再逐一對應。
- **有些 ID 是刻意共用的。** `mindMapNodes` 和 `mindMapTextNodes`／`mindMapCardNodes` 用同一個 node ID，`actionItems` 和 `actionItemAdds` 也是。檢查重複 ID 要在同一張表內做，不要跨表報告。
- **Card 被丟進垃圾桶，instance 不一定跟著消失。** 使用中的白板可能還有指向已刪除或不存在卡片的 instance，要列出來問使用者。

## 第二步：建立 Migration Decision Table

每一種 data type 都要回答：誰建立、是否刻意維護、是否有使用證據、關係價值、內容是否唯一、轉換品質，以及資料量與維護成本。

| Value／來源狀態 | 預設行為 |
| --- | --- |
| High value | Convert：轉換並保留內容、關係與 metadata。 |
| Medium／Ambiguous | Ask：先提出映射方式、風險與建議，只詢問真正需要決定的項目。 |
| Low value | Archive only：保留來源，不主動變成正式 Obsidian 內容。 |
| Source deleted／system artifact | Skip active migration：不恢復成正式內容，但保留排除原因與 audit evidence。 |

決策表至少包含 `Data type / Count / Value / Recommended action / Reason`。Count 必須標明範圍與單位，區分 unique content、instances、relationships 和 active／deleted；未知值寫「待盤點」，不要填成 0。

決策表要涵蓋 `All-Data.json` 裡**每一個非空的陣列**，不只 Card 和 Whiteboard。容易被漏掉的有：sections、connections、PDF cards、media cards、`sources`（例如 Readwise 匯入）、`insights`（Heptabase AI 自動產生的洞察，不是使用者寫的）、`chats`、`templates`、`collections`、`tabs`、`actionItems`。使用者價值不明的系統資料，預設 Archive only。

Journal 也要分開盤點：instance 可能指向沒有正文的日期，也可能有空白日記或匯出日期之後的日期。這些都列成例外，不要默默略過。

## 第三步：定義目標 Vault 邊界

建議讓資料夾主要表達來源、系統或 workflow，而不是讓 AI 自動推測內容分類：

```text
Active Vault/
  heptabase/       # 已核准的 Heptabase migration layer
  daily/           # Journal / Daily Notes
  _local/          # Obsidian-native 使用者內容
  _meta/           # 必須留在 production 的 metadata preservation artifacts
  Readwise/
  ai-workspace/
  .obsidian/
```

`heptabase/` 內的子資料夾（例如 `cards/`、`sources/`、`media/`）是遷移時的設計選擇，不是 Heptabase 的原始結構；Heptabase export 的 Card Library 把 `.md` 放在同一層，附件放在各自的 `-assets` 子資料夾。分流規則要寫下來，才能重現。

重建用的 manifests、來源快照與 audit artifacts 應留在 migration workspace，不要因歷史範例而自動塞進 production Vault。每個 Whiteboard 可以保留自己的 Markdown、Canvas 與 assets 子資料夾，但移動任何檔案後，必須更新整個 Vault 的 references，而不只更新同一個白板或只更新 Canvas。

## 第四步：決定每種物件的 Obsidian 表示方式

先用 `All-Data.json` 確認 source object type、source ID 與 instance relationship，再決定 representation。不要只根據 Canvas 上看起來像文字或圖片來判斷。

| Heptabase object | 建議的 Obsidian representation |
| --- | --- |
| Card | Markdown note；Canvas 使用 file node 引用。 |
| 沒有標題的 Card | 讀取 content JSON 的 `type` 判斷；image Card 轉成含 embed 的 Markdown note 與 Canvas file node。不可因為 title 空白就略過或變成空白。 |
| Whiteboard | `.canvas`。 |
| Nested whiteboard | Parent Canvas 中指向 child Canvas 的 file node，保留 placement 與 geometry。 |
| MindMap | Canvas text／file nodes 加 edges，保留文字、階層與關係。 |
| URL-only 或一般 `textElement` | Canvas text node，保留可讀、可點擊的 URL。 |
| 含圖片或媒體的 rich `textElement` | Markdown file node，保留內容與 embed order。 |
| Journal | 正文只存一份 `daily/YYYY-MM-DD.md`；各白板 instance 用 Canvas file node 引用。 |
| Highlight | 將 ProseMirror 文字與註解轉成可讀 Markdown；真正空白的才移除。 |
| Media card | Binary 可取得時保留媒體與 metadata；無法取得時留下可讀的缺失說明與 audit evidence。 |
| Tag／Property | Frontmatter；不要批次覆寫使用者已有的 metadata。 |

### 沒有標題，不等於沒有內容

Card 的 `title` 可以是空的，但 `content` 仍可能是一張完整的圖片：`{"type":"image","attrs":{"fileId":"..."}}`。以 title 是否存在來決定要不要轉換，會靜默遺失內容。實測中，17 個原本被轉成 `*(empty card)*` 的 Canvas nodes，回查後全部是可以復原的 image Card；另有 3 張來源真的沒有內容，才保留 empty representation。

注意「沒有標題的 image Card」和 `mediaCards` 是兩種不同的東西：前者是 `cardList` 裡 title 空白、`content` 是圖片的卡片，圖片通常在 export 裡找得到；後者是獨立的 media card，binary 常常不在 export 裡（見「Placeholder 是警訊」）。

### 檔名的預設規則

第一次建立 Obsidian 檔名，和事後 rename 是兩件事。下面是建議預設，POC 時先給使用者看，同意後再批次套用：

- **有標題**：用標題當檔名。
- **沒有標題**：依內容命名，例如取正文第一行的前 30 個字；圖片卡用「未命名圖片卡」加 6 碼 source ID。不要用 UUID 當主要檔名。
- **Windows 與 Obsidian 不允許的字元**（`\ / : * ? " < > |`），以及會干擾連結的 `# ^ [ ]`：換成全形字元或移除。
- **同名碰撞**：加 6 碼 source ID 後綴，不要覆寫。
- **全部寫進 mapping log**：source ID、原標題、最後的檔名、套用的規則。

### Canvas 的最低資料

```json
{
  "id": "stable-node-id",
  "type": "file|text|group",
  "x": 0,
  "y": 0,
  "width": 420,
  "height": 240,
  "file": "vault/relative/path.md",
  "text": "Markdown text"
}
```

JSON Canvas 並不要求 Tab 縮排。舊案例曾把 Canvas 開啟失敗歸因於 space indentation，但這不是通用規格。真正應驗證的是合法 JSON、欄位型別、唯一 node IDs、有效 edge endpoints、有限座標與正確的 Vault-relative paths。

幾個要事先決定、並寫進報告的對應：

- **摺疊的卡片**（`isFolded: true`）：Canvas 沒有摺疊狀態。選擇用完整 `height`（內容完整可見）或 `foldedHeight`（外觀接近原本），全部一致。
- **箭頭樣式**：`connections` 的 `beginStyle`／`endStyle` 對應 Canvas edge 的 `fromEnd`／`toEnd`（`arrow` 或 `none`）。
- **連線文字**：`description` 轉成 edge 的 `label`。

### MindMap 與 nested whiteboards

MindMap 不能因為目標格式不同就直接 skip。實測的來源結構（欄位名稱仍要先實際檢查）：

- `mindMaps`：MindMap 本體；`mindMapInstances`：它放在哪張白板、外框座標。
- `mindMapNodes`：階層，用 `parentId` 與 `childNodeIds` 表示；沒有獨立的 edges 陣列，edges 要從 parent → child 推出來。
- `mindMapTextNodes`／`mindMapCardNodes`：節點的文字（ProseMirror）或指向的 Card，ID 與 `mindMapNodes` 相同。

每個 instance 使用穩定 ID（instance ID + node ID）轉成 Canvas nodes 與 edges，edges 保留 `fromNode`／`toNode`／`fromSide`／`toSide`。節點本身沒有座標，用 instance 的 `x`／`y`／`width`／`height` 當外框，內部採可重現的樹狀 layout。`boundingBoxRelativeX/Y` 的確切意義尚未確認，不要單憑推測換算。

**事先告訴使用者：MindMap 搬過去後會重新排版，外觀和 Heptabase 不同，但節點、文字與階層應該一致。** 否則使用者很可能以為搬壞了。

Nested whiteboard 的關係記在 `whiteboardInstances`：`whiteboardId` 是 child，`containerId` 是 parent，`containerType` 為 `whiteboard` 時才是白板裡的白板（實測另有 `map`，是放在總覽地圖上的位置）。實測中同一張 child 可能同時放在兩張以上的 parent 裡，也可能有 `containerId` 指向不存在的白板，兩者都要列出來。

Nested whiteboard 的 fidelity 不只包括 child Canvas 檔案存在，還包括 parent → child relationship、child 在 parent 中的位置、child 自身內容與最終 reference。把所有 Canvas 攤平成互不相干的檔案，仍然是結構遺失；而且這種遺失不會產生任何 broken reference。

### Journal

同一天的 Journal 正文只保存一份，多個白板 instances 都指向同一份 daily note。Manifest 應記錄 instance ID、日期、daily path、whiteboard ID、geometry 與 folded state。Unique dates、正文檔案數和 instances 數量要分開對帳。

## 第五步：用可回復的 Pipeline 執行

1. Inspect export schema、來源狀態與 data types。
2. 建立 Inventory 與 Migration Decision Table。
3. 在隔離 test vault 重建內容、關係與空間結構。
4. 驗證結構、references、metadata、assets 與代表性案例。
5. 凍結通過結構驗證的 canonical source。
6. 從副本建立 production candidate，才開始 usability cleanup。
7. 若已有 Active Vault，先 audit user-created、user-modified 與 Active-preferred files。
8. 建立完整可還原 backup 與 replace／import／merge mapping。
9. 只取代已確認未修改的 migration layer；先保護使用者內容。
10. 對 user-modified notes 做 semantic merge，不用新版直接覆蓋。
11. 更新 Canvas、Markdown links、wikilinks、embeds 與 attachment references。任何 move／rename／資料夾重組之後都要重做這一步，範圍是整個 Vault 的筆記，不只 Canvas。
12. 執行分層程式驗證與 Obsidian UI 抽查。
13. 比較 canonical source 與 Active Vault 的相對路徑差集。
14. 只有證據充分的 residuals 才移到 Vault 外 quarantine。
15. 再跑一次完整驗證，保存 counts、hashes、例外與 quarantine report。

## 常見技術陷阱

### ProseMirror 不是純文字

Heptabase 富文本使用 ProseMirror JSON。必須遞迴處理；空 paragraph 不一定有 `content`，不要用固定深度索引。

不只要取文字，還要把每一種 node 轉成對應的 Markdown。先列出資料裡實際出現過的所有 node 與 mark 類型，再逐一決定轉法，未處理的類型要報告，不要默默丟掉。實測出現過：

- **Nodes**：`paragraph`、`heading`、`bullet_list_item`、`numbered_list_item`、`todo_list_item`、`toggle_list_item`、`blockquote`、`code_block`、`horizontal_rule`、`table`／`table_row`／`table_cell`／`table_header`、`image`、`video`、`audio`、`file`、`math_inline`／`math_display`、`date`、`mention`、`card`、`whiteboard`、`pdf_card`、`image_card`、`video_card`、`highlight_element`、`section`、`chat`、`embed`、`hard_break`。
- **Marks**：`strong`、`em`、`underline`、`strike`、`code`、`link`、`highlight`、`color`、`anchor`。

兩個特別要處理的情況：

- **內部連結寫成網址。** 指向其他卡片的連結可能是 `https://app.heptabase.com/<space>/card/<id>`（實測 553 個），要依 source ID 轉成 Obsidian 的 wikilink。
- **圖片直接內嵌在正文裡。** 部分圖片以 base64 `data:image/...` 存在 `content` 中（實測 16 個），要取出成獨立檔案再 embed。

Connection 的 `description` 也是 ProseMirror，轉成 Canvas edge 的 `label` 文字。

### 欄位名稱不能用猜的

各種 instance 指向來源物件的欄位名稱並不一致。實測中，`mediaCardInstances` 指向 media card 的欄位是 `cardId`，不是 `mediaCardId`；`textElements` 直接帶 `whiteboardId`，不經過 instance table。每種 instance 都要先實際檢查欄位，再寫 join 邏輯。

### Placeholder 是警訊，不是終點

`*(empty card)*` 這類 placeholder 或 fallback，不代表轉換成功，而是在問：為什麼這個 source object 最後只能變成 placeholder？每個 placeholder 都要回查來源，分成「可以復原」與「來源真的沒有內容」。

同樣地，缺圖要分開兩種情況：Migration 漏掉的，與來源 export 本來就沒有 binary 的。若 source object、placement 與 `fileId` 都在，但 export 裡確實沒有檔案，標記為 `source-unrecoverable` 並保留 object 與位置，不要讓 Agent 反覆重試不存在的檔案。

獨立的 `mediaCards` 特別要先盤點比例。實測中，146 張圖片類 media card，以 `files` 表的大小比對，在 export 裡一張都找不到對應檔案，影片與音訊類也一樣；另有 78 張沒有 `files` 紀錄。遇到這種情況要在 Inventory 階段就告訴使用者，並建議他：重要的幾張可以回 Heptabase 手動另存。

### 媒體檔名可能碰撞

大量來源檔可能都叫 `image.png`，export 後才被加上數字後綴。export 的檔名裡也沒有 `fileId`。

建議的對應方式：

1. 從 `files` 表用 `fileId` 取得 `name` 與 `size`。
2. 在 export 裡找 size 相同的檔案當候選；再用檔名、所在的 `-assets` 資料夾與卡片標題縮小範圍。
3. 只剩一個候選時才算對上。多個候選時，比較內容 hash：內容完全相同就可以共用；內容不同就列為 ambiguous，不能直接選第一個。
4. 找不到候選時，標記 `source-unrecoverable`。

### 重複的 Canvas

多輪遷移容易留下重複或過期的 Canvas。判斷方法是比對 Canvas node IDs 與 `All-Data.json` 的 source instance IDs：完全對不上的，是其他輪次留下的產物，應在備份後移除。不要用「檔名帶數字後綴」當作刪除條件。

### 路徑正規化與重新命名要分開

Canvas 內使用 `/` 與 Vault-relative paths。先以 reference syntax 解決 encoding 問題，rename 是最後手段。所有 move／rename 都保存 mapping log，至少包含來源 ID、old/new path、原因、規則版本、碰撞處理、受影響 references 與驗證結果。

Windows 長路徑支援取決於系統、API 與應用程式。`\\?\` 可以協助某些工具讀取來源，但不能寫進 Canvas 或 Markdown references。縮短路徑時保留可辨識前綴、副檔名與穩定 hash，並檢查碰撞。

### Markdown 圖片路徑含 `)`

實測 image embed 路徑含 `)` 時，angle-bracket destination 仍可能無法正常 render。可嘗試將 `)` 編碼為 `%29`、空格編碼為 `%20`，但最後仍需在 Obsidian 實際確認圖片顯示。Reference resolved 不等於 embed rendered。

### PowerShell 的方括號

PowerShell 會把 `[]` 當 wildcard。驗證真實路徑時使用：

```powershell
Test-Path -LiteralPath $path
```

否則可能把存在的檔案誤報為 broken reference。

### Block IDs 與 transcripts

不要把每個來源 UUID 都變成使用者可見的 `^UUID`。先建立 definitions 與 references 的雙向索引，只保留確實被 block link 或結構依賴使用的 anchors。實測中，保留全部 anchors 會產生超過六萬個可見 ID，但真正被引用的只有幾百個。

Transcript 應依來源順序轉成可讀、含 timestamp 的 Markdown；failed、processing 或 unavailable 狀態顯示清楚的 placeholder。Raw JSON 與 technical IDs 留在 audit layer，不進 normal reading content。

## Existing Active Vault：不能直接覆蓋

先將檔案分類：

| 分類 | 處理原則 |
| --- | --- |
| old-migration-duplicate | 確認未被使用者修改後才可由新版取代。 |
| old-only-user-created | 保留；必要時搬到 `_local/` 並更新 references。 |
| same-file-identical | 保留，不需要重寫。 |
| same-file-user-modified | 使用 baseline、Active 與新 reconstruction 做三方 semantic merge。 |
| attachments-old-only | 保留並檢查 Active references。 |
| `.obsidian` 與 templates | 保留 Active 設定，不可整包覆蓋。 |
| Active-preferred | 明確記錄理由與 hash，保留 Active version。 |

Semantic merge 要保留使用者新增、修訂與刪除意圖，同時整合新版恢復的來源內容、frontmatter、附件和 links。缺 baseline 或無法可靠判斷時，保留雙方版本與差異供人工決定，不猜測使用者意圖。

較好的新版結構，應以 **selective backport** 回植已確認的改進（例如 nested whiteboards、MindMap 結構、漏掉的 image Cards），同時保留 Active Vault 既有的資料夾慣例、Journal 用法與使用者內容。遷移完成後，Active Vault 就是使用者的工作區；之後的每次修正都是 reconciliation，不是 overwrite。

Cleanup 優先移到 Vault 外 quarantine，不直接永久刪除。Quarantine report 應保存原路徑、新路徑、原因、來源判斷、hash、時間與 reference 影響。

## 四層驗證

| 層次 | 要回答的問題 |
| --- | --- |
| 內容完整 | 原始內容有沒有留下？ |
| 關係完整 | Card、Whiteboard、MindMap 之間的關係有沒有留下？ |
| 空間與結構 | 位置、階層與 nesting 是否合理保留？ |
| 實際可用 | 在 Obsidian 裡是不是真的能讀、能開、能點？ |

程式 audit 與實際 UI 使用，是兩個不同的驗收層級，要分開記錄。

### 內容完整

- [ ] Source snapshot、範圍、counts、paths 和 hashes 已保存。
- [ ] 每個 data type 都有來源數量、輸出數量與例外分類。
- [ ] Canonical source 在 import／merge 前後保持不變。
- [ ] 每個 placeholder 都已回查來源，並分類為「已復原」或「來源確實沒有內容」。
- [ ] Recoverable media 已恢復；unrecoverable 項目有來源證據且未被捏造或靜默刪除。
- [ ] User-created、user-modified 與 Active-preferred files 都有明確處理紀錄。
- [ ] Tags、Properties 與其他使用者 metadata 沒有被批次覆寫。

### 關係完整

- [ ] 所有 Canvas file nodes 指向 Vault 內存在的 mapped target。
- [ ] Markdown links、wikilinks、images、attachments 與 block links 分別驗證。
- [ ] 任何 rename／move／資料夾重組之後，**全部筆記**內的 references 都已依 mapping 更新，不只 Canvas。
- [ ] MindMap 的 node text、edges、hierarchy 與 relationships 已對帳。
- [ ] Journal instances 指向正確且唯一的 daily notes。

### 空間與結構

- [ ] 所有 Canvas 都是合法 JSON，node IDs 唯一，edges endpoints 有效。
- [ ] Nested whiteboard containment、placement 與 child contents 已對帳。
- [ ] Canvas node 的 type 與來源 object type 一致（圖片不是 text placeholder）。
- [ ] 重複或過期的 Canvas 已用 source ID 比對並處理。

### 實際可用

- [ ] 圖片與附件不只路徑存在，也能在 Obsidian 正常 render／開啟。
- [ ] 筆記正文沒有大量不必要的 UUID anchors 或 raw JSON。
- [ ] 抽查代表性白板：在 Obsidian 實際點擊卡片、連結與嵌入內容。

### 收尾

- [ ] Invalid Canvas JSON = 0。
- [ ] Canvas broken file refs = 0。
- [ ] 筆記內 Markdown links／wikilinks 的 broken 數量 = 0，或每一個都已分類說明。
- [ ] Unresolved recoverable items = 0。
- [ ] 已知無法恢復的項目逐筆分類並保留 evidence。
- [ ] Active-only residuals 全部分類；沒有未判定就刪除的檔案。

**Zero broken references 並不能證明 structural fidelity。** 如果一個 relation 根本沒被建立，就不會出現 broken reference：child Canvas、MindMap edges、geometry、內容順序或 media rendering 遺失時，即使每個 path 都存在，遷移仍未完成。

反過來也要注意，「Canvas broken refs = 0」只涵蓋 Canvas。實測中，最終驗收時 Canvas 全數正常，但資料夾重組只更新了 Canvas 內的路徑，筆記裡仍有約 1,700 個連結指向已不存在的舊資料夾；補做全 Vault 的筆記連結掃描後才發現並修復。

## 完成的定義

一次可信的遷移至少要能回答：

- 哪個來源 snapshot 被處理？
- 每一種資料被轉換、封存或排除的理由是什麼？
- 哪些資訊被完整保留，哪些只能部分保留？
- 哪些項目無法恢復，證據在哪裡？
- 使用者在 Active Vault 的修改是否仍在？
- 如果結果不理想，是否能回復到遷移前狀態？

Migration success 不是「所有檔案都存在」，而是內容、關係、空間結構與可用性都被保存，所有例外也誠實可追溯。這份 Guide 真正要回答的問題是：**怎麼知道你真的搬到了你以為自己搬到的東西？**

## 變更紀錄

- **1.0.1-beta（2026-09-25）**：依一次盲測修訂（一個不知道先前過程的 AI Agent，只靠 README 與本指南完成 Inventory 與一張白板的 POC）。新增：工作資料夾與報告位置、POC 選法與最低驗證、實測 schema 表、正文以 `All-Data.json` 為準、檔名預設規則、`fileId` 對應方式、ProseMirror 類型清單、Windows 長路徑提前檢查。更正：MindMap 欄位名稱、native export 其實可能有 `Mindmap/` 與 `Text Element/`、Card Library 不是完全扁平、media card 缺檔的規模。
- **1.0.0-beta（2026-09-25）**：第一個公開版本。

## 版本、來源與授權

- Public Edition：`1.0.1-beta`（依盲測結果修訂；變更見文末）
- 技術來源：Heptabase → Obsidian Migration Guide v3.2 / field-tested specification（2026-09-21）
- 技術來源 SHA-256：`25F67AB318F56DD75F2FF24FB51AEFCF1E552CC6984CC54E7727ADD76E4B2DE8`
- Public Edition 更新日期：2026-09-25
- 授權：Creative Commons Attribution 4.0 International（CC BY 4.0）

你可以依 CC BY 4.0 分享與改作這份 Public Edition，但必須標示作者 Cindy Young、作品名稱、授權方式，並說明是否做過修改。案例中的個人資料、原始 Vault、第三方圖片與軟體本身不因本文件授權而自動改變其權利狀態。
