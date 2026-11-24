# 📈 tw-stock-sentiment-llm

[![Python](https://img.shields.io/badge/Python-3.10-blue.svg)](https://www.python.org/)
[![Base Model](https://img.shields.io/badge/Base%20Model-Qwen%202.5-purple.svg)](https://huggingface.co/Qwen)
[![Labeling](https://img.shields.io/badge/Labeling-Gemini%20API-orange.svg)](https://ai.google.dev/)
[![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Model-zeroone--000%2Ftw--stock--qwen2.5--lora-yellow.svg)](https://huggingface.co/zeroone-000/tw-stock-qwen2.5-lora)

針對台灣股市重點電子與半導體族群打造的 **End-to-End 財經新聞情緒分析管線**。本專案結合 **Gemini API 偽標籤標註（Pseudo-Labeling）**、**人機協作錯題本（Error Book）資料清洗**，以及 **Qwen 2.5 模型微調（Fine-Tuning）與推論**，建立可持續迭代的台股新聞情緒評分模型。

---

## 🎯 追蹤標的（台股 10 檔重點股）

| 股票代號 | 公司名稱 | 股票代號 | 公司名稱 |
| :---: | :--- | :---: | :--- |
| **2330** | 台積電 | **3711** | 日月光投控 |
| **2317** | 鴻海 | **2327** | 國巨 |
| **2454** | 聯發科 | **2408** | 南亞科 |
| **2308** | 台達電 | **3481** | 群創 |
| **2303** | 聯電 | **6182** | 合晶 |

---

## 📂 專案結構

```text
tw-stock-sentiment-llm/
├── data_pipeline/               # 資料蒐集、清洗、轉換與 Gemini 標註
│   ├── catch_data.py            # 新聞資料擷取
│   ├── url_transfer.py          # 新聞連結處理與轉換
│   ├── data_washing.py          # 文本清洗與雜訊過濾
│   ├── data_transfer.py         # 訓練／標註資料格式轉換
│   ├── translate_dataset.py     # 資料集翻譯與語料處理
│   ├── local_dataset.py         # 本地資料集管理與載入
│   ├── gemini_labeler.ipynb     # 呼叫 Gemini API 進行自動化情緒標註
│   ├── output_transfer.py       # 標註結果格式轉換
│   ├── count_sensitive.py       # 情緒分佈與敏感度統計
│   ├── score_avg.py             # 情緒分數彙整與平均計算
│   └── test.py                  # 功能測試腳本
├── model/                       # Qwen 2.5 微調與推論
│   ├── qwen2_5_finetune.ipynb   # Qwen 2.5 模型微調 Notebook
│   └── qwen_inference.ipynb     # 模型推論與表現評估 Notebook
├── .gitignore
└── README.md
```

---

## 🔄 人機協作迭代流程（SOP）

為確保模型評分貼近台股市場實務，本專案採用「模型對比 → 異常排查 → 黃金資料集擴充」的閉環流程：

1. **每日推論與對比**：使用 Gemini API 與當前版本的 Qwen 模型，對最新抓取的新聞同步進行情緒評分。
2. **極端值排查**：透過統計與比對腳本篩選出兩模型評分誤差 **大於 3 分** 的異常新聞。
3. **人工定調清洗**：針對高誤差案例進行人工覆核，確認正確的情緒分數與判斷依據。
4. **擴充黃金資料集**：將確認後的「新聞內容 + 最終正確分數」以標準 JSON 格式寫入資料集。
5. **Prompt 短期優化（Error Book）**：將代表性的誤判模式整理為錯題本，納入 `SYSTEM_PROMPT` 作為 Few-Shot 教材。
6. **模型長期微調**：累積超過 **2,000 筆**高品質標註資料後，進入 `model/` 執行 Qwen 2.5 微調。

### Error Book 資料格式
```json
{
  "input": "新聞標題與內文",
  "correct_score": 3,
  "reason": "人工判定該分數的核心財經依據"
}
```

---

## 🚀 快速開始

### 階段一：資料管線與 Gemini 標註（本機環境）

1. 建立並啟動 Python 3.10 環境：
   ```bash
   conda create -n gemini-labeler python=3.10 -y
   conda activate gemini-labeler
   ```
2. 安裝必要套件：
   ```bash
   pip install google-generativeai
   ```
3. 依序執行 `data_pipeline/` 內的爬蟲與清洗腳本，並於 `gemini_labeler.ipynb` 設定你的 Gemini API Key 開始標註。

---

### 階段二：Qwen 2.5 微調與推論（Google Colab / GPU 環境）

1. **模型微調**：於 Colab 開啟 `model/qwen2_5_finetune.ipynb`，掛載整理完成的黃金資料集即可進行微調訓練。
2. **推論與評估**：於 Colab 開啟 `model/qwen_inference.ipynb`，或直接從 Hugging Face Hub 載入已微調的 LoRA 權重：

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel

base_model_id = "Qwen/Qwen2.5-14B-Instruct"  # 請依實際微調之基礎模型版本調整
adapter_id = "zeroone-000/tw-stock-qwen2.5-lora"

tokenizer = AutoTokenizer.from_pretrained(base_model_id)
base_model = AutoModelForCausalLM.from_pretrained(base_model_id, device_map="auto")
model = PeftModel.from_pretrained(base_model, adapter_id)
```

---

## 🤗 模型權重

微調後的 LoRA 權重託管於 Hugging Face Hub（未直接存放於本 GitHub 倉庫）：
* **Hugging Face Repo**：[zeroone-000/tw-stock-qwen2.5-lora](https://huggingface.co/zeroone-000/tw-stock-qwen2.5-lora)
