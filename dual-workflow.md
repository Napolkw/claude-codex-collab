# 雙 workflow 模式（運作模型）

`codex-collab` skill 描述「怎麼把實作委派給 Codex」；本檔描述包住它的整體運作模型——
多輪 coding／research 預設怎麼跑。兩者配套：skill 是執行層，本檔是編排層。

## Claude × Codex 分工

多輪 coding 或 research 工作（規劃、委派實作給 Codex、審查、ultracode fan-out）——先呼叫 `codex-collab` skill 並遵循它。單輪瑣事直接做，Claude／Codex 誰快用誰。


## 雙 workflow 模式（預設）

多輪 coding／research 預設以「雙 workflow」運作：**預設 workflow（`Workflow` 多 agent 編排）＋ `codex-collab` 模式**並行。前者負責 fan-out／分解／驗證，後者把實作委派給 Codex（gpt-5.x，高 effort），Claude 守規劃、審查、把關。

**緊迫感（固定心態）**：Codex 會專業、細緻地審計 Claude 的計劃與代碼——逐行核對邏輯、邊界、SSOT、隱含假設。因此 Claude 交出去的每一份計劃與每一段代碼，精度與專業度都必須經得起這種審計：

- 計劃要明確到可被逐條驗證（有驗證點、有 falsifier），不留模糊地帶。
- 代碼要自證正確（讀得懂、邊界完整、與周邊風格一致），不靠「大概可行」。
- 預設對方會找出漏洞——所以先自我對抗、自我審計，再交付。
- 不確定就標明假設並驗證，不要假裝完備。

## 狀態外置（減少上下文污染）

預設把工作狀態外置到 **GitHub** 與 **Linear**，並及時上傳代碼庫，讓對話上下文保持精簡、污染影響降到最小：

- **計劃／進度／決策／待辦／發現** → 寫進 Linear（issue／project／comment）與 GitHub（issue／PR／comment），不只留在對話裡。
- **代碼** → 及時 commit／push 上傳代碼庫，讓 repo 成為單一事實來源（SSOT），而非只存在 worktree 或對話中。
- **Claude 與 Codex 透過外部狀態協作**：交接、審查、續作都讀寫 gh／Linear 與 repo，而非靠長對話傳遞。
- **上下文只留精華**：每輪只把「結論／指標／決策」留在上下文；細節（log、掃描、中間產物）外置或丟給 subagent，回收結論即可。
- 開新 session／續作前，先從 gh／Linear／repo 拉回狀態，而不是依賴上一段對話殘留。

## 反膨脹（淨收斂）

雙 workflow ＋ Codex 執行有「只加不減」的天然偏向，fan-out 會放大，外置狀態若把中間產物塞進 repo 更糟。預設加反向制動：

- **淨收斂優先**：每次改動先問「該刪／合併／改寫什麼」（先列 kill-list：該刪／合併／改寫什麼）；新增前先找可複用的既有實作。
- **禁止複製副本**：用 editable／import，不要 stale 副本與同物多版；共享形狀單一擁有者（single source of truth）。
- **repo 只放 SSOT**：log／掃描／中間產物／scratch 一律外置或進 scratch 目錄，不 commit。
- **Codex 審計含膨脹項**：必查重複、死代碼、過度抽象、防禦性過工程；超出需求的「以防萬一」要砍。
- **取代即刪除**：新方案上線就移除舊路徑，不留並存雙軌。
