# Log-Triage SLM

A small language model fine-tuned to read a raw log line and return a structured record.

Input:

```
BLOCK* NameSystem.addStoredBlock: blockMap updated: 10.251.74.79:50010 is added to blk_-2827716238972737794 size 67108864
```

Output:

```json
{"service": "dfs.DataNode$DataXceiver", "severity": "INFO", "identifiers": ["-2827716238972737794"], "recommended_action": "no_action"}
```

The question I am actually trying to answer is not whether a 3B model can do this. It is how much accuracy survives when you compress the model to run cheaply, and what each compression level buys you in memory, latency and throughput on ordinary hardware. Measuring that trade-off is the point of the project.

Shubham Alyan, B.Tech Computer Engineering, NIT Kurukshetra.

## Where it stands

The dataset, the frozen evaluation set, the untuned baseline and the fine-tune are done, and the fine-tuned model has been scored at Q8_0. The quantisation sweep, the serving layer and the benchmarks are not built yet.

## Results

Frozen evaluation set, 329 examples. Both models served at Q8_0 through Ollama, same machine, same prompt, same settings.

| Metric | Untuned | Fine-tuned | Always most-common answer |
|---|---|---|---|
| Parsed / schema-valid | 99.7% | 99.7% | - |
| Strict JSON | 0.0% | 99.7% | - |
| Action accuracy | 82.7% | 97.0% | 85.7% |
| Action macro-F1 | 24.1% | 88.2% | 23.1% |
| Severity accuracy | 66.6% | 94.8% | 84.5% |
| Service accuracy | 22.2% | 79.9% | 23.1% |
| Identifier F1 | 15.8% | 91.2% | - |
| Identifiers exactly right | 4.6% | 80.5% | - |
| check_network found | 0/16 | 16/16 | 0/16 |
| escalate_oncall found | 0/6 | 6/6 | 0/6 |

The column that matters is the last one. 86% of the evaluation set is `no_action`, so a model that always answers `no_action` already scores 85.7%, and the untuned model does worse than that. The fine-tuned model beats it on every field, which is the only comparison worth making here.

Broken down by whether the log pattern appeared in training:

| | seen patterns (185) | unseen patterns (144) |
|---|---|---|
| Action accuracy | 98.4% | 95.1% |
| Severity accuracy | 96.2% | 93.1% |
| Service accuracy | 95.7% | 59.7% |
| Identifier F1 | 94.6% | 88.1% |

Unseen patterns are the real test. Service falls off a cliff there because the component name never appears in the input text, so for a new pattern the model has nothing to copy and can only guess from wording.

## How it works

Data comes from LogHub-2k (HDFS, BGL, OpenStack, Spark): 8,000 lines over 213 patterns. The model sees the message text only. The level and component headers are stripped so they cannot leak into the answer.

Service, severity and identifiers come from LogHub's own annotations. The recommended action is distilled from a larger teacher model, one label per (pattern, severity) group.

The split is by pattern with a fixed seed, so held-out patterns never appear in training, and the evaluation set was frozen and hashed before any training started. For the training set I dropped exact duplicates and capped each `no_action` pattern at 20 examples, since one pattern alone had over 900 lines and would otherwise swamp the rare actions. That turns 5,451 lines into 1,057 training examples.

The model is SmolLM3-3B, fine-tuned with QLoRA on a free Kaggle T4: 4-bit base, LoRA rank 16 on all linear layers, learning rate 2e-4, effective batch 16, three epochs, about half an hour. Loss is computed on the answer tokens only. I picked the checkpoint by dev loss and nothing else; epoch 1 won.

For scoring, the adapter is merged, converted to GGUF with a pinned llama.cpp build, and served through Ollama in raw mode at temperature 0 with a fixed seed. Every model ever scored uses the same prompt, settings and machine. Output that will not parse counts as wrong on every field. Confidence intervals are bootstrapped.

## Layout

```
src/        dataset builder, teacher labelling, splitting, freezing, training-set builder, runner, scorer
notebooks/  Kaggle fine-tuning notebook
data/       splits, frozen evaluation set and its hash
results/    predictions and scores for every model
models/     Ollama Modelfile (weights are published separately, not in git)
```

With Ollama installed and the model imported:

```
python src/run_ollama.py --model <ollama model name> --tag <short name>
python src/score.py results/preds_eval_<tag>.jsonl
```

## Limits worth knowing

Service is a memorisation task. The component name is not in the input, so 59.7% on unseen patterns is roughly the ceiling of the setup, not something more training fixes.

`escalate_oncall` has six lines in the evaluation set. All six were found, but precision is 0.50. That class is anecdotal and should be read that way.

LogHub-2k limits how large the evaluation set can be. It has many duplicate lines, and 120 of the 213 patterns contain only one distinct message.

One evaluation line is unscorable because its correct answer is longer than the output limit. It counts as wrong for every model, including the baseline.

The action labels come from a teacher model, so the ceiling is that model's judgement rather than an expert's. Serving runs on CPU, which caps throughput, and every timing comes from one fixed machine, so the numbers are comparable to each other and to nothing else.

## Next

Quantise to F16, Q8_0, Q5_K_M, Q4_K_M and Q3_K_M, score each level on the frozen set and publish the weights. Then put llama.cpp behind a FastAPI service in Docker and benchmark it: time to first token, median and 95th-percentile latency, tokens per second, peak memory, and the same under concurrent load. The end product is a figure plotting accuracy against cost, plus a live demo.

Beyond the project itself, a few things would make the result stronger. Training on more log sources would show whether accuracy on unseen formats actually improves instead of only being measured. Having an expert relabel a sample of the actions would show how far the teacher model's labels sit from real judgement. And comparing against something cheap and non-neural, like nearest-neighbour pattern matching, would show how much the language model is really adding.