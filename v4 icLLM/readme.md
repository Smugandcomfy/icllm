The v4 WASM patched graph_max_nodes. freed up ~200-300MB of memory.
This raised the model size ceiling from ~469MB to ~650-750MB, enabling higher-quality Q8_0 quantization for the coder model. 

the ICP instruction limit is the real bottleneck — 0.8B+ models are too slow to run.



