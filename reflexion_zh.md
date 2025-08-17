# 反思：帶有語言強化學習的語言代理

Noah Shinn  東北大學  noahshinn024@gmail.com

Federico Cassano  東北大學  cassano.f@northeastern.edu

Edward Berman  東北大學  berman.ed@northeastern.edu

Ashwin Gopinath  麻省理工學院  agopi@mit.edu

Karthik Narasimhan  普林斯頓大學  karthikn@princeton.edu

Shunyu Yao  普林斯頓大學  shunyuy@princeton.edu

# 摘要

大型語言模型（LLMs）越來越多地被用作與外部環境（例如，遊戲、編譯器、API）互動的目標導向代理。然而，對於這些語言代理來說，要快速有效地從試錯中學習仍然具有挑戰性，因為傳統的強化學習方法需要大量的訓練樣本和昂貴的模型微調。我們提出了一種名為 **Reflexion** 的新穎框架，它不是透過更新權重來強化語言代理，而是透過**語言回饋**。具體來說，Reflexion 代理會口頭反思任務回饋訊號，然後將其自身的反思文本保存在一個情景記憶緩衝區中，以誘導在後續試驗中做出更好的決策。Reflexion 框架足夠靈活，可以納入各種類型（純量值或自由形式語言）和來源（外部或內部模擬）的回饋訊號，並在多樣化的任務（循序決策、程式編碼、語言推理）上顯著優於基線代理。例如，Reflexion 在 HumanEval 編碼基準測試上達到了 **$91\%$** 的 pass@1 準確度，超越了先前最先進的 **GPT-4** 的 **$80\%$**。我們還進行了消融（ablation）和分析研究，使用不同的回饋訊號、回饋整合方法和代理類型，並提供關於它們如何影響效能的見解。我們在 https://github.com/noahshinn024/reflexion 釋出所有程式碼、示範和資料集。

# 1 引言

諸如 ReAct [30]、SayCan [1]、Toolformer [22]、HuggingGPT [23]、生成式代理 [19] 和 WebGPT [17] 等近期工作已經證明了基於大型語言模型（LLM）核心的自主決策代理是可行的。這些方法使用 LLMs 來生成文本和「動作」，這些文本和動作可以用於 API 呼叫並在環境中執行。由於它們依賴於參數龐大的模型，所以到目前為止，這些方法僅限於使用**上下文範例**作為教導代理的方式，因為像帶有梯度下降的強化學習等更傳統的優化方案需要大量的運算和時間。

在本文中，我們提出了一種名為 **Reflexion** 的替代方法，它使用**口頭強化**來幫助代理從先前的失敗中學習。Reflexion 將來自環境的**二元**或**純量回饋**轉換為**文本摘要**形式的口頭回饋，然後在下一個情景中作為額外的上下文添加到 LLM 代理中。這種**自我反思式回饋**作為一種「語義」梯度訊號，為代理提供了具體的改進方向，幫助它從先前的錯誤中學習，以便更好地執行任務。這類似於人類如何透過反思過去的失敗，在少數幾次嘗試中**迭代地學習**完成複雜任務，從而為下一次嘗試形成改進的攻擊計畫。例如，在圖 1 中，Reflexion 代理透過試驗、錯誤和自我反思，學習優化自身的行為，以解決決策、編程和推理任務。

生成有用的反思性回饋是一項挑戰，因為它需要對模型在哪裡出錯有很好的理解（即**歸因問題** [25]），以及生成包含可操作改進見解的摘要的能力。我們探索了三種實現方式：**簡單的二元環境回饋**、針對常見失敗情況的**預定義啟發式演算法**，以及使用 LLMs 進行的**自我評估**，例如使用 LLM 進行二元分類（決策制定）或**自我編寫單元測試**（編程）。在所有實施中，評估訊號被放大為**自然語言體驗摘要**，可以儲存在**長期記憶**中。

與更傳統的 RL 方法（如基於策略或基於價值）相比，Reflexion 具有幾個優勢：1）它很**輕量**，不需要微調 LLM；2）它允許更細緻的回饋形式（例如，對動作進行有針對性的更改），而純量或向量獎勵難以進行準確的歸因；3）它允許對先前經驗進行更明確和可解釋的**情景記憶**形式；4）它為未來的情景中的動作提供了更明確的提示。同時，它的缺點是**依賴於 LLM 的自我評估能力（或啟發式演算法）**，並且沒有成功的正式保證。然而，隨著 LLM 能力的提高，我們預計這種範式會隨著時間的推移變得更好。

我們進行了以下實驗：（1）**決策制定任務**，以測試長軌跡中的循序動作選擇；（2）**推理任務**，以測試知識密集、單步生成的改進；（3）**編程任務**，以教導代理有效使用外部工具（如編譯器和解釋器）。在這三種類型的任務中，我們觀察到 Reflexion 代理是更好的決策者、推理者和程式設計師。更具體地說，Reflexion 代理在 12 個迭代學習步驟中，在 **AlfWorld** [24] 決策制定任務上比強大的基線方法**絕對提高了 $22\%$**；在 **HotPotQA** [28] 推理問題上**提高了 $20\%$**；在 **HumanEval** [6] Python 編程任務上**提高了高達 $11\%$**。

總結來說，我們的貢獻如下：

- 我們提出 Reflexion，這是一種用於「口頭」強化的新範式，它將策略參數化為代理的記憶編碼與 LLM 參數的選擇相結合。
- 我們探索了 LLM 中這種自我反思的**湧現特性**，並透過實驗證明，自我反思對於在少數幾次試驗中學習複雜任務極為有用。
- 我們引入了 **LeetcodeHardGym**，這是一個程式碼生成 RL 體育館環境，包含 40 個具有挑戰性的 Leetcode 問題（「困難級別」），涵蓋 19 種編程語言。
- 我們展示了 Reflexion 在多項任務上優於強大基線，並在各種程式碼生成基準測試上取得了**最先進的結果**。

# 2 相關工作

**推理與決策制定** Self-Refine [15] 採用**迭代框架**，透過自我評估來自主改進生成。這些自我評估和自我改進步驟以給定的任務約束為條件，例如「如何以更積極的方式寫這段生成內容」。Self-Refine 是有效的，但**僅限於單次生成推理任務**。Pryzant et al. [21] 執行類似的**語義提示編寫優化**，但同樣僅限於單次生成任務。Paul et al. [20] 微調批評者模型，以在軌跡內提供**中間回饋**來改進推理回應。Xie et al. [27] 使用**隨機集束搜尋**（stochastic beam search）來執行更高效的決策制定搜尋策略，這使得代理能夠利用其自我評估組件帶來的先見之明。Yoran et al. [31] 和 Nair et al. [16] 使用決策者模型來**推理多個生成**。Kim et al. [10] 在沒有評估步驟的情況下，使用**固定步數的重試模式**。Goodman [9] 執行**定性評估步驟**，對先前的生成提出優化建議。在本文中，我們展示了其中幾個概念可以透過**自我反思**來增強，以建立一個持續存在的自我反思體驗記憶，這使得代理能夠識別自己的錯誤，並自我建議從錯誤中學習的教訓。

