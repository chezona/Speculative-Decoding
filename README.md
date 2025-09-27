Speculative Decoding for Large Language Model Inference Acceleration
A Class Project for Efficient AI/Hardware (Spring 2024)
NYU Tandon School of Engineering
Author: Franklyn Okechukwu

Abstract
Large Language Models (LLMs) have demonstrated remarkable capabilities across various domains, but their inference latency, primarily due to the autoregressive nature of token generation, presents a significant challenge for real-time applications. Speculative Decoding (SD) has emerged as a promising technique to mitigate this bottleneck by using a smaller, faster "draft" model to generate candidate token sequences, which are then verified in parallel by the larger "target" LLM. This report details our investigation into vanilla speculative decoding (SD). We present experimental results comparing this technique against other speculative decoding frameworks and methodologies, including the baseline autoregressive decoding, analyzing performance metrics such as latency, throughput, and resource utilization. Our findings highlight the potential of SD while underscoring the complexities of optimizing them, particularly choosing the best speculation length γ (or k) within dynamic serving environments. The code is available at https://github.com/chezona/Speculative-Decoding.

Introduction
The deployment of large-scale Transformer-based LLMs 

5,17
 has revolutionized numerous applications, yet their operational efficiency is often hampered by significant inference latency 

9
. The standard autoregressive (AR) decoding process, where tokens are generated sequentially, creates a bottleneck, particularly as model sizes grow into the hundreds of billions of parameters 

6,17
. Each token generation requires a full forward pass of the large model, often bound by memory bandwidth rather than computation, making latency scale poorly 

18,9
.

Speculative Decoding (SD) offers a compelling approach to accelerate this process without compromising the output distribution of the target model 

1,2
. The core idea involves using a smaller, faster draft model (M 
q
​
 ) to predict a sequence of γ tokens, which are then validated efficiently in parallel by the target model (M 
p
​
 ) 

1,2
. Accepted tokens effectively allow the generation process to jump ahead, potentially generating multiple tokens per single validation pass of the target model 

1,2
. This technique, along with its variants, promises significant latency reductions, often achieving 2-3x speedups in ideal conditions 

1,2
.

This report presents an evaluation of several decoding strategies based on an experiment conducted using state-of-the-art models and frameworks. We examine standard autoregressive decoding, speculative decoding using a smaller draft LLM, Prompt Lookup Decoding (PLD) which leverages context repetition, and SmartSpec, a dynamic SD approach designed for optimizing performance in serving systems 

3
. We analyze the trade-offs involved, benchmark performance, and discuss the implications for deploying accelerated LLM inference solutions.

Contributions
This project was carried out by Franklyn Okechukwu as part of fulfilling the requirements of the class: Efficient AI and Hardware.

Problem Description
The primary challenge addressed by speculative decoding is the inherent latency in generating text from large autoregressive LLMs. Standard decoding requires T sequential forward passes of the target model to generate a sequence of T tokens. For large models (e.g., 70B+ parameters), a single forward pass can take tens to hundreds of milliseconds, especially in distributed serving setups where communication overhead adds further delay 

16,9
. This latency stems from:

Sequential Dependency: Each token x 
t
​
  depends on all preceding tokens x 
<t
​
 , forcing a serial generation process 

5
.

Memory Bandwidth Bottleneck: Inference is often limited by the time taken to load model parameters from memory rather than the computational operations (FLOPs), especially for large models and small batch sizes 

18,9
.

Scale: Larger models naturally require more time per forward pass and often necessitate model parallelism, adding communication costs 

16
.

While techniques like batching (especially continuous batching 

8
) improve overall throughput and GPU utilization by processing multiple requests concurrently, they do not fundamentally alter the sequential nature of generation for each individual request. Speculative decoding directly targets this sequential bottleneck by attempting to generate multiple tokens per target model evaluation cycle 

1,2
. However, SD introduces its own overheads (draft model execution, verification complexity) and its effectiveness depends critically on the accuracy of the draft predictions and the available system resources 

3
. Optimizing SD, especially in dynamic, high-load serving environments, remains a complex problem 

3
.

Related Work
Numerous approaches have been explored to accelerate LLM inference. Techniques like knowledge distillation 

11
 and quantization 

12
 aim to reduce model size and computational cost, often with some trade-off in accuracy. Architectural modifications, such as efficient attention mechanisms like multi-query attention 

18
, target specific computational bottlenecks.

Parallel decoding techniques
