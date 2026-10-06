# CPU-only thread sweep (-ngl 0), tg128, 5 reps, raw llama-bench output
# confounder: macOS mediaanalysisd was using ~100% of one core during this run

| model                          |       size |     params | backend    | threads |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | --------------: | -------------------: |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       4 |           tg128 |         38.82 ± 0.20 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       6 |           tg128 |         46.02 ± 0.25 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       8 |           tg128 |         47.46 ± 0.21 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |      10 |           tg128 |         32.70 ± 1.90 |
build: 9d77fa172 (10488)
