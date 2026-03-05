# Waifu-AI llama.cpp Canister (v2)
# upgraded to the latest upstream llama.cpp.
Internet Computer Specific

Based on [onicai/llama_cpp_canister](https://github.com/onicai/llama_cpp_canister), 

## What's New in v2

This is a major upgrade from the v1 canister (`.ernie-deploy/`), which used an older llama.cpp fork.

### New Model Architectures

| Architecture | Models | Status |
|---|---|---|
| **Qwen3.5** | Qwen3.5-0.8B | Uploaded, testing |
| **Qwen3** | Qwen3-0.5B, 1.5B | Supported |
| **Qwen2** | Qwen2.5-0.5B, 1.5B | Supported |
| **Phi-3** | Phi-3-mini (3.8B) | Supported (if memory allows) |
| **SmolLM3** | SmolLM2-135M | Uploaded, testing |
| **DeepSeek** | DeepSeek-R1 distills | Supported |
| **Gemma** | Gemma-2B | Supported |
| **StableLM** | StableLM-2 | Supported |
| **LLaMA** | LLaMA-3.2-1B | Supported |

### Performance Improvements

- **WASM SIMD** (`-msimd128`) — hardware-accelerated vector operations for faster inference
- **Better quantization** — improved Q4_K_M and other formats for smaller models with less quality loss
- **Optimized compute graphs** — per-model graph building and backend scheduler for efficient execution
- **Backend sampling** — sampler runs in the compute graph, reducing overhead per token

### Technical Upgrades

- Per-model architecture files (vs. monolithic source in v1)
- Improved KV cache management with ISWA support
- Better memory allocation and buffer management
- Updated GGUF format support for newer model metadata

## ICP WASM Patches

llama.cpp is written for native execution. To run inside an ICP canister (WASM sandbox), the following patches are applied:

### 1. Exception Handling
ICP WASM is compiled with `-fno-exceptions`. All `throw` statements are replaced with `IC_API::trap()`, and all `try-catch` blocks are replaced with just the try body.

### 2. Threading Removal
ICP canisters are single-threaded. All `std::thread` and `std::mutex` usage is removed. `std::thread::hardware_concurrency()` returns 1. Parallel worker loops are replaced with sequential execution.

### 3. Environment Variables
`getenv()` is not available in the WASM sandbox. All `getenv()` calls are replaced with safe defaults (typically `0` or `nullptr`).

### 4. No Memory-Mapped I/O
`mmap` is not available. File I/O uses the virtual filesystem provided by [ic-wasi-polyfill](https://github.com/wasm-forge/ic-wasi-polyfill).

### 5. WASM Globals Limit
ICP enforces a maximum of 1000 WASM globals. A post-build optimization step (`scripts/optimize_wasm.py`) uses [binaryen](https://github.com/WebAssembly/binaryen) to reduce globals from ~1037 to 1 and shrink the WASM from ~9.4MB to ~5.4MB.

## Build

### Prerequisites
- Python 3.11+ with `icpp-pro` installed (`pip install -r requirements.txt`)
- `dfx` CLI ([install](https://internetcomputer.org/install.sh))

### Build WASM
```bash
icpp build-wasm
```
This compiles all source files with wasi-sdk (clang++ targeting wasm32-wasi), links with a 32MB stack, and runs the binaryen optimizer. Takes ~15-20 minutes.

### Build Configuration
Key settings in `icpp.toml`:
- **Compiler flags:** `-DNDEBUG -DGGML_USE_CPU`
- **C flags:** `-msimd128` (WASM SIMD)
- **Stack size:** 32MB (`-Wl,-z,stack-size=33554432`)
- **Post-build:** `scripts.optimize_wasm.main` (binaryen optimization)

## Deploy

```bash
export DFX_WARNING=-mainnet_plaintext_identity

# Deploy or upgrade the canister
echo "yes" | dfx canister install llama_cpp_v2 --network ic --mode upgrade --wasm build/llama_cpp.wasm

# Verify
dfx canister call llama_cpp_v2 health --network ic
# → (variant { Ok = record { status_code = 200 : nat16 } })
```

## Upload a Model

Models are uploaded in 2MB chunks using the dfx-based upload script:

```bash
# Download a model (example: Qwen3.5-0.8B)
mkdir -p models/Qwen
wget -O models/Qwen/qwen3.5-0.8b-q4_k_m.gguf \

# Upload to the canister (takes ~25 min for 500MB)
python -u scripts/dfx_upload.py --network ic --canister llama_cpp_v2 models/Qwen/qwen3.5-0.8b-q4_k_m.gguf

# Verify the upload
dfx canister call llama_cpp_v2 file_upload_verify '(record { filename = "models/model.gguf" })' --network ic
```
**Note:** Use `scripts/dfx_upload.py` (dfx-based) rather than `scripts/upload.py` (ic-py based) — the dfx version is more reliable for mainnet uploads.

## Load and Run a Model

```bash
# Load the model (use -fit off to skip memory fitting)
dfx canister call llama_cpp_v2 load_model '(record {
  args = vec {"--model"; "models/model.gguf"; "-fit"; "off"}
})' --network ic

# Set max tokens per update call
dfx canister call llama_cpp_v2 set_max_tokens '(record {
  max_tokens_query = 1 : nat64;
  max_tokens_update = 10 : nat64
})' --network ic

# Start a new chat
dfx canister call llama_cpp_v2 new_chat '(record {
  args = vec {"--prompt-cache"; "prompt.cache"}
})' --network ic

# Run inference
dfx canister call llama_cpp_v2 run_update '(record {
  args = vec {
    "--prompt-cache"; "prompt.cache"; "--prompt-cache-all";
    "--temp"; "0.6";
    "-sp";
    "-p"; "<|im_start|>system\nYou are a helpful assistant.<|im_end|>\n<|im_start|>user\nHello!<|im_end|>\n<|im_start|>assistant\n";
    "-n"; "512"
  }
})' --network ic
```
## Canister API

| Endpoint | Type | Access | Description |
|---|---|---|---|
| `health` | Query | Public | Returns HTTP 200 status |
| `ready` | Query | Public | Check if model is loaded |
| `file_upload_chunk` | Update | Controller | Upload model file chunks |
| `file_upload_verify` | Query | Controller | Verify upload SHA256 |
| `load_model` | Update | Controller | Load GGUF model into memory |
| `set_max_tokens` | Update | Controller | Configure generation limits |
| `new_chat` | Update | Configurable | Start a new chat session |
| `run_update` | Update | Configurable | Run inference (generate tokens) |
| `set_access` | Update | Controller | Configure access control |
| `log_pause` / `log_resume` | Update | Controller | Toggle verbose logging |

## Credits

- [onicai/llama_cpp_canister](https://github.com/onicai/llama_cpp_canister) — original ICP canister implementation
- [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) — upstream llama.cpp
- [icpp-pro](https://github.com/icppWorld/icpp-pro) — C++ canister build toolchain
