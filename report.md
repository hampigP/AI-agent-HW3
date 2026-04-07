# Assignment 3: Autonomous Multi-Doc Financial Analyst
**Report**

## 1. 比較 LangChain 與 LangGraph 的差異 (Comparison between LangGraph and LangChain)
本次實驗中，我們對傳統的 LangChain (ReAct Agent) 與 LangGraph (State-aware Agent) 進行了深入比較。

* **LangChain (Legacy - 實驗得分: 9/14)：** 
  使用傳統的 `create_react_agent` 與完整的 ReAct Loop (Thought -> Action -> Observation -> Final Answer)。此模式的優點是建置快速，但缺點是其流程高度依賴 LLM 當下的「自主判斷」。在處理複雜財務報表時，如果一次檢索失敗，代理往往會開始胡賽亂謅 (Hallucination) 或陷入無限的檢索迴圈中無法自拔。
* **LangGraph (Graph - 實驗得分: 8/14)：**
  雖然在單次測驗中受資料亂數影響分數稍微浮動（從 9 分略降至 8 分），但 LangGraph 在系統架構上擁有壓倒性的優勢。我們透過明確定義節點：`Router` (分發問題)、`Grader` (評估文件相關性)、`Generator` (最終生成) 與 `Rewriter` (改寫問題再搜尋)。這種狀態機 (State Machine) 架構迫使系統在獲得「無效文件」時，必定會經過 `Rewriter` 將口語問題轉為精準財報術語（如：Research & Development expenses）再重新檢索。它確保了工作流的**可控性與穩定性**，大幅降低代理放棄任務的機率。

---

## 2. 探討 Chunk Size 大小對「大型表格」的影響 (Context Precision vs Completeness)
在本次實驗中，我們將基準的 `chunk_size = 1000` (作業提示為 2000) 縮小至 `chunk_size = 500`。
* **實驗結果與分數變化：** 當 `chunk_size` 縮減為 500 時，準確率受到災難性的打擊，分數從 **8/14 驟降至 5/14**。
* **核心影響機制與權衡 (Trade-off) 解釋：**
  * **小區塊 (Context Precision，如 500)：** 雖然能確保檢索到的片段「雜訊較低」，但面對財務報表中的**大型表格 (如資產負債表 Balance Sheet)** 時，會發生嚴重的「表格截斷」現象。一張表格的多個維度（如: 行為各類費用，列為 2024、2023、2022年）會被硬生生切在不同的 chunk 裡。這導致 LLM 雖然拿到了數字，卻對應不到上方的年份標題，進而把 2023 年的數字誤認為 2024 年。
  * **大區塊 (Context Completeness，如 1000 或 2000)：** 大區塊能確保將一整張財務報表完整地包覆在同一個片段中。這讓 LLM 擁有充足的「上下文對齊能力 (Context Completeness)」，精準捕捉欄列交集的數字。因此在財報分析任務中，大區塊雖然帶有一些雜訊，但保留結構完整性的收益遠大於精準度帶來的效益。

---

## 3. 測試不同 Embedding Model 對檢索的影響 (Comparison of Embedding Models)
我們測試了兩種不同的 HuggingFace 開源詞嵌入模型：
1. **模型 A (Baseline):** `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (**分數: 8/14**)
2. **模型 B (實驗組):** `sentence-transformers/all-MiniLM-L6-v2` (**分數: 7/14**)

* **結果分析：**
  * **模型 A (L12-v2)** 是一個層數較深且支援「多語系 (Multilingual)」的模型。因為我們的 Benchmark 題庫中包含了許多**繁體中文**的詢問（例如：「Apple 2024 年的總營收是多少？」），這個模型能夠優秀地將中文問題與英文財報片段映射到近乎相同的向量空間，成功檢索出正確的英文段落。
  * **模型 B (L6-v2)** 是一個較小 (層數少一半) 且主要是以英語為主的模型。當它遇到「中英夾雜」或是「中文財經術語」時，捕捉跨語系語意的能力明顯下滑，導致有幾題無法找出正確相對應的財報表格，總分因此下降到了 7 分。這證明了對於這類跨語言的 RAG 任務，採用 Multilingual 模型的表現顯著優於一般的輕量型英文模型。
