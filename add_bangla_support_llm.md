# Step-by-Step Guide: Adding Bangla (বাংলা) Support to an Open-Source LLM on Hugging Face

Target scenario  
• **Base model** – any permissively-licensed open-source LLM checkpoint on Hugging Face (e.g. *BLOOM-560M*, *Llama-2-7B-hf*, *Falcon-7B-instruct*).  
• **Goal** – extend it so the model can understand, generate, and evaluate Bangla text while preserving performance in previously supported languages.

The workflow is organised into eight phases:

0. Project scoping & hardware planning  
1. Bangla corpus acquisition  
2. Data cleaning & preprocessing  
3. Tokenizer adaptation  
4. Continued pre-training / instruction tuning  
5. Evaluation & diagnostics  
6. Packaging & model card creation  
7. Publishing on Hugging Face Hub  
8. Post-release maintenance

Each phase is expanded below into concrete, reproducible steps with references to standard open-source tools.

---
## 0  Project scoping & hardware planning

0-a **Define use-case** – translation, chat, code-mixed, formal writing, etc.  
0-b **Pick a base checkpoint** matching licence, GPU/TPU budget, latency target.  
0-c **Estimate tokens needed** – rule-of-thumb: **20–50 B Bangla tokens** for strong results; 2–10 B still valuable.  
0-d **Hardware** – ~2 000 GPU-hours per 7 B model to ingest 10 B tokens at 2 tok/s on A100 80 GB; scale accordingly.  
0-e Budget time for tokenizer surgery, training experiments, evaluation, and red-team review.

---
## 1  Bangla corpus acquisition

1-a **Public sources**  
• Wikimedia dumps (Bangla Wikipedia, Wikisource, Wiktionary)  
• CommonCrawl extractions (OSCAR, CCNet, C4)  
• OpenSubtitles, PMIndia, ILCI, FLORES-200 (parallel corpora)  
• Bangla news datasets (bdnews24, Prothom Alo) if licence permits  
• Bangla literature in public domain (Project Gutenberg)

1-b **Private/curated sources** (seek permission): OCR'd books, social-media dumps, domain-specific data (finance, legal, medical).  
1-c Aggregate raw text; store each document with metadata: *source*, *licence*, *timestamp*, *language-probability score*.

---
## 2  Data cleaning & preprocessing

2-a **Language identification** – fastText `lid.176.bin` or CLD3, keep ≥ 90 % Bangla confidence.  
2-b **Deduplication**  
 • Near-duplicate: MinHash/LSH (datasketch)  
 • Exact duplicate: SHA-256 hashing of lines/paragraphs  
2-c **Normalize text**  
 • Unicode NFC, convert Arabic digits to Bengali numerals if desired  
 • Remove invisible control chars, HTML entities, boilerplate  
2-d **Sentence/paragraph segmentation** – Indic NLP library or spaCy `bn` pipeline (≤ 2 048 chars).  
2-e **Quality filtering** – heuristics: token-to-unique-token ratio, average word length, profanity filter, perplexity filtering with a small Bangla LM.  
2-f **Formatting for training** – store as JSONL:

```json
{"text": "…Bangla text…", "meta": {"source": "wiki", "quality": 0.93}}
```

---
## 3  Tokenizer adaptation

3-a **Inspect current tokenizer** (e.g. Llama-2 uses SentencePiece with 32 k merges).  
3-b **Measure Bangla coverage**  
 • % of Bangla characters appearing in single-char tokens  
 • Average tokens per Bangla word (goal < 2.5)  
3-c **Choose strategy**  
 Option 1: *Add tokens* (fast, backward-compatible)  
 Option 2: *Train new SentencePiece/BPE* (requires embedding surgery, may degrade other languages)  
