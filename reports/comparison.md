# Memory vs No-Memory Comparison

| Metric | Memory-enabled | No-memory | Delta |
| --- | ---: | ---: | ---: |
| Evidence hit rate | 90.9% | 18.2% | +72.7% |
| Passed cases | 10/11 | 2/11 | +8 |
| Avg retrieval latency (ms) | 949.4 | 0.1 | +949.3 |
| Avg token reduction | 15.6% | 81.8% | -66.2% |

## Interpretation

Short-term cases can pass without durable memory because their evidence is still in the current thread. Cross-session, episodic and semantic cases should fail in the no-memory baseline and recover when memory retrieval is enabled.

A no-memory baseline may show near-100 percent token reduction simply because it retrieves nothing. Treat token reduction as useful only together with evidence hit rate; dropping all context is cheap but incorrect.