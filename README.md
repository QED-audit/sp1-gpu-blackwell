<div align="center">

![SP1 Cluster](./.github/assets/header.png)

[![Github Actions][gha-badge]][gha-url] [![Telegram Chat][tg-badge]][tg-url] [![Docs][docs-badge]][docs-url]

The official GPU prover implementation for SP1, written in CUDA.

[gha-badge]: https://img.shields.io/github/actions/workflow/status/succinctlabs/sp1/pr.yml?branch=main
[gha-url]: https://github.com/succinctlabs/sp1-gpu/actions/docker.yml
[tg-badge]: https://img.shields.io/endpoint?color=neon&logo=telegram&label=chat&url=https%3A%2F%2Ftg.sumanjay.workers.dev%2F%2BAzG4ws-kD24yMGYx
[tg-url]: https://t.me/+AzG4ws-kD24yMGYx
[docs-badge]: https://img.shields.io/badge/docs-gray?logo=docusaurus
[docs-url]: https://docs.succinct.xyz
</div>

---

# QED-audit fork — SP1 GPU proving on NVIDIA GB10 (Grace-Blackwell / ARM64)

This is a fork of [`succinctlabs/sp1-gpu`](https://github.com/succinctlabs/sp1-gpu) (the
"moongate" prover) **patched to build and produce verifying proofs on the NVIDIA GB10
(DGX Spark) — Grace-Blackwell, `aarch64`, compute capability 12.1 (`sm_121`) — with the
CUDA 13 toolkit.** Upstream ships only an `x86_64` prebuilt GPU server and does not support
ARM64/Blackwell; this fork makes the prover run natively on that hardware.

Verified: `moongate-perf --program fibonacci` **proves and verifies** on a GB10
(14.4M cycles, 8 shards, ~19 s, core stage). Base done on commit `8fd1ef7`.

## What was wrong and the two fixes

**1. Blackwell `sm_121` SASS miscompilation → invalid proofs (`core/build.rs`).**
NVIDIA GPUs compile CUDA to a virtual ISA (**PTX**) and then to per-generation machine
code (**SASS**) via `ptxas`. Under CUDA 13, compiling **natively to `sm_121` SASS**
miscompiles a heavy kernel (NTT/low-degree-extension): the proof is generated but fails
verification with an *out-of-domain evaluation mismatch*. (Base and degree-4 extension
field arithmetic were micro-tested correct on `sm_121` — only a complex kernel is
affected.) CUDA 13 also dropped SASS codegen for the upstream `sm_86`/`sm_89` targets.
**Fix:** emit only **`compute_90` (Hopper) PTX** and let the GPU **driver JIT** it to the
running arch at load — the driver's JIT `ptxas` produces correct SASS. Also JITs fine on
Hopper/Ada/Ampere.

**2. `bit_rev_permutation_z` launch failure (`sppark/ntt/ntt.cuh`).**
The cache-optimized bit-reversal kernels use `__launch_bounds__(192, 2)` plus dynamic
shared memory that exceed the GB10's per-block resource limits, so the launch is rejected
(`cudaErrorLaunchOutOfResources`). **Fix:** fall back to the simple swap-based
`bit_rev_permutation` kernel (in-place safe via its `idx < rev` guard, no dynamic shared
memory), and clear a sticky pre-existing CUDA error before the launch.

Both fixes are small and localized; search for `QED-audit` in the source to find them.

## Build & run on the GB10 (spark)

Requires: CUDA 13 (`/usr/local/cuda-13.0`), Go 1.24 (for the gnark FFI), the SP1/Succinct
Rust toolchain, and a Rust toolchain. Native build, no Docker:

```bash
export PATH=/usr/local/cuda-13.0/bin:/usr/local/go/bin:$HOME/.sp1/bin:$HOME/.cargo/bin:$PATH
export CUDA_HOME=/usr/local/cuda-13.0
export CUDACXX=/usr/local/cuda-13.0/bin/nvcc
export LD_LIBRARY_PATH=/usr/local/cuda-13.0/lib64:$LD_LIBRARY_PATH

# prove + verify a program end-to-end on the GPU
RUST_LOG=info cargo run --release -p moongate-perf -- --program fibonacci
```

A run that prints the timing table and exits `0` has **passed proof verification**.
`moongate-server` (the drop-in GPU prover server for the SP1 SDK) builds the same way:
`cargo build -p moongate-server --release`.

## Notes / caveats

- The `compute_90` PTX-JIT path is a correctness workaround for the CUDA 13 `sm_121`
  codegen bug, not a performance-tuned native build. If a future CUDA toolkit fixes
  `sm_121` codegen, restore a native `code=sm_121` line in `core/build.rs`.
- Upstream pins **SP1 5.2.1** (BabyBear, 32-bit RISC-V). Match your guest ELF / SDK
  version accordingly.

---

## Profiling

### Jaeger

Setup Jaeger:
```
sudo docker run -it --rm -d -p4318:4318 -p4317:4317 -p16686:16686 jaegertracing/all-in-one:latest
```

Run a benchmark:
```
RUST_LOG=debug cargo run --release -p moongate-perf -- --program fibonacci
```

### Nvidia Nsight Systems

Run a benchmark:
```
RUST_LOG="debug" nsys profile --trace=cuda,nvtx cargo run --release -p moongate-perf -- --program fibonacci --trace nvtx 
```

## Server

Build the server image:
```
sudo docker build -f Dockerfile.server -t moongate-server .
```

```
DOCKER_BUILDKIT=1 docker build -f Dockerfile.server -t jtguibas/sp1-gpu:v4.0.0-rc1 --ssh default=${SSH_AGENT_AUTH_SOCK} .
```

Run the server:
```
sudo docker run -e "RUST_LOG=debug" -p 3000:3000 --rm --runtime=nvidia --gpus all moongate-server
```

## Docker Images

Our Docker images are automatically built and pushed to Amazon ECR Public using GitHub Actions. The process is defined in the `.github/workflows/docker.yml` file.

### Build and Release Process

1. The Docker image is built on every push to the `main` branch and for all pull requests targeting the `main` branch.
2. The image is always tagged with the Git commit SHA.
3. If the commit has a Git tag:
   - An additional image is pushed with that tag.
   - The image is also tagged as `latest`.

### Accessing the Images

You can browse and pull our Docker images from the Amazon ECR Public Gallery:

https://gallery.ecr.aws/succinct-labs/sp1-gpu

To pull the latest tagged release:

```
docker pull public.ecr.aws/succinct-labs/sp1-gpu:latest
```

To pull a specific version (replace `<tag>` with the desired version):

```
docker pull public.ecr.aws/succinct-labs/sp1-gpu:<tag>
```
