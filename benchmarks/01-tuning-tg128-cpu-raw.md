# CPU-only thread sweep (ngl=0), tg128, 2 reps, raw llama-bench output

```
ggml_metal_device_init: tensor API disabled for pre-M5 and pre-A19 devices
ggml_metal_library_init: using embedded metal library
ggml_metal_library_init: loaded in 0.014 sec
ggml_metal_rsets_init: creating a residency set collection (keep_alive = 180 s)
ggml_metal_device_init: GPU name:   MTL0 (Apple M4)
ggml_metal_device_init: GPU family: MTLGPUFamilyApple9  (1009)
ggml_metal_device_init: GPU family: MTLGPUFamilyCommon3 (3003)
ggml_metal_device_init: GPU family: MTLGPUFamilyMetal4  (5002)
ggml_metal_device_init: simdgroup reduction   = true
ggml_metal_device_init: simdgroup matrix mul. = true
ggml_metal_device_init: has unified memory    = true
ggml_metal_device_init: has bfloat            = true
ggml_metal_device_init: has tensor            = false
ggml_metal_device_init: use residency sets    = true
ggml_metal_device_init: use shared buffers    = true
ggml_metal_device_init: recommendedMaxWorkingSetSize  = 12713.12 MB
| model                          |       size |     params | backend    | threads |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | --------------: | -------------------: |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       1 |           tg128 |         20.99 ± 0.01 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       2 |           tg128 |         34.22 ± 0.09 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       3 |           tg128 |         37.84 ± 0.08 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       4 |           tg128 |         38.59 ± 0.13 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       5 |           tg128 |         42.51 ± 0.36 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       6 |           tg128 |         46.28 ± 0.10 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |       8 |           tg128 |         46.89 ± 0.45 |
| gemma4 E2B Q4_K - Medium       |   2.95 GiB |     4.65 B | MTL,BLAS   |      10 |           tg128 |         29.20 ± 6.20 |

build: 9d77fa172 (10488)
```
