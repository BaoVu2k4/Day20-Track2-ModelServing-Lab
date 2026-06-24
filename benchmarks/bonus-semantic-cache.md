# Bonus C8 — Semantic Cache (offline threshold sweep)

Command:

```powershell
python BONUS-llama-cpp-optimization\semantic-cache-demo.py --offline --sweep
```

| Similarity threshold | Cache hits | Hit rate |
|---:|---:|---:|
| 0.70 | 3 / 8 | 38% |
| 0.80 | 3 / 8 | 38% |
| 0.85 | 3 / 8 | 38% |
| 0.90 | 3 / 8 | 38% |
| 0.95 | 3 / 8 | 38% |

At threshold 0.80, three paraphrased requests were served from cache, avoiding three simulated 250 ms LLM calls (about 750 ms total). This offline demo uses lexical bag-of-words embeddings, so the threshold sweep is intentionally coarse; a real embedding model would distinguish semantically related but lexically different prompts more effectively. Lowering a semantic-cache threshold may improve hit rate but raises the risk of returning a wrong or stale answer.
