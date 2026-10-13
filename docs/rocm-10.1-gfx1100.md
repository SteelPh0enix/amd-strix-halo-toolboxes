# Desktop ROCm toolbox (`rocm-10.1-gfx1100`)

Baseline `llama.cpp` from `ggml-org/llama.cpp:master` against the ROCm 10.1 Core SDK, built for
desktop RDNA3 cards (`gfx1100` - Radeon RX 7900 XT / XTX / GRE). None of the Strix Halo patches or
iGPU workarounds are applied: no `llama-cpp-25992` host-buffer patch, no `llama-grammar` patch, no
custom ROCm runtime.

The image is not published to Docker Hub and is not part of the automatic rebuild matrix. Build it
locally, or dispatch the `Build & Publish` workflow with `backends=rocm-10.1-gfx1100` on a fork.

## Host requirements

* `amdgpu` with KFD enabled: `/dev/kfd` and `/dev/dri/renderD*` present, user in the `render` and
  `video` groups.
* No `amdgpu.gttsize` / `ttm.pages_limit` / `iommu=pt` tuning. Those exist to give system RAM to the
  Strix Halo iGPU; a discrete card already has its own VRAM.
* No `HSA_OVERRIDE_GFX_VERSION`. `gfx1100` is natively supported by ROCm 10.1, and the image installs
  AMD's own `amdrocm10.1-gfx1100` kernel and library set.

## Build

```sh
cd toolboxes
docker build --no-cache -t llama-rocm-10.1-gfx1100 -f Dockerfile.rocm-10.1-gfx1100 .
```

| Build arg | Default | Notes |
| :--- | :--- | :--- |
| `REPO` / `BRANCH` | `ggml-org/llama.cpp` / `master` | Upstream llama.cpp. |
| `GPU_TARGET` | `gfx1100` | `gfx1101` / `gfx1102` for the smaller Navi 32/33 cards. |
| `CPU_ARCH` | `znver3` | Passed to the host compiler. Set it to your CPU (`znver4`, `native` is not used). |
| `FA_QUANTS` | `f16-f16;q8_0-q8_0;q8_0-f16;f16-q8_0` | FlashAttention K/V quantizations to compile. `f16-f16` is always compiled in. More types means a longer build. |

`GGML_NATIVE` is deliberately `OFF`: the CPU code is compiled for `CPU_ARCH` instead of whatever
machine ran the build, so the image is safe to build on one box and run on another. With `ON` the
build host's instruction set gets baked in and the binaries can crash with `SIGILL` elsewhere.

## Run

```sh
docker run --rm -it \
  --device /dev/dri --device /dev/kfd \
  --group-add video --group-add render \
  --ulimit memlock=-1:-1 \
  -v /mnt/NVMe/LLMs:/models:ro \
  -p 8080:8080 \
  llama-rocm-10.1-gfx1100
```

Rootless Docker works as-is: the container reaches `/dev/kfd` and the render node, and
`ulimit -l` comes up unlimited inside it. Toolbx equivalent:

```sh
toolbox create llama-rocm-10.1-gfx1100 --image localhost/llama-rocm-10.1-gfx1100 \
  -- --device /dev/dri --device /dev/kfd --group-add video --group-add render \
  --security-opt seccomp=unconfined
```

Check that the device is visible before loading a model:

```sh
llama-cli --list-devices        # expect: ROCm0 gfx1100
amd-smi static --asic
```

## Flags worth using on a 20 GB card

`-fa on` with `-ctk q8_0 -ctv q8_0` is the combination this build is compiled for; it is the cheap way
to buy context. Everything else follows normal discrete-GPU practice:

* `-ngl all`, `-b 2048 -ub 512`, `--fit on` are good defaults.
* No `--no-mmap`. That workaround is for the Strix Halo iGPU and unified memory; mmap is fine here, and
  `load-mode = none` (read the file in at load) is a reasonable choice if you swap models often.
* MoE models that do not fit: keep attention on the GPU and move experts out with `-n-cpu-moe N`
  rather than lowering `-ngl`.
* `GGML_CUDA_ENABLE_UNIFIED_MEMORY=1` is the fallback when a model exceeds VRAM. It is slower than
  spilling experts deliberately, and it is not needed for anything that fits.

## Leave the card some room

On a desktop GPU that also drives displays, do not let `--fit` take the whole card. `--fit`
sizes the KV cache from what HIP reports free, and that figure does not know what the
compositor still needs. With a 512 MiB margin the first request on a Hyprland desktop took
the card to 21.43 of 21.47 GiB and the machine hung hard - no OOM message, nothing in the
journal, no TDR on Linux to save you. Use a real margin (`fit-target = 2048` or more) or
fix the context with `-c` and skip `--fit` altogether. Watch it during the first runs:

```sh
cat /sys/class/drm/card1/device/mem_info_vram_used   # bytes
```

## What was checked

Built against llama.cpp master `48e9857`, run on an RX 7900 XT:

* `llama-cli --list-devices` reports `ROCm0: AMD Radeon RX 7900 XT (20464 MiB, ...)`.
* `test-backend-ops -b ROCm0 -o MUL_MAT` - 2444/2444 OK, so the hipBLAS GEMM path for gfx1100 works.
* `test-backend-ops -b ROCm0 -o FLASH_ATTN_EXT,COPY,RMS_NORM` - 4046/4046 OK, WMMA FlashAttention
  included, with HIP graphs active (`CUDA graph warmup complete` in the log).

Real-model throughput is a separate matter - run `llama-bench` on the card you actually want to size,
and against the `vulkan-radv` image before picking a backend.

## Reusing a Vulkan preset file

The preset files only need the device swapped:

```sh
sed 's/^dev = Vulkan0/dev = ROCm0/' llama-server.ini > llama-server-rocm.ini
```

Then `llama-server --models-preset /models/llama-server-rocm.ini`, with the model directory mounted
at the path the preset refers to. Nothing else in the file is backend-specific: `flash-attn`,
`cache-type-k/v = q8_0`, `n-cpu-moe`, `fit` and `spec-type` all behave the same on ROCm.
