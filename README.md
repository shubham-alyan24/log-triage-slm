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

Done: the dataset, the frozen evaluation set, the untuned baseline, the fine-tune, the quantisation sweep (five levels, all scored on the frozen set) and the serving layer (a FastAPI service in front of Ollama, also packaged with Docker). The weights for all five levels are uploaded to Hugging Face; that repository goes public with the final results.

Next: the speed and memory benchmarks for each level, and the accuracy-against-cost figure.

## Results

Frozen evaluation set, 329 examples. Every model is served through Ollama 0.34.2, same machine, same prompt, same settings.

### Does fine-tuning work?

Both models at Q8_0:

| Metric | Untuned | Fine-tuned | Always most-common answer |
|---|---|---|---|
| Parsed / schema-valid | 99.1% | 99.7% | - |
| Strict JSON | 0.0% | 99.7% | - |
| Action accuracy | 83.3% | 97.0% | 85.7% |
| Action macro-F1 | 24.3% | 88.2% | 23.1% |
| Severity accuracy | 66.6% | 95.7% | 84.5% |
| Service accuracy | 20.1% | 79.9% | 23.1% |
| Identifier F1 | 29.0% | 92.2% | - |
| Identifiers exactly right | 2.4% | 81.8% | - |
| check_network found | 0/16 | 16/16 | 0/16 |
| escalate_oncall found | 0/6 | 6/6 | 0/6 |

The column that matters is the last one. 86% of the evaluation set is `no_action`, so a model that always answers `no_action` already scores 85.7%, and the untuned model does worse than that. The fine-tuned model beats it on every field, which is the only comparison worth making here.

Ollama updated itself partway through the project (0.32.6 to 0.34.2), so both models were re-scored on the new version. The same inputs changed 262 of 329 untuned outputs but only 13 of 329 fine-tuned ones; no conclusion changed.

Broken down by whether the log pattern appeared in training (fine-tuned, Q8_0):

| | seen patterns (185) | unseen patterns (144) |
|---|---|---|
| Action accuracy | 98.4% | 95.1% |
| Severity accuracy | 97.8% | 93.1% |
| Service accuracy | 95.7% | 59.7% |
| Identifier F1 | 96.8% | 87.9% |

Unseen patterns are the real test. Service falls off a cliff there because the component name never appears in the input text, so for a new pattern the model has nothing to copy and can only guess from wording.

### How much survives compression?

The fine-tuned model at five precision levels:

| Level | Size | Real bits/weight | Action acc | Action macro-F1 | Severity | Service | Identifier F1 | escalate_oncall found (precision) |
|---|---|---|---|---|---|---|---|---|
| F16 | 6.16 GB | 16.02 | 97.0% | 88.2% | 95.7% | 79.9% | 91.9% | 6/6 (0.50) |
| Q8_0 | 3.28 GB | 8.52 | 97.0% | 88.2% | 95.7% | 79.9% | 92.2% | 6/6 (0.50) |
| Q5_K_M | 2.21 GB | 5.76 | 97.3% | 89.5% | 95.7% | 79.9% | 92.0% | 6/6 (0.50) |
| Q4_K_M | 1.92 GB | 4.98 | 96.0% | 80.5% | 95.7% | 79.3% | 91.3% | 3/6 (0.38) |
| Q3_K_M | 1.57 GB | 4.09 | 93.9% | 77.7% | 93.6% | 80.9% | 89.2% | 4/6 (0.27) |

- F16, Q8_0 and Q5_K_M are the same within noise. Q5_K_M is 64% smaller than F16 with no measurable loss. Its slightly higher macro-F1 is one or two lines, so it is not better, just not worse.
- Q4_K_M is where the first damage shows, and it lands on the most critical class: only 3 of 6 `escalate_oncall` lines found. Six lines is too few to prove anything, so this is a warning sign rather than a result.
- Q3_K_M declines across the board. It makes 20 action errors against F16's 10, and all ten extra ones are on BGL logs; severity and identifiers drop too. It is still far above the untuned model.
- Lower levels raise more false alarms: `escalate_oncall` precision falls from 0.50 to 0.27.
- "Real bits/weight" is higher than the name suggests because the shared input/output embedding is kept in a higher 6-bit format (Q6_K) in every K level, so "Q4_K_M" is really about 5 bits per weight.

## How it works

Data comes from LogHub-2k (HDFS, BGL, OpenStack, Spark): 8,000 lines over 213 patterns. The model sees the message text only. The level and component headers are stripped so they cannot leak into the answer.

Service, severity and identifiers come from LogHub's own annotations. The recommended action is distilled from a larger teacher model, one label per (pattern, severity) group.