3-d **Implement Option 1 (recommended)**  
 • Extract top-N frequent sub-words/words missing from vocab (e.g. 10 k)  
 • Append them to `vocab.json` or `tokenizer.model`  
 • Resize model embeddings:

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
model = AutoModelForCausalLM.from_pretrained("base")
tokenizer = AutoTokenizer.from_pretrained("base", use_fast=False)
# load updated tokenizer containing new Bangla tokens
model.resize_token_embeddings(len(tokenizer))
model.save_pretrained("bn-adapted")
```

3-e **Verify** round-trip encoding/decoding for new tokens.

---
## 4  Continued pre-training / instruction tuning

4-a **Build training dataset** – mix ratio Bangla 30–50 %, original corpus 50–70 % (avoid catastrophic forgetting).  
4-b **Hyper-parameters** (example for 7 B model):  
• Context length 2 048  
• Batch size ≈ 1 024 tokens/GPU (use gradient accumulation for global 4 M tokens)  
• LR schedule – cosine, peak 2 × 10⁻⁵, warm-up 2 % steps  
• Continue for 1–3 epochs over mixed corpus  
4-c **Infrastructure** – Hugging Face `trl` or `transformers` Trainer + DeepSpeed ZeRO-3 / FSDP.  
4-d **Checkpoints & safety nets** – save every 500 M tokens; run eval; keep best perplexity.  
4-e **Optional instruction tuning** – collect Bangla instruction-response pairs (translate ShareGPT/UltraChat or craft native data) and fine-tune with LoRA (rank 256) for 3 epochs at 2 × 10⁻⁴ LR.

---
## 5  Evaluation & diagnostics

5-a **Intrinsic metrics**  
• Perplexity on held-out Bangla validation set  
• Tokenization efficiency (tokens/word)  
• Embedding-norm drift (stability)  
5-b **Downstream tasks**  
• Bangla reasoning: xWinograd, XCOPA bn, IndicXNLI  
• Generative quality: BLEU/chrF on WMT22 bn-en translation, ROUGE-L on Bengali summarisation (BUET dataset)  
5-c **Human evaluation** – native speakers grade 200 prompts (coherence, grammar, culture, toxicity) using GEMBA or MT-MQM rubric.  
5-d **Safety checks** – HateCheck-Indic, disallowed-content prompts, bias tests on gendered Bangla names & religious references.

---
## 6  Packaging & model card creation

6-a Select best checkpoint; merge LoRA into base if used (`peft.merge_and_unload`).  
6-b Push artefacts via `huggingface-cli login` then `model.save_pretrained(<repo>)`:
• `config.json`, `pytorch_model-*.bin`, `tokenizer.json|model`, `generation_config.json`  
6-c Write **README.md / Model Card**:  
• Training procedure, hardware, data sources & licences  
• Intended use & limitations  
• Evaluation results with plots  
• Ethical considerations & citation policy  
6-d Add Hub tags: `llm`, `bangla`, `multilingual`, `causal-lm`, `transformers`, `apache-2.0`.

---
## 7  Publishing on Hugging Face Hub

7-a Create a **Space** with Gradio chat demo referencing the checkpoint.  
7-b Configure repository: enable Discussions, PR mode; link evaluation datasets.  
7-c Announce release on social media, Hugging Face forum, Bengali NLP communities.

---
## 8  Post-release maintenance

8-a Monitor issues: hallucinations, safety breaches.  
8-b Plan v2 with more Bangla tokens and richer instruction data.  
8-c Periodically update tokenizer with emerging slang/loan words.  
8-d Encourage community fine-tunes and contributions via PRs.

---
## Key tools & scripts (ready-made)

• `oscar-dedupe`, `clean-cc` – web-corpus cleaning  
• `indic-nlp-library` – Bangla normalisation & tokenisation  
• Hugging Face **tokenizers** – adding new tokens  
• `transformers` Trainer & `trl` SFTTrainer  
• `evaluation-suite` by BigScience for multilingual metrics  
• `gradio-templates/LLM-chat` – Space UI boilerplate

---
## Checklist before hitting "Upload"

- [ ] Data-source licences audited & cited  
- [ ] Tokenizer passes encode-decode idempotency test  
- [ ] Model converged (≤ 2 % PPL drop vs. no-Bangla baseline)  
- [ ] Evaluation benchmarks show ≥ +10 BLEU (or similar gains)  
- [ ] Human reviewers signed off on quality & safety  
- [ ] Model card complies with HF community guidelines

---
## End-to-end script outline (pseudo-bash)

```bash
# 1. Gather & clean
python gather_bangla_corpus.py --out raw_bangla.txt
python clean_corpus.py raw_bangla.txt > bangla_clean.jsonl

# 2. Tokenizer surgery
python add_tokens.py --base-model base --new-text bangla_clean.jsonl --out tokenizer_bn

# 3. Training
accelerate launch train.py \
  --model_name_or_path base \
  --tokenizer_name tokenizer_bn \
  --train_data mix_corpus.jsonl \
  --output_dir bn-trained \
  --num_train_epochs 2 --per_device_train_batch_size 4 ...

# 4. Evaluation
python evaluate.py --model bn-trained --tasks indicxnli,xcopa ...

# 5. Push to HF
huggingface-cli upload bn-trained maz/bn-llm --commit-message "Bangla v1"
```

Follow these steps to build a reproducible, community-friendly Bangla-enabled version of your chosen open-source LLM and publish it on the Hugging Face Hub. 