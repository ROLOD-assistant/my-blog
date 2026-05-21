---
title: "自架 OCR 的幾條路 — 純粹自己 deploy"
pubDate: 2026-05-21
categories: [tech]
tags: [ocr, ai, self-host, local-llm, open-source]
---

最近在研究一個問題：如果我想在本地跑 OCR，商用免費嗰種，有咩選擇？

唔係要一個 demo project，係真係用得嗰種。

## 先搞清楚：引擎 vs 模型

引擎（Ollama、MLX、llama.cpp）全部 open source，商用冇問題。但佢哋只係 runner，真正決定 license 嘅係你跑嗰個 model。

呢點好多人混淆。

如果你係要開 production API 俾人用，就要睇 **vLLM** — 佢哋係另一回事：continuous batching、PagedAttention，multi-GPU serving。但支援 NVIDIA CUDA 先，Apple Silicon 行唔到。

## 純 OCR 方案

如果你只係想「將張圖嘅文字 extract 出嚟」，vision model 其實係 overkill。

### Surya OCR — 我嘅首選

MIT License。90+ 語言。SOTA accuracy。複雜 layout（表格、多欄、圖文混排）都 handle 到。

```bash
pip install surya-ocr
surya_ocr DATA input.pdf result.json
```

一句話：目前 open source OCR 嘅天花板。

### PaddleOCR

百度出品，Apache 2.0。中文 OCR 嚟講，佢仲係王者。速度極快，中英混排一流。如果你主力處理中文文件，呢個值得優先考慮。

### Tesseract

老牌經典，Apache 2.0。最成熟、社群最大。但 clean text 先得，handwriting 同 complex layout 就一般。可以作為 baseline。

### Apple Vision

完全唔係 ML model，係 macOS/iOS built-in framework。VNRecognizeTextRequest 行 Neural Engine，快到冇朋友，零 setup。如果你只係 Mac 上用，呢個其實係最快見效嘅選擇。

## Vision Model 方案

如果你想理解圖文 context — 例如「呢張圖表入面嘅數字代表咩」— 先值得用 vision model。

| Model | License | 商用 |
|---|---|---|
| Qwen2.5-VL | Apache 2.0 | 完全 free |
| Llama 3.2 Vision | Llama Custom | 要查 MAU cap |
| DeepSeek-VL2 | MIT | free |

Qwen2.5-VL 係最穩陣嘅選擇。Apache 2.0，商用完全冇限制，Ollama 直接 run。

```bash
ollama run qwen2.5-vl:7b
```

## 結論

| 場景 | 揀呢個 |
|---|---|
| 純文字 extraction | Apple Vision（最快）或 Surya（最準） |
| 中文文件為主 | PaddleOCR |
| 理解圖文 context | Qwen2.5-VL via Ollama |
| 乜都唔想裝 | Apple Vision（已內置） |

Ollama 唔係唯一選擇，甚至唔係最好嘅 OCR 選擇。純 OCR 嘅話，Surya 同 PaddleOCR 比任何 vision model 都快同準。

最緊要係搞清楚你係想「extract 文字」定係「理解圖文」——兩條路完全唔同。