The split is by pattern with a fixed seed, so held-out patterns never appear in training, and the evaluation set was frozen and hashed before any training started. For the training set I dropped exact duplicates and capped each `no_action` pattern at 20 examples, since one pattern alone had over 900 lines and would otherwise swamp the rare actions. That turns 5,451 lines into 1,057 training examples.

The model is SmolLM3-3B, fine-tuned with QLoRA on a free Kaggle T4: 4-bit base, LoRA rank 16 on all linear layers, learning rate 2e-4, effective batch 16, three epochs, about half an hour. Loss is computed on the answer tokens only. I picked the checkpoint by dev loss and nothing else; epoch 1 won.

The adapter is merged into the full-precision model and converted to GGUF with a pinned llama.cpp build. F16 and Q8_0 come straight from the converter; the K levels are made from F16 with `llama-quantize`, without an importance matrix.

For scoring, each level is served through Ollama in raw mode at temperature 0 with a fixed seed. Every model ever scored uses the same prompt, settings and machine. Output that will not parse counts as wrong on every field. Confidence intervals are bootstrapped.

The API (`src/serve.py`) sends exactly the prompt, settings and parser that scoring used, so it answers the way the scored model answered. The benchmarks run on the same Ollama on the same machine, so both axes of the accuracy-against-cost figure come from one runtime. The original plan was to serve through llama.cpp directly; I switched to Ollama so the accuracy and cost numbers can't disagree simply because they were measured with different programs.

## Layout

```
src/                   dataset builder, teacher labelling, splitting, freezing, training-set builder,
                       runner (run_ollama.py), scorer (score.py), API (serve.py)
notebooks/             Kaggle notebooks: fine-tuning, quantisation, Hugging Face upload
data/                  splits, frozen evaluation set and its hash
results/               predictions and scores for every model, quantisation details
models/                one Ollama Modelfile per level (weights are not in git)
Dockerfile, docker-compose.yml, .dockerignore    the containerised API
requirements-serve.txt                           what the API needs, nothing else
```

## Running it

Put the GGUF files in `models/` and import a level into Ollama, for example Q5_K_M:

```
ollama create smollm3-ft-q5km -f models/Modelfile.q5km
```

Score a model on the frozen set:

```
python src/run_ollama.py --model smollm3-ft-q5km --tag ft_q5km
python src/score.py results/preds_eval_ft_q5km.jsonl
```

Run the API (Ollama must be running), then open http://localhost:8000/docs:

```
pip install -r requirements-serve.txt
uvicorn serve:app --app-dir src --port 8000
```

`POST /triage` takes `{"log": "<one log line>"}` and returns the structured record, the model's raw output and its timings. `GET /health` reports the Ollama version and whether the model is loaded. The model can be changed with the `MODEL` environment variable.

Or run everything in Docker (Ollama 0.34.2 and the API), then open http://localhost:8080:

```
docker compose up -d --build
docker compose exec ollama ollama create smollm3-ft-q5km -f /models/Modelfile.q5km
```

The model import is needed once; it stays in a Docker volume after `docker compose down`. The Docker setup is for demonstration. Every number in this README comes from the native setup.

## Limits worth knowing

Service is a memorisation task. The component name is not in the input, so 59.7% on unseen patterns is roughly the ceiling of the setup, not something more training fixes.

`escalate_oncall` has six lines in the evaluation set and only 11 distinct lines in training. All six were found at F16, Q8_0 and Q5_K_M, but precision is 0.50 and falls further at lower levels. That class is anecdotal and should be read that way. Every `check_network` training example comes from HDFS, so that action has never been learned from the other systems.

LogHub-2k limits how large the evaluation set can be. It has many duplicate lines, and 120 of the 213 patterns contain only one distinct message.

One evaluation line is unscorable because its correct answer is longer than the output limit. It counts as wrong for every model, including the baseline.

The action labels come from a teacher model, and the evaluation labels were reviewed by an AI assistant, not checked by hand. The ceiling is that model's judgement rather than an expert's. Serving runs on CPU, which caps throughput, and every timing comes from one fixed machine, so the numbers are comparable to each other and to nothing else.

## Next

Benchmark every level on the same machine: time to first token, median and 95th-percentile latency, tokens per second and peak memory, then the same under concurrent load. The end product is a figure plotting accuracy against cost, plus a live demo.

Beyond the project itself, a few things would make the result stronger. Training on more log sources would show whether accuracy on unseen formats actually improves instead of only being measured. Having an expert relabel a sample of the actions would show how far the teacher model's labels sit from real judgement. And comparing against something cheap and non-neural, like nearest-neighbour pattern matching, would show how much the language model is really adding.
