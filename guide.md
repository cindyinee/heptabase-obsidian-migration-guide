# Heptabase → Obsidian Migration Guide

Public Edition `1.0.0-beta` · 更新於 2026-09-25 · [CC BY 4.0](LICENSE) · 作者 Cindy Young（[IdeaMeka](https://ideameka.com)）

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

把 Heptabase export、這份 Guide 與一個隔離的 test vault 放進同一個 workspace。第一輪只要求分析，不修改任何檔案：

```text
Read the migration guide first.

Do not modify any files yet.

Inspect the Heptabase export and target Obsidian environment.

Then provide:

1. Source inventory
2. Migration Decision Table
3. Ambiguous items that require my decision
4. Major migration risks
5. A representative POC plan

Treat the original Heptabase export as read-only.
Do not begin batch conversion until I approve the POC.
```

確認 Inventory 與 POC 後才進入批次轉換。POC 應挑一個同時含 Card、圖片、文字、Connection、MindMap 與 nested Whiteboard 的代表性白板，轉完後在 Obsidian 實際打開檢查。

## 第一步：凍結來源並建立 Inventory

至少保存以下資料：

- 完整 Heptabase export 與 `All-Data.json`；
- Markdown、assets 及其他附件；
- 來源 snapshot ID、匯出時間與工具版本；
- 每個檔案的相對路徑、大小與 SHA-256；
- 各 data type 的數量、active／deleted 狀態與抽樣結果。

來源應設為唯讀或使用不可變快照。Hash manifest 放在來源目錄之外，避免 manifest 自己改變計算範圍。

### Markdown export 不是完整的語意來源

`All-Data.json` 是必要來源，不是選配。Markdown export 適合取得已 render 的文字與附件，但它會遺失 object type、instance ID、白板歸屬、座標、邊線與巢狀關係。

| 資料來源 | 主要用途 |
| --- | --- |
| Markdown／native export | 已 render 的文字、附件、使用者可直接閱讀的檔案 |
| `All-Data.json` | object type、ID、relation、placement、`fileId`、semantic structure |

實測中，以下資料只存在或只完整存在於 `All-Data.json`：MindMap（native export 不匯出）、nested whiteboard 的 parent → child 關係、`textElements`（沒有對應的 card 檔案）、`mediaElements` 的座標，以及沒有標題的 image Card。只靠 Markdown export，很容易得到一個「看起來差不多」但少了結構的 Vault。

不要假設 Whiteboard elements 一定巢狀在 `whiteBoardList`。實測 export 中，`cardInstances`、`textElements`、`mediaElements`、`highlightElements`、`journalInstances`、`sections`、`connections` 與 `mindMapInstances` 可能是頂層陣列，再用 `whiteboardId` 關聯 Whiteboard。缺 key、型別不符、孤立引用與重複 ID 都應明確報告，不能默默變成空陣列。

## 第二步：建立 Migration Decision Table

每一種 data type 都要回答：誰建立、是否刻意維護、是否有使用證據、關係價值、內容是否唯一、轉換品質，以及資料量與維護成本。

| Value／來源狀態 | 預設行為 |
| --- | --- |
| High value | Convert：轉換並保留內容、關係與 metadata。 |
| Medium／Ambiguous | Ask：先提出映射方式、風險與建議，只詢問真正需要決定的項目。 |
| Low value | Archive only：保留來源，不主動變成正式 Obsidian 內容。 |
| Source deleted／system artifact | Skip active migration：不恢復成正式內容，但保留排除原因與 audit evidence。 |

決策表至少包含 `Data type / Count / Value / Recommended action / Reason`。Count 必須標明範圍與單位，區分 unique content、instances、relationships 和 active／deleted；未知值寫「待盤點」，不要填成 0。

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

`heptabase/` 內的子資料夾（例如 `cards/`、`sources/`、`media/`）是遷移時的設計選擇，不是 Heptabase 的原始結構；Heptabase export 本身是扁平的 Card Library。分流規則要寫下來，才能重現。

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

### MindMap 與 nested whiteboards

MindMap 不能因為目標格式不同就直接 skip。來源在 `mindMapList` 的 nodes 與 `mindMapEdges`；每個 instance 使用穩定 ID 轉換 nodes 與 edges，edges 保留 `fromNode`／`toNode`／`fromSide`／`toSide`。沒有原始座標時可採可重現的樹狀 layout，但必須說明這不是像素級還原。

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

Heptabase 富文本可能使用 ProseMirror JSON。必須遞迴提取 text nodes；空 paragraph 不一定有 `content`，不要用固定深度索引。

### 欄位名稱不能用猜的

各種 instance 指向來源物件的欄位名稱並不一致。實測中，`mediaCardInstances` 指向 media card 的欄位是 `cardId`，不是 `mediaCardId`；`textElements` 直接帶 `whiteboardId`，不經過 instance table。每種 instance 都要先實際檢查欄位，再寫 join 邏輯。

### Placeholder 是警訊，不是終點

`*(empty card)*` 這類 placeholder 或 fallback，不代表轉換成功，而是在問：為什麼這個 source object 最後只能變成 placeholder？每個 placeholder 都要回查來源，分成「可以復原」與「來源真的沒有內容」。

同樣地，缺圖要分開兩種情況：Migration 漏掉的，與來源 export 本來就沒有 binary 的。若 source object、placement 與 `fileId` 都在，但 export 裡確實沒有檔案，標記為 `source-unrecoverable` 並保留 object 與位置，不要讓 Agent 反覆重試不存在的檔案。

### 媒體檔名可能碰撞

大量來源檔可能都叫 `image.png`，export 後才被加上數字後綴。可以使用檔名、size、source ID 與 metadata 建立候選，但同 size 不保證是同一檔案。多候選必須列為 ambiguous，不能直接選第一個。

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

## 版本、來源與授權

- Public Edition：`1.0.0-beta`
- 技術來源：Heptabase → Obsidian Migration Guide v3.2 / field-tested specification（2026-09-21）
- 技術來源 SHA-256：`25F67AB318F56DD75F2FF24FB51AEFCF1E552CC6984CC54E7727ADD76E4B2DE8`
- Public Edition 更新日期：2026-09-25
- 授權：Creative Commons Attribution 4.0 International（CC BY 4.0）

你可以依 CC BY 4.0 分享與改作這份 Public Edition，但必須標示作者 Cindy Young、作品名稱、授權方式，並說明是否做過修改。案例中的個人資料、原始 Vault、第三方圖片與軟體本身不因本文件授權而自動改變其權利狀態。