![](https://cdn-mineru.openxlab.org.cn/result/2025-08-14/8ceea438-aa32-4770-aadc-9b35f2086ce4/08b2a4fb09712c8dc9a376e2c1a20a1789fad9bf198ee1cf9433956f37202e44.jpg)

圖 1: Reflexion 適用於決策制定 4.1、編程 4.3 和推理 4.2 任務。

![](https://cdn-mineru.openxlab.org.cn/result/2025-08-14/8ceea438-aa32-4770-aadc-9b35f2086ce4/d763ecbeb58e03fab0682408983ef46e9b26e4423e04ae34c628957ae9f14db5.jpg)

**編程** 過去和最近的幾項工作採用了**測試驅動開發**或**程式碼除錯**實踐的變體。AlphaCode [14] 在隱藏的測試案例上評估一組生成內容。CodeT [5] 使用**自我生成的單元測試**來評分生成的函式實現。Self-Debugging [7] 使用一個**除錯組件**，該組件在收到來自程式碼執行環境的回饋後，用於改進現有的實現。CodeRL [12] 使用**actor-critic**設定將問題置於 RL 框架中，以根據來自執行環境的回饋來除錯程式。AlphaCode、Self-Debugging 和 CodeRL 在修復較不複雜的程式錯誤方面是有效的，但它們依賴**基本事實測試案例**，這會使 pass@1 的資格失效，並且不使用自我反思來彌合錯誤識別和實現改進之間的鴻溝。CodeT 不會存取隱藏的測試案例，但沒有實施自我學習步驟來改進程式碼編寫。

# 3 Reflexion：透過口頭反思進行強化

我們為 Reflexion 開發了一個**模組化公式**，利用了三個不同的模型：一個被稱為 **$M__{a}$** 的**執行者**（Actor），它負責生成文本和動作；一個被稱為 **$M__{e}$** 的**評估者**（Evaluator）模型，它對 **$M__{a}$** 產生的輸出進行評分；以及一個被稱為 **$M__{sr}$** 的**自我反思模型**（Self-Reflection），它生成口頭強化提示來協助執行者進行自我改進。我們將詳細描述這些模型中的每一個，並隨後闡明它們如何在 Reflexion 框架內協同運作。

![[img1.png]]

# 深入了解 Reflexion 框架的運作機制

Reflexion 是一個基於大型語言模型（LLM）的框架，旨在透過**自我反思**來提升智能體在複雜任務中的決策能力。它主要由四個核心組件構成：**Actor**、**Evaluator**、**Self-reflection** 和**Memory**。

## 核心組件

Reflexion 的每個組件都扮演著關鍵角色，共同形成一個高效的學習循環。

### 1. Actor（執行者）

Actor 是整個框架的核心執行單元，它是一個經過特別提示（prompted）的大型語言模型。

* **工作原理**：根據環境觀測（\(o_{t}\)）生成必要的文字或動作（\(a_{t}\)）。這類似於傳統強化學習（RL）中的策略（policy）角色。
* **探索模型**：我們探索了多種 Actor 模型，包括 **Chain of Thought** [26] 和 **ReAct** [30]，以探討不同的文字與動作生成方式。
* **額外功能**：為了提供更多上下文，我們還加入了一個記憶組件 `mem`，其靈感來自於 Brooks et al. [3] 提出的「上下文學習」（in-context learning）方法。

### 2. Evaluator（評估者）

Evaluator 負責評估 Actor 產出的品質，確保其符合任務要求。

* **工作原理**：接收 Actor 生成的軌跡（trajectory），並計算一個獎勵分數（reward score），以反映其在特定任務中的表現。
* **獎勵函數的挑戰**：由於定義適用於語義空間的價值和獎勵函數十分困難，我們嘗試了多種 Evaluator 變體：
    * **推理任務**：使用**精確匹配**（EM）來確保輸出與預期答案一致。
    * **決策任務**：使用預先定義的**啟發式函數**進行評估。
    * **通用方法**：直接使用另一個 **LLM** 來生成決策與程式設計任務的獎勵。

### 3. Self-reflection（自我反思）

Self-reflection 模組是 Reflexion 框架的關鍵創新點，它能將稀疏的獎勵訊號轉化為豐富且有意義的反饋。

* **工作原理**：
    1.  接收稀疏的獎勵訊號（例如**成功**/**失敗**）、當前軌跡和記憶 `mem`。
    2.  生成**口頭式的自我反思**，提供比單一分數更具資訊量的回饋。
    3.  這些反饋會被儲存在記憶 `mem` 中，供未來試驗使用。
* **範例**：在多步驟決策任務中，如果智能體失敗，它可以反思是哪個特定動作（\(a_{i}\)）導致了後續的錯誤，並推斷出**應採取**的正確動作（\(a_{i}^{\prime}\)），然後將此經驗存入記憶。

### 4. Memory（記憶）

Memory 是 Reflexion 實現長期與短期學習的關鍵。

* **短期記憶**：軌跡歷史（trajectory history）被視為短期記憶，記錄最近發生的細節。
* **長期記憶**：來自 Self-reflection 模組的輸出會被儲存為長期記憶，記錄從多次試驗中提煉出的重要經驗。
* **協同運作**：這兩種記憶共同為 Actor 提供了豐富的上下文，使其決策既能參考最近的細節，也能從過去的教訓中學習，這是 Reflexion 優於其他 LLM 決策方法的關鍵優勢。

> **備註**：為了符合 LLM 的最大上下文限制，記憶 `mem` 的大小會被限制在一個最大值 \(\Omega\)（通常設定為 1-3 個經驗）。

## Reflexion 的運作流程

Reflexion 是一個迭代優化的過程。

1.  **第一次試驗**：Actor 透過與環境互動，產生軌跡 \(\tau_0\)。
2.  **評估**：Evaluator 根據軌跡 \(\tau_0\) 產生一個分數 \(r_0\)，即 \(r_t = M_e(\tau_0)\)。
3.  **反思**：Self-reflection 模組分析軌跡 \(\tau_0\) 和分數 \(r_0\)，生成反思摘要 \(sr_0\)。
4.  **記憶更新**：\(sr_0\) 被儲存在記憶 `mem` 中。
5.  **循環**：Actor、Evaluator 和 Self-reflection 模組不斷循環，直到 Evaluator 認為軌跡 \(\tau_t\) 正確為止。每次試驗後，`sr_t` 都會被加入 `mem`。

# 4 實驗

我們評估各種自然語言 RL 設定在決策、推理和程式碼生成任務上的表現。具體來說，我們挑戰一個代理程式在 HotPotQA [28] 上執行基於搜尋的問答、在 AlfWorld [24] 中執行常見家庭環境中的多步驟任務，以及在 HumanEval [6]、MIPP [2] 和 LeetcodeHard（一個新的基準測試）中，於類似競賽的環境中使用解釋器和編譯器來執行程式碼撰寫任務。最值得注意的是，Reflexion 在 AlfWorld 中的表現比強大的基準線提高了 \(22\%\)，在 HotPotQA 中提高了 \(20\%\)，在 HumanEval 中提高了 \(11\%\)。

---

## 4.1 序列決策：ALFWorld

AlfWorld 是一個基於 TextWorld [8] 的文字環境套件，旨在挑戰代理程式在各種互動式環境中解決多步驟任務。我們遵循 Yao 等人 [30] 的方法，在 134 個 AlfWorld 環境中執行代理程式，涵蓋六種不同的任務，包括尋找隱藏物品（例如，在抽屜裡找到鍋鏟）、移動物品（例如，將刀子移到砧板上）以及使用其他物品操作物品（例如，將番茄放入冰箱中冷卻）。我們使用 ReAct [30] 作為動作生成器，因為 Yao 等人 [30] 已經證明了它在長軌跡決策中，使用明確的中間思想取得了成功。AlfWorld 任務自然需要一個自我評估步驟，因為環境只能標示任務是否完成。為了實現完全自主的行為，我們實施了兩種自我評估技術：使用大型語言模型（LLM）進行自然語言分類和手寫啟發式演算法。這個啟發式演算法很簡單：如果代理程式執行相同的動作並收到相同的回應超過 3 個週期，或者如果當前環境中採取的動作數量超過 30 次（效率低下），我們就會進行自我反思。在基準線運行中，如果建議進行自我反思，我們會跳過自我反思過程，重設環境並開始新的嘗試。在 Reflexion 運行中，代理程式使用自我反思來找出其錯誤，更新其記憶體，重設環境並開始新的嘗試。為了避免可能超過最大限制的過長提示視窗，我們將代理程式的記憶體截斷為最近 3 次自我反思（經驗）。

為了避免語法錯誤，我們為代理程式提供了兩個特定領域的少樣本軌跡。我們使用與 Yao 等人 [30] 相同的少樣本軌跡範例與 GPT-3 作為 LLM。AlfWorld 任務、ReAct 少樣本提示和 Reflexion 範例都包含在附錄中。

**結果**：ReAct + Reflexion 使用簡單的啟發式演算法來偵測幻覺和低效規劃，顯著優於 ReAct，在 134 個任務中完成了 130 個。此外，ReAct + Reflexion 在連續 12 次嘗試中透過學習解決了額外的任務。在僅 ReAct 的方法中，我們看到性能提升在第 6 次和第 7 次嘗試之間停止了。

**分析**：在基準線失敗的 AlfWorld 軌跡中，一個常見的錯誤是代理程式認為它擁有一件物品，但實際上並沒有。代理程式會繼續執行幾個動作，形成一個很長的軌跡，並且無法回溯其動作以找出錯誤。Reflexion
![](images/7189bd3ede86094f7523430f25337e6e5a6bf7aa7a37c26b657dbdbe6c696f83.jpg)
圖 3: (a) AlfWorld 在 134 個任務上的表現，顯示使用自我評估技術（啟發式）和（GPT）進行二元分類的累積解決任務比例。(b) AlfWorld 軌跡按失敗原因分類。

透過使用自我反思，將失敗的長軌跡提煉成相關經驗，這些經驗在未來可以作為「自我提示」來使用，從而消除了幾乎所有這些情況。在 AlfWorld 中，長期記憶在兩種主要情況下幫助代理程式：1) 可以在長軌跡中的早期錯誤輕鬆識別出來。代理程式可以提出一個新的動作選擇，甚至是一個新的長期計畫。2) 檢查物品的表面/容器太多。代理程式可以利用其在多次嘗試中的經驗記憶來徹底搜尋一個房間。在圖 3 中，學習曲線表明學習過程發生在幾次經驗中，這意味著代理程式成功地平衡了圖中所示的案例 1 和 2，在最初兩次嘗試之間的改進中出現了立竿見影的提升，然後在接下來的 11 次嘗試中穩步增加，達到了近乎完美的表現。另一方面，圖 3 顯示僅 ReAct 的代理程式收斂於 \(22\%\) 的幻覺率，沒有長期恢復的跡象。

---

## 4.2 推理：HotpotQA

HotPotQA [28] 是一個基於維基百科的資料集，包含 11.3 萬個問答對，挑戰代理程式解析內容並在多個支援文件上進行推理。為了僅測試推理能力的改進，我們實施了 Reflexion + Chain-of-Thought (CoT) [26] 的逐步 \(Q \rightarrow A\) 和 \(Q, C_{gt} \rightarrow A\) 實現，其中 \(Q\) 是問題，\(C_{gt}\) 是來自資料集的真實情境，\(A\) 是最終答案。由於 CoT 不是一種多步驟決策技術，我們將 \(C_{gt}\) 提供給代理程式，以便我們可以在提供的長文本部分中隔離推理行為。為了測試整體問答能力，這需要推理和動作選擇，我們實施了一個 Reflexion + ReAct [30] 代理程式，該代理程式可以使用維基百科 API 檢索相關情境，並使用逐步的顯式思維來推斷答案。對於 CoT 實現，我們使用 6 次少樣本提示；對於 ReAct，我們使用 2 次少樣本提示；對於自我反思，我們使用 2 次少樣本提示。所有範例都可以在附錄中找到。

穩健地評估自然語言答案是 NLP 中一個長期存在的問題。因此，在嘗試之間，我們使用環境中的精確匹配答案評分來給予代理程式一個二元成功信號。每次嘗試後，都會使用自我反思迴圈來放大二元信號，類似於 AlfWorld 中的決策設定 4.1，記憶體大小為 3 次經驗。

**結果**：Reflexion 在幾個學習步驟中以顯著的優勢超越了所有基準線方法。此外，僅 ReAct、僅 CoT 和僅 CoT (GT) 的實現未能概率性地改進任何任務，這意味著任何基準線方法在第一次嘗試中失敗的任務，在後續嘗試中使用 \(0.7\) 的溫度都無法被解決。在 Reflexion 運行中，我們允許代理程式收集經驗並在失敗的任務上重試，直到它在特定任務上連續產生 3 次失敗嘗試。自然地，CoT (GT) 由於可以存取問題的真實情境而獲得了更高的準確率。儘管如此，CoT (GT) 代理程式仍然無法正確推斷出 \(39\%\) 問題的正確答案，但 Reflexion 幫助代理程式在無法存取真實答案的情況下糾正其錯誤，從而將其準確率提高了 \(14\%\)。
![](images/acd78fea60546b01bf6f9e94fe27ad70e010993d4290f7fea355ac40b7ce8be0.jpg)
圖 4: Chain-of-Thought (CoT) 和 ReAct。Reflexion 提高了在 100 個 HotPotQA 問題上的搜尋、資訊檢索和推理能力。(a) Reflexion ReAct 與 Reflexion CoT (b) 僅用於推理的 Reflexion CoT (GT) (c) Reflexion 與情節記憶消融。

**分析**：我們進行了一項消融實驗，以隔離自我反思步驟對推理的優勢，使用 CoT (GT) 作為基準線方法。回想一下，CoT (GT) 使用了帶有提供真實情境的 Chain-of-Thought 推理，這測試了在長情境上的推理能力。接下來，我們透過包含最近的軌跡來添加一個情節記憶 (EPM) 的元素。對於 Reflexion 代理程式，我們實施了標準的自我反思步驟作為最後一道關卡。直觀地說，我們測試代理程式是否透過使用第一人稱撰寫的語言來進行口頭解釋，從而更有效地進行迭代學習。圖 4 顯示，自我反思比情節記憶的學習優勢高出 \(8\%\) 的絕對提升。這個結果支持了這樣的論點：僅精煉的方法不如自我反思引導的精煉方法有效。

---

## 4.3 程式設計

我們在 MBPP [2]、HumanEval [6] 和我們的新資料集 LeetcodeHardGym 上評估了基準線和 Reflexion 方法在 Python 和 Rust 程式碼撰寫方面的表現。MBPP 和 HumanEval 衡量的是給定自然語言描述時的函式主體生成準確性。我們使用基準語言編譯器 MultiPL-E [4]，將 HumanEval 和 MBPP 的子集翻譯成 Rust 語言。MultiPL-E 是一系列小型編譯器，可用於將 Python 基準問題翻譯成其他 18 種語言。我們包含了 Rust 程式碼生成的實驗，以證明 Reflexion 程式碼生成的實現與語言無關，並且可以用於解釋型和編譯型語言。最後，我們引入了一個新的基準測試 LeetcodeHardGym，這是一個互動式程式設計練習場，包含 40 個在 2022 年 10 月 8 日之後發布的 Leetcode hard 等級問題，這是 GPT-4 [18] 的預訓練截止日期。

程式設計任務提供了一個獨特的機會，可以使用更紮實的自我評估方法，例如自我生成的單元測試套件。因此，我們基於 Reflexion 的程式設計任務實施符合 pass@1 準確性報告的資格。為了生成測試套件，我們使用 Chain-of-Thought 提示 [26] 來產生具有相應自然語言描述的多樣化、廣泛的測試。然後，我們透過嘗試為每個提議的測試構建一個有效的抽象語法樹（AST）來篩選出語法有效的測試語句。最後，我們從生成的單元測試集合中採樣 \(n\) 個測試，以產生一個測試套件 \(T\)，記作 \(\{t_0, t_1, \ldots , t_n\}\)。我們將 \(n\) 設為最多 6 個單元測試。除了單元測試套件部分，Reflexion 程式設計代理程式的學習迴圈設置與推理和決策代理程式的設置相同，記憶體最大限制為 1 次經驗。

| 基準測試 + 語言 | 先前 SOTA Pass@1 | SOTA Pass@1 | Reflexion Pass@1 |
| :--- | :--- | :--- | :--- |
| HumanEval (PY) | 65.8 (CodeT [5] + GPT-3.5) | 80.1 (GPT-4) | 91.0 |
| HumanEval (RS) | - | 60.0 (GPT-4) | 68.0 |
| MBPP (PY) | 67.7 (CodeT [5] + Codex [6]) | 80.1 (GPT-4) | 77.1 |
| MBPP (RS) | - | 70.9 (GPT-4) | 75.4 |
| LeetcodeHard (PY) | - | 7.5 (GPT-4) | 15.0 |

表 1: 各種模型-策略-語言組合的 Pass \(@1\) 準確性。基本策略是單次程式碼生成樣本。所有基於指令的模型都遵循零樣本程式碼生成。

| 基準測試 + 語言 | Base | Reflexion | TP | FN | FP | TN |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| HumanEval (PY) | 0.80 | 0.91 | 0.99 | 0.40 | 0.01 | 0.60 |
| MBPP (PY) | 0.80 | 0.77 | 0.84 | 0.59 | 0.16 | 0.41 |
| HumanEval (RS) | 0.60 | 0.68 | 0.87 | 0.37 | 0.13 | 0.63 |
| MBPP (RS) | 0.71 | 0.75 | 0.84 | 0.51 | 0.16 | 0.49 |

表 2: HumanEval 和 MBPP 的整體準確性和測試生成表現。對於 Rust，HumanEval 是從 HumanEval Python 翻譯成 Rust 的最難的 50 個問題，使用 MultiPL-E [4]。TP: 單元測試通過，解決方案通過；FN: 單元測試失敗，解決方案通過；FP: 單元測試通過，解決方案失敗；TN: 單元測試失敗，解決方案失敗。

**結果**：Reflexion 超越了所有基準線準確性，並在除 MBPP Python 之外的所有 Python 和 Rust 基準測試上設定了新的最先進標準 1。我們進一步調查了 Reflexion 在 MBPP Python 上表現不佳的原因。

**分析**：我們承認，自我反思的程式碼生成代理程式受限於其編寫多樣化、全面測試的能力。因此，在模型生成了一個不穩定的測試套件的情況下，所有測試都可能在一個不正確的解決方案上通過，並導致程式碼完成的**偽陽性**標籤 [11]。另一方面，如果模型生成了一個編寫不正確的測試套件，一些測試可能在一個正確的解決方案上失敗，導致生成一個基於**偽陰性**程式碼完成的自我反思。鑑於 Reflexion 的實施，偽陰性比偽陽性更受歡迎，因為代理程式可能能夠使用自我反思來識別不正確的測試並提示自己保持原始的程式碼完成不變。另一方面，如果一個無效的測試套件返回一個偽陽性完成（所有內部測試案例都通過，但實施不正確），代理程式將過早地報告一個無效提交。在表 2 中，測量了各種條件來分析 pass@1 準確性之外的表現。之前，我們展示了 Reflexion 在 MBPP Python 上不如基準線 GPT-4 的表現。在表 2 中，我們觀察到內部測試執行產生的偽陽性標籤之間存在顯著差異，P(pass@1 生成不正確 | 測試通過)。也就是說，一個提交在通過所有單元測試後會失敗的機率。對於 HumanEval 和 MBPP Python，基準線 pass@1 準確性相對相似，分別為 82% 和 80%。然而，MBPP Python 的偽陽性測試執行率為 16.3%，而 HumanEval Python 的比率僅為 1.4%，這導致了 91% 的整體準確性 1。

| 方法 | 測試生成 | 自我反思 | Pass@1 (準確性) |
| :--- | :--- | :--- | :--- |
| 基本模型 | 否 | 否 | 0.60 |
| 省略測試生成 | 否 | 是 | 0.52 |
| 省略自我反思 | 是 | 否 | 0.60 |
| Reflexion | 是 | 是 | 0.68 |

表 3: 使用 GPT-4 作為基本模型，在 HumanEval Rust - 50 個最難問題的 Reflexion 方法上，各種受損方法的 Pass@1 準確性。

**消融研究**：我們在 50 個最難的 HumanEval Rust 問題的子集上測試了 Reflexion 複合方法中測試生成和自我反思的協同作用。我們的 Rust 編譯器環境提供了詳細的錯誤日誌和有用的除錯提示，因此可以作為受損方法的良好試驗場。首先，我們省略了內部測試生成和執行步驟，這測試了代理程式在沒有當前實施指導的情況下進行自我反思的能力。表 3 顯示了 52% 的準確性，低於 60%（基準線），這表明代理程式在沒有單元測試的情況下無法確定當前實施是否正確。因此，代理程式必須參與運行的所有迭代，沒有提早返回的選項，從而對實施進行有害的編輯。

接下來，我們透過在失敗的單元測試套件評估後省略自然語言解釋步驟來測試自我反思的貢獻。直觀地說，這挑戰了代理程式在所有失敗的單元測試中，結合錯誤識別和實施改進的任務。有趣的是，受損代理程式的性能並沒有比基準線運行有所提高。我們觀察到，測試生成和程式碼編譯步驟能夠捕獲語法和邏輯錯誤，但實施修復並未反映這些指示。這些實證結果表明，最近一些在沒有自我反思的情況下提出盲目試錯除錯技術的工作，在諸如撰寫複雜 Rust 程式等較難的任務上是無效的。

> **「消融研究」**的實驗，旨在探討一個稱為 Reflexion 的複合方法中，各個組成部分的重要性。
> 簡單來說，研究人員想要知道：
>  - 整個方法為什麼有效？
>  - 如果移除某些部分，效果會變差多少？
>  - 每個部分在整個過程中扮演了什麼角色？
> 這段文字將實驗過程和結果拆成兩部分來解釋。
> 實驗一：移除「內部測試生成和執行」
> 這項實驗的目的是想看看，如果沒有單元測試的指導，代理人（agent）是否還能進行有效的自我反思
>  - 實驗方法：研究人員移除了 Reflexion 方法中，讓 AI 自己生成並運行測試的步驟。
>  - 預期：代理人無法確定自己寫的程式碼是否正確，因為它沒有任何客觀的依據。
>  - 實驗結果：準確率從基準線的 60% 下降到 52%。這證明了代理人必須有單元測試來引導它。如果沒有單元測試，它就無法判斷程式碼是否需要修改，甚至可能做出有害的修改，導致程式碼變得更糟。
>  實驗二：移除「自然語言解釋步驟」（自我反思
>  這項實驗的目的是想看看，如果沒有自然語言的解釋，代理人是否能單純依靠測試結果來修復程式碼。
>   **實驗方法**：研究人員移除了 Reflexion 方法中，讓 AI 根據失敗的測試結果，以自然語言進行反思和解釋的步驟。
>   **預期**：這將迫使代理人直接從錯誤中學習，挑戰它在沒有人類可讀的解釋下，結合錯誤識別和程式碼改進的能力。
>   **實驗結果**：準確率沒有任何提升，表現與基準線一樣。這表明，儘管編譯器和單元測試能夠捕捉到語法和邏輯錯誤，但代理人無法單獨將這些錯誤轉換成有效的程式碼修復。這說明了「自然語言解釋」（或稱「自我反思」）這一步驟，在將錯誤訊息轉化為可行的改進策略上，扮演了至關重要的角色。
>   *總結**
>   這項消融研究的結論是，「內部測試生成和執行」 與 「自然語言自我反思」 這兩個步驟，對於 Reflexion 方法在 Rust 程式碼生成這類複雜任務上是缺一不可的。如果少了其中任何一個，方法的性能都會顯著下降。這也間接證明了，僅僅依賴「盲目試錯」的除錯方法，在面對複雜的程式設計問題時是無效的。

---

## 5 侷限性

Reflexion 的核心是一種使用自然語言進行策略優化的優化技術。策略優化是一種透過經驗來改進行動選擇的強大方法，但它仍然可能屈服於非最佳的局部最小值解決方案。在本研究中，我們將長期記憶限制為具有最大容量的滑動視窗，但我們鼓勵未來的研究工作使用更進階的結構來擴展 Reflexion 的記憶體組件，例如向量嵌入資料庫或傳統 SQL 資料庫。針對程式碼生成，測試驅動開發在指定準確的輸入-輸出對應方面存在許多實際侷限性，例如非確定性生成器函式、與 API 互動的非純函式、根據硬體規格而變化的輸出函式，或調用平行或並行行為而難以預測的函式。

---

## 6 更廣泛的影響

大型語言模型越來越多地被用於與外部環境（例如網際網路、軟體、機器人等）和人類互動。我們的工作有潛力強化這些代理程式，使其實現更大的自動化和工作效率，但當這些代理程式被濫用時，也會放大其風險。我們認為，這個研究方向將需要在安全和倫理考量方面投入更多努力。

另一方面，強化學習一直因其黑箱策略和優化設置而飽受解釋性和對齊方面的挑戰。我們提出的「口頭」強化學習可能會解決其中一些問題，並使自主代理程式更具可解釋性和可診斷性。例如，在人類可能難以理解的工具使用情況下，可以監控自我反思，以確保在使用工具之前意圖正確。

---

## 7 結論

在這項工作中，我們提出了 Reflexion，一種利用口頭強化來教導代理程式從過去的錯誤中學習的方法。我們透過實證證明，Reflexion 代理程式利用自我反思，顯著優於目前廣泛使用的決策方法。在未來的工作中，Reflexion 可以被用來採用在傳統 RL 設定中已被徹底研究的更進階技術，例如自然語言中的價值學習或離線探索技術。

---

## 8 可重現性

我們強烈建議其他人在進行自主程式碼撰寫實驗時使用隔離的執行環境，因為生成的程式碼在執行前未經驗證。

## 參考文獻

[1] Ahn, M., Brohan, A., Brown, N., Chebotar, Y., Cortes, O., David, B., Finn, C., Gopalakrishnan, K., Hausman, K., Herzog, A., et al. (2022). **「言行不一」：將語言紮根於機器人的行動力** (Do as i can, not as i say: Grounding language in robotic affordances)。arXiv 預印本 arXiv:2204.01691。

[2] Austin, J., Odena, A., Nye, M., Bosma, M., Michalewski, H., Dohan, D., Jiang, E., Cai, C., Terry, M., Le, Q., et al. (2021). **使用大型語言模型的程式合成** (Program synthesis with large language models)。arXiv 預印本 arXiv:2108.07732。

[3] Brooks, E., Walls, L., Lewis, R. L., and Singh, S. (2022). **情境內策略迭代** (In- context policy iteration)。arXiv 預印本 arXiv:2210.03821。

[4] Cassano, F., Gouwar, J., Nguyen, D., Nguyen, S., Phipps- Costin, L., Pinckney, D., Yee, M.- H., Zi, Y., Anderson, C. J., Feldman, M. Q., Guha, A., Greenberg, M., and Jangda, A. (2022). **Multipl-e: 一種用於基準測試神經網路程式碼生成的、可擴展且可延伸的方法** (Multipl- e: A scalable and extensible approach to benchmarking neural code generation)。

[5] Chen, B., Zhang, F., Nguyen, A., Zan, D., Lin, Z., Lou, J.- G., and Chen, W. (2022). **Codet: 使用生成的測試進行程式碼生成** (Codet: Code generation with generated tests)。arXiv 預印本 arXiv:2207.10397。

[6] Chen, M., Tworek, J., Jun, H., Yuan, Q., Pinto, H. P. d. O., Kaplan, J., Edwards, H., Burda, Y., Joseph, N., Brockman, G., et al. (2021). **評估在程式碼上訓練的大型語言模型** (Evaluating large language models trained on code)。arXiv 預印本 arXiv:2107.03374。

[7] Chen, X., Lin, M., Schärli, N., and Zhou, D. (2023). **教導大型語言模型進行自我除錯** (Teaching large language models to self- debug)。arXiv 預印本 arXiv:2304.05128。

[8] Cote, M.- A., Kadar, A., Yuan, X., Kybartas, B., Barnes, T., Fine, E., Moore, J., Hausknecht, M., El Asri, L., Adada, M., et al. (2019). **Textworld: 一個用於基於文字遊戲的學習環境** (Textworld: A learning environment for text- based games)。收錄於《Computer Games: 7th Workshop, CGW 2018: Held in Conjunction with the 27th International Conference on Artificial Intelligence, IJCAI 2018, Stockholm, Sweden, July 13, 2018, Revised Selected Papers 7》，第 41-75 頁。Springer。

[9] Goodman, N. (2023). **Meta-prompt: 一個簡單的自我改進語言代理** (Meta- prompt: A simple self- improving language agent)。noahgoodman.substack.com。

[10] Kim, G., Baldi, P., and McAleer, S. (2023). **語言模型可以解決電腦任務** (Language models can solve computer tasks)。arXiv 預印本 arXiv:2303.17491。

[11] Lam, W., Winter, S., Wei, A., Xie, T., Marinov, D., and Bell, J. (2020). **一項關於不穩定測試的大規模縱向研究** (A large- scale longitudinal study of flaky tests)。Proc. ACM Program. Lang., 4(OOPSLA)。

[12] Le, H., Wang, Y., Gotmare, A. D., Savarese, S., and Hoi, S. C. H. (2022). **Coderl: 透過預訓練模型和深度強化學習掌握程式碼生成** (Coderl: Mastering code generation through pretrained models and deep reinforcement learning)。收錄於《Advances in Neural Information Processing Systems, 35:21314- 21328》。

[13] Li, R., Allal, L. B., Zi, Y., Muennighoff, N., Kocetkov, D., Mou, C., Marone, M., Akiki, C., Li, J., Chim, J., et al. (2023). **StarCoder: 願原始碼與你同在！** (Starcoder: may the source be with you!)。arXiv 預印本 arXiv:2303.06161。

[14] Li, Y., Choi, D., Chung, J., Kushman, N., Schrittwieser, J., Leblond, R., Eccles, T., Keeling, J., Gimeno, F., Dal Lago, A., et al. (2022). **使用 AlphaCode 進行競賽級別的程式碼生成** (Competition- level code generation with alphacode)。Science, 378(6624):1092- 1097。

[15] Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegrefe, S., Alon, U., Dziri, N., Prabhumoye, S., Yang, Y., et al. (2023). **自我精煉：透過自我回饋進行迭代精煉** (Self- refine: Iterative refinement with self- feedback)。arXiv 預印本 arXiv:2303.17651。

[16] Nair, V., Schumacher, E., Tso, G., and Kannan, A. (2023). **Dera: 使用對話啟用的解析代理增強大型語言模型的補全** (Dera: Enhancing large language model completions with dialog- enabled resolving agents)。arXiv 預印本 arXiv:2303.17071。

[17] Nakano, R., Hilton, J., Balaji, S., Wu, J., Ouyang, L., Kim, C., Hesse, C., Jain, S., Kosaraju, V., Saunders, W., et al. (2021). **Webgpt: 透過瀏覽器輔助的人類回饋問答** (Webgpt: Browser- assisted question- answering with human feedback)。arXiv 預印本 arXiv:2112.09332。

[18] OpenAI (2023). **Gpt-4 技術報告** (Gpt- 4 technical report)。ArXiv。

[19] Park, J. S., O'Brien, J. C., Cai, C. J., Morris, M. R., Liang, P., and Bernstein, M. S. (2023). **生成式代理：人類行為的互動式模擬** (Generative agents: Interactive simulacra of human behavior)。arXiv 預印本 arXiv:2304.03442。

[20] Paul, D., Ismayilzada, M., Peyrard, M., Borges, B., Bosselut, A., West, R., and Faltings, B. (2023). **Refiner: 中間表示法上的推理回饋** (Refiner: Reasoning feedback on intermediate representations)。arXiv 預印本 arXiv:2304.01904。

[21] Pryzant, R., Iter, D., Li, J., Lee, Y. T., Zhu, C., and Zeng, M. (2023). **使用「梯度下降」和束搜索進行自動提示優化** (Automatic prompt optimization with "gradient descent" and beam search)。arXiv 預印本 arXiv:2305.03493。

[22] Schick, T., Dwivedi- Yu, J., Dessi, R., Raileahu, R., Lomeli, M., Zettlemoyer, L., Cancedda, N., and Scialom, T. (2023). **Toolformer: 語言模型可以教自己使用工具** (Toolformer: Language models can teach themselves to use tools)。arXiv 預印本 arXiv:2302.04761。

[23] Shen, Y., Song, K., Tan, X., Li, D., Lu, W., and Zhuang, Y. (2023). **Hugginggpt: 與 ChatGPT 及其在 Hugging Face 上的朋友們解決 AI 任務** (Hugginggpt: Solving ai tasks with chatgpt and its friends in huggingface)。arXiv 預印本 arXiv:2303.17580。

[24] Shridhar, M., Yuan, X., Cote, M.- A., Bisk, Y., Trischler, A., and Hausknecht, M. (2021). **ALFWorld: 對齊文字與具身環境以進行互動式學習** (ALFWorld: Aligning Text and Embodied Environments for Interactive Learning)。收錄於《Proceedings of the International Conference on Learning Representations (ICLR)》。

[25] Sutton, R. S. and Barto, A. G. (2018). **強化學習：導論** (Reinforcement Learning: An Introduction)。The MIT Press, 第二版。

[26] Wei, J., Wang, X., Schuurmans, D., Bosma, M., Chi, E., Le, Q., and Zhou, D. (2022). **思維鏈提示引發大型語言模型中的推理** (Chain of thought prompting elicits reasoning in large language models)。arXiv 預印本 arXiv:2201.11903。

[27] Xie, Y., Kawaguchi, K., Zhao, Y., Zhao, X., Kan, M.- Y., He, J., and Xie, Q. (2023). **透過自我評估引導解碼增強分解式推理** (Decomposition enhances reasoning via self- evaluation guided decoding)。arXiv 預印本 arXiv:2305.00633。

[28] Yang, Z., Qi, P., Zhang, S., Bengio, Y., Cohen, W. W., Salakhutdinov, R., and Manning, C. D. (2018). **HotpotQA: 一個用於多跳問答的、可解釋的多樣化資料集** (HotpotQA: A dataset for diverse, explainable multi- hop question answering)。收錄於《Conference on Empirical Methods in Natural Language Processing (EMNLP)》。

[29] Yao, S., Chen, H., Yang, J., and Narasimhan, K. (預印本). **Webshop: 邁向與紮根語言代理進行可擴展的真實世界網路互動** (Webshop: Towards scalable real- world web interaction with grounded language agents)。收錄於《ArXiv》。

[30] Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., and Cao, Y. (2023). **ReAct: 在語言模型中協同推理與行動** (ReAct: Synergizing reasoning and acting in language models)。收錄於《International Conference on Learning Representations (ICLR)》。

[31] Yoran, O., Wolfson, T., Bogin, B., Katz, U., Deutch, D., and Berant, J. (2023). **透過對多個思維鏈進行元推理來回答問題** (Answering questions by meta- reasoning over multiple chains of thought)。arXiv 預印本 arXiv:2304.13007。

### A 使用額外模型的評估

我們進一步探討了*試錯問題解決*方法在不同強度的模型上的適用性。我們發現，指定**自我修正**的能力是更強大、更大型模型的**新興特質**。

| 方法 | Pass@1 準確度 (8 次試驗的平均值) | Pass@1 準確度 (標準差) |
|---|---|---|
| 基準線 | 0.26 | 0.00481 |
| Reflexion | 0.26 | 0.00305 |

**表 4:** 在使用 **Starcoder-beta** [13] 的 HumanEval Python 上，Pass@1 的準確度。

| 模型 | 基準線準確度 | Reflexion 準確度 |
|---|---|---|
| Cot (GT) + text-davinci-003 | 0.60 | 0.77 |
| CoT (GT) + gpt-3.5-turbo | 0.57 | 0.71 |
| CoT (GT) + gpt-4 | 0.68 | 0.80 |
| ReAct + text-davinci-003 | 0.30 | 0.55 |
| ReAct + gpt-3.5-turbo | 0.26 | 0.38 |
| ReAct + gpt-4 | 0.39 | 0.51 |

**表 5:** 在使用各種模型的 100 個 HotPotQA 上，Pass@1 的準確度。

### B 決策

**環境**：你在一間房間的中央。快速環顧四周，你看到一張**床 1**、一張**書桌 2**、一張**書桌 1**、一個**抽屜 6**、一個**抽屜 5**、一個**抽屜 4**、一個**抽屜 3**、一個**抽屜 2**、一個**抽屜 1**、一個**垃圾桶 1**、一個**洗衣籃 1**、一個**保險箱 1**、一個**架子 6**、一個**架子 5**、一個**架子 4**、一個**架子 3**、一個**架子 2** 和一個**架子 1**。

**任務**：用檯燈檢查馬克杯。

#### 試驗 #1

`\(\gimel\)` **思考**：要解決這個任務，我需要找到並拿起一個馬克杯，然後找到並使用一個檯燈。`\(\gimel\)` **思考**：首先我需要找到一個馬克杯。馬克杯更有可能出現在抽屜 (1-6)、書桌 (1-2)、架子 (1-6)、垃圾桶 (1)、洗衣籃 (1) 中。我可以一個一個檢查，從抽屜 1 開始。`\(\gimel\)` **前往 抽屜 1**。**抽屜 1** 是關閉的。

`\(\gimel\)` **打開 抽屜 6**

你打開了**抽屜 6**。**抽屜 6** 是打開的。你在裡面看到一個**鑰匙圈 2**。

`\(\gimel\)` **前往 書桌 1**

在**書桌 1** 上，你看到一張**信用卡 3**、一盞**檯燈 1**、一台**筆記型電腦 2**、一個**馬克杯 1**、一支**鋼筆 1** 和一支**鉛筆 1**。

`\(\gimel\)` **思考**：我現在找到了一個馬克杯 (1)。接下來，我需要拿走它。

`\(\gimel\)` **從 書桌 1 拿起 馬克杯 1**

你從**書桌 1** 上拿起**馬克杯 1**。

`\(\gimel\)` **思考**：我現在拿起了一個馬克杯 (1)。接下來，我需要找到一盞檯燈。檯燈更有可能出現在書桌 (1-2)、邊桌 (1-2)、架子 (1-6)、床 (1)、抽屜 (1-6) 中。我可以一個一個檢查，從書桌 1 開始。

`\(\gimel\)` **前往 書桌 1**

沒有任何事情發生。

`\(\gimel\)` **前往 書桌 2**

在**書桌 2** 上，你看到一個**鬧鐘 1**、一個**碗 1**、一個**馬克杯 3**、一支**鉛筆 3** 和一支**鉛筆 2**。

`\(\gimel\)` **思考**：我現在找到了一盞檯燈 (1)。接下來，我需要使用它。

`\(\gimel\)` **使用 檯燈 1**

沒有任何事情發生。

`\(\gimel\)` **使用 檯燈 1**

沒有任何事情發生。

**狀態**：一般

**反思**：在這個環境中，我的計劃是先找到一個馬克杯，然後找到並使用一盞檯燈。然而，任務要求是用檯燈檢查馬克杯。我應該先尋找檯燈，然後再尋找馬克杯。我注意到檯燈是在**書桌 1** 上找到的。在下一次試驗中，我會前往**書桌 1**，找到檯燈，然後尋找馬克杯並用檯燈檢查它。

#### 試驗 #2

`\(\gimel\)` **前往 書桌 1**

在**書桌 1** 上，你看到一張**信用卡 3**、一盞**檯燈 1**、一台**筆記型電腦 2**、一個**馬克杯 1**、一支**鋼筆 1** 和一支**鉛筆 1**。

`\(\gimel\)` **思考**：要解決這個任務，我需要找到並拿起一個馬克杯，然後找到並使用一個檯燈。

`\(\gimel\)` **從 書桌 1 拿起 馬克杯 1**

你從**書桌 1** 上拿起**馬克杯 1**。

`\(\gimel\)` **思考**：要解決這個任務，我需要找到並拿起一個馬克杯，然後找到並使用一個檯燈。

`\(\gimel\)` **使用 檯燈 1**

你打開了**檯燈 1**。

**狀態**：成功

**圖 5**：[上] 一個 AlfWorld 軌跡，其中代理因為**低效率的規劃**而失敗。在反思中，代理意識到它應該先尋找檯燈再尋找馬克杯，而不是反過來。[下] 代理能夠修正其推理軌跡，並以**簡潔**的方式執行一系列動作。

#### B.1 WebShop 的限制

在 5 中，我們簡要提到 **Reflexion** 在克服需要**極具創造性行為**才能擺脫的局部最小值選擇時會遇到困難。我們在 WebShop [29] 的實驗中觀察到了這一缺點。WebShop 是一個基於網路的問題解決基準測試，它測試代理在給定客戶請求的情況下，瀏覽電子商務網站來尋找和購買產品。我們在 100 個環境中測試了一個 **two-shot ReAct + Reflexion** 代理。然而，在僅僅四次試驗後，我們就終止了運行，因為代理沒有顯示出改進的跡象 6。此外，代理在失敗嘗試後也沒有產生有幫助、直觀的自我反思。我們得出結論，**Reflexion** 無法解決需要大量多樣性和探索的任務。在 AlfWorld 中，代理能夠充分探索新環境，因為**允許的動作**可以在觀察中看到。在 HotPotQA 中，代理面臨與 WebShop 類似的**搜尋查詢**任務，但更成功，因為 Wikipedia 文章的搜尋空間更多樣化，並且需要的搜尋查詢也**不那麼精確**。電子商務搜尋引擎的一個常見問題是**如何正確處理自然語言搜尋解釋中的歧義**。因此，WebShop 呈現了一個需要 **Reflexion** 代理做出非常多樣化和獨特行為的任務。

![](images/276254b497c9ef33aaa80a280ca288f910cf8e69fe53e0be22a7bfe377f74c17.jpg)

**圖 6:** Reflexion 與 ReAct 在 100 個客戶購物請求上的 WebShop 效能比較。**ReAct + Reflexion** 未能顯著超越 **ReAct**。

### C 程式設計

程式設計 LLM 呼叫需要**嚴格的指令**才能只產生**函數主體**，這是由於 LLM 的大量對話訓練所致。下面報告了一些程式設計範例，其中指令以**藍色**和模板突出顯示。完整的實作請參閱 *https://github.com/noahshinn024/reflexion*。

#### C.1 程式設計函數實作範例 (HumanEval Python)

**函數簽名範例**：

```python
1 def minSubArraySum(nums):
2 
3     Given an array of integers nums, find the minimum sum of any
4     non- empty sub- array of nums.
5     Example
6     minSubArraySum[2,3,4,1,2,4]) == 1
7     min_sum = float('inf')     
8     for i in range(len(nums)): 
9        current_sum = 0 
10       for j in range(i, len(nums)): 
11          current_sum += nums[j] 
            if current_sum < min_sum: 
            
               min_sum = current_sum 
       return min_sum
```

# C. Reflexion 代理指令與範例

## C.2 Reflexion Actor 指令

你是一個 **Python 程式寫作助理**。你將收到你之前實作的函數、一系列單元測試結果，以及你對之前實作的自我反思。請僅回應**改進後的函數主體**來應用必要的變更。不要在你的回應中包含函數簽名。你回應的第一行應該有 **4 個空格的縮排**，以便在語法上與使用者提供的簽名相符。

> **範例格式**：(指令) (函數實作) (單元測試回饋) (自我反思) (下一個函數實作的指令)

*完整範例請參考此連結：[https://github.com/noahshinn024/reflexion](https://github.com/noahshinn024/reflexion)*

## C.3 Reflexion 自我反思指令與範例

你是一個 **Python 程式寫作助理**。你將收到你之前實作的函數、一系列單元測試結果，以及你對之前實作的自我反思。請僅回應**改進後的函數主體**來應用必要的變更。不要在你的回應中包含函數簽名。你回應的第一行應該有 **4 個空格的縮排**，以便在語法上與使用者提供的簽名相符。

> **範例格式**：(指令) (函數實作) (單元測試回饋) (自我反思) (下一個函數實作的指令)

## C.4 Reflexion 程式設計無自我反思消融範例

Reflexion 無自我反思消融 Actor 生成遵循以下格式：

> **範例格式**：(指令) (函數實作) (單元測試回饋) (自我反思) (下一個函數實作的指令)

## C.5 Reflexion 程式設計無測試生成消融範例

Reflexion 無測試生成消融 Actor 生成遵循以下格式：

> **範例格式**：(指令) (函數實作) (單元測試回饋) (自我反思) (下一個函數實作的指令)

---

# D. 推理範例

### D.1 完整範例：HotPotQA

此範例展示了 Reflexion + ReAct 代理如何利用自我反思來改進搜尋策略。

* **圖 7**：在同一環境和任務中的兩次 HotPotQA 試驗。Reflexion + ReAct 代理利用自我反思來確定下一次試驗中更好的搜尋方法。
* **問題**：`《長大後》` (Grown-Ups) 的演員，最廣為人知的是在 `《'Allo 'Allo!》` 中扮演哪個角色？

#### **試驗 #1：失敗**

1.  **思考**：需要搜尋**《長大後》和《'Allo 'Allo!》**，找到共同演員，再找出演員在`《'Allo 'Allo!》`中最廣為人知的角色。
2.  **行動**：搜尋`[Grown-Ups]`，找到演員名單，其中包含 **Sam Kelly**。
3.  **行動**：搜尋`["Allo'Allo!"]`，但找不到結果。
4.  **行動**：搜尋`[Gorden Kaye]`，發現他扮演的角色是 `Rene Artois`。
5.  **結論**：完成`[Rene Artois]`。
6.  **觀察**：答案不正確。

#### **反思**

我搜尋了錯誤的節目名稱 `"Allo 'Allo!"`，導致沒有結果。我應該搜尋該節目的主要演員 **Gorden Kaye**，以找出他在節目中最廣為人知的角色。

---

### D.2 另一 HotPotQA 範例

此範例展示了代理如何從錯誤中學習，並改進其對於共同職業的推理。

* **問題**：John Lanchester 和 Alan Dean Foster 有什麼共同的職業？

#### **試驗 #1：失敗**

1.  **思考**：John Lanchester 是小說家、記者和評論家。Alan Dean Foster 是小說家和編劇。
2.  **結論**：完成`[小說家和編劇]`。
3.  **觀察**：答案不正確。

#### **試驗 #2：成功**

1.  **思考**：John Lanchester 是小說家、記者和金融記者。Alan Dean Foster 是小說家和編劇。
2.  **結論**：完成`[小說家]`。
3.  **觀察**：答案正確。

#### **反思**

我對 John Lanchester 和 Alan Dean Foster 共同職業的推理解釋失敗了，因為我錯誤地假設他們都有相同的職業。將來，當嘗試回答這個問題時，我應該更專注於更好地研究這兩位作者各自的背景，以確保我準確地識別他們的職業。

---

### D.3 HotPotQA 思維鏈 (GT) + Reflexion

此範例顯示代理如何透過反思來提供更全面和準確的答案。

* **問題**：在美國獨立戰爭中，為了控制紐約市和新澤西州，於 1776 年 10 月 28 日在紐約州白原市附近進行的一系列戰役是什麼？

#### **試驗 #1：失敗**

1.  **思考**：從情境中，我知道這是白原市戰役。
2.  **結論**：完成`[白原市戰役]`。
3.  **觀察**：答案不正確。

#### **試驗 #2：成功**

1.  **思考**：根據情境，我知道這一系列戰役是紐約和新澤西戰役。
2.  **結論**：完成`[紐約和新澤西戰役]`。
3.  **觀察**：答案正確。

#### **反思**

我可能沒有提供足夠的情境來準確回答問題。問題詢問的是**一系列戰役**，但我只提供了一個戰役的名稱。將來，我會確保提供更多情境，例如戰役活動的名稱，即**紐約和新澤西戰役**，以及它是一系列戰役的事實。

---

### D.4 HotPotQA 記憶 (EPM) 消融提示

此處展示兩種不同的記憶消融實驗。

#### D.4.1 (EPM) 思維鏈\(^+\) Reflexion

此範例顯示代理如何透過反思來糾正對音樂家樂團成員身份的錯誤比較。

* **問題**：Jonny Craig 和 Pete Doherty 誰曾是更多樂團的成員？

#### **試驗 #1：失敗**

1.  **思考**：Jonny Craig 曾是六個樂團的成員。Pete Doherty 曾是七個樂團的成員。
2.  **結論**：完成`[Pete Doherty]`。
3.  **觀察**：答案不正確。

#### **試驗 #2：成功**

1.  **思考**：我將研究這兩位音樂家過去和現在的樂團...發現 Jonny Craig 曾是七個樂團的成員。Pete Doherty 曾是七個樂團的成員。
2.  **結論**：完成`[Jonny Craig]`。
3.  **觀察**：答案正確。

#### **反思**

我對哪位音樂家曾是更多樂團成員的推理失敗了，因為我沒有考慮到 Jonny Craig 在過去曾是更多樂團的成員，儘管 Pete Doherty 目前是更多樂團的成員。將來，我應該專注於研究這兩位音樂家**過去和現在**的樂團。

#### D.4.2 (EPM) 思維鏈 (GT)\(^+\) Reflexion

此範例展示代理如何從混淆學位類別與特定領域的錯誤中學習。

* **情境**：...Hari Bahadur Basnet 擁有**工程學碩士** (M.Sc.) 學位...
* **問題**：Rastriya Janashakti Party 外交關係部門的負責人所擁有的學位，可以縮寫為 MS, M.S. 或 ScM，是在哪個領域？

#### **試驗 #1：失敗**

1.  **思考**：我知道這是**理學碩士學位**，通常授予**科學、工程和醫學領域**。
2.  **結論**：完成`[科學、工程和醫學]`。
3.  **觀察**：答案不正確。

#### **試驗 #2：成功**

1.  **思考**：我知道這個學位是理學碩士學位...然而，問題是詢問研究領域，而不是學位。根據情境，我知道 Hari Bahadur Basnet 擁有**工程學**碩士 (M.Sc.) 學位。
2.  **結論**：完成`[工程學]`。
3.  **觀察**：答案正確。

#### **反思**

我誤解了問題。我以為問題是在詢問一個學位的類別，而不是一個特定的學位。我從我的錯誤中吸取了教訓，並在第二次試驗中更專注於問題，以理解它是在詢問一個**特定的研究領域**。