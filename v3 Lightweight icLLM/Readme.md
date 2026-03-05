a lightweight architecture for the ic to run small language models


1. all of the upstream imorovements from llama CPP
2. compelete rewrite of oncai/v2 with persistant memory and SIMD improvements.
3. ic specific optimzations of the constructor.

example:
  
          Model           max_tokens_update           
    Qwen 0.5B            25                         ~8.5s first call, ~4s follow-up 
    SmolLM2 135M         50                         ~6s first call, ~8s follow-up   
    Qwen 0.5B            25                         ~8.5s first call, ~4s follow-up 
    Qwen 0.5B Coder      25                         ~8.5s first call, ~4s follow-up
