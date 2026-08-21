# RAG+Embedding

RAG企业知识库项目



## Embedding模型



### 什么是embedding

**Embedding** 是一种技术/过程：把文本、图片等数据转换成能够表达其语义的**稠密向量**。

```python
"我喜欢骑自行车"
        ↓ Embedding
[0.12, -0.38, 0.76, ..., 0.21]
```

那么**谁负责算出这串数字？**就是由Embedding模型



### 什么是embedding模型

**Embedding 是一种将数据映射到向量空间的表示方法/技术；Embedding 模型是经过训练、实际负责完成这种映射的模型。**

```python
Embedding                  技术/方法
    ↓
BAAI/bge-m3                Embedding模型
    ↓
[0.21, -0.37, 0.82 ...]   Embedding向量
```

 

**embedding模型有哪些？**

商业模型：OpenAI、Google的embedding模型

开源模型：Qwen3-Embedding、BGE的模型等



### embedding模型的推理框架

**本地加载运行调用 Embedding 模型需要推理工具库，目前常见的 Embedding 模型加载/调用的推理工具库有：Hugging Face Transformers、Sentence-Transformers、FlagEmbedding 等，LangChain 则提供更上层的 Embeddings 抽象和集成。**



* **HuggingFace-Transformers**

  huggingFace-transformers是通用的transformer模型库，并不是专门为embedding设计的，它可以干很多事情：文本生成（Qwen/GPT模型）、文本分类、问答、embedding、其他transformer任务。

  它的特点是：**底层+通用+灵活**，但是做embedding时没有Sentence-Transformers那么方便

  

* **Sentence-Transformers**

  Sentence-Transformers 建立在 PyTorch、Transformers 等生态之上，专门面向 Embedding、语义相似度和语义检索等任务进行了封装，Sentence-Transformers更专注把文本变成好用的语义向量

  

* **FlagEmbedding**

  FlagEmbedding 是 BAAI 团队围绕**检索模型**提供的一套工具库，和 BGE 系列关系非常紧密。是建立在huggingFace/Pytorch的生态之上，它关注不只是将文本转向量，还包括稠密检索 / 稀疏检索 / 多向量检索等，**FlagEmbedding 更偏向 BGE 生态和信息检索**，当然FlagEmbedding 并不是“只能运行 BGE”



* **Langchain集成的HuggingFaceBgeEmbeddings**

  LangChain Embeddings 本质上是一个统一抽象/接口层，它主要是 LangChain 为了让你方便接入 Embedding 模型做的一层**适配器/封装**，**它的目标：**我不管你底层到底怎么产生 Embedding，只要你能按照我的 Embeddings 接口提供**`embed_query()`**、**`embed_documents()`** 等能力，我上层的 RAG、VectorStore 就可以统一使用。

  ```tex
                      LangChain
                    Embeddings接口
                         │
            ┌────────────┼────────────┐
            ↓            ↓            ↓
   HuggingFace相关集成   Ollama       云端Embedding API
            ↓
   Sentence-Transformers等
            ↓
         BGE模型
  ```

  

### 一句话总结

​	Embedding 是一种将文本、图片等数据映射到向量空间的表示方法，Embedding 模型则是经过训练、实际实现这种映射的模型。要在本地加载和运行 Embedding 模型，通常需要模型加载与推理工具库。Hugging Face Transformers 是底层且通用的 Transformer 模型库，并非专门服务于 Embedding；Sentence-Transformers 建立在 Hugging Face/PyTorch 生态之上，针对句子 Embedding、语义相似度和语义检索进行了封装；FlagEmbedding 是 BAAI 团队提供的检索模型工具库，与 BGE 系列关系紧密，除 Embedding 外还覆盖 Reranker 等检索能力；LangChain Embeddings 则处于更上层，它不是单纯对 Hugging Face 的二次封装，而是定义统一的 Embedding 调用接口，使 Hugging Face、本地模型、Ollama 或云端 Embedding API 都能以统一方式接入 RAG 和向量数据库。

















embedding是无监督学习/弱监督学习



Embedding 向量放在向量空间里，有啥用？ 

距离表示相似度 

向量之间越近：意义越相似 

向量之间越远：意义越不同



这里的意义表示的是语义



需要知道每个embedding模型的优缺点



embedding是什么，有什么好处？



www.modelscope.cn/collections







降维：在高维度空间中，数据点之间可能存在很大的距离，使得样本稀疏，嵌入模型可以减少数据 稀疏性。



什么是高纬空间：一个字一个词，用二进制的方式来进行向量化表示数量会非常庞大可能是达到几万几十万，这在大模型中非常不方便的。



问题：

qwen3-embedding是可执行文件还是什么？



embedding模型的作用是什么，为什么要本地下载练习？这是将数据转向量，觉得我们不是大模型拿来也没什么作用。



推理框架：Sentence-Transformers:



Sentence-Transformers与qwen3-embedding是什么关系?





## 一些基础问题



### NLP是什么

**NLP = Natural Language Processing，自然语言处理**：NLP 就是让计算机能够处理、理解和生成人类语言的一系列技术。

例如：

- **文本分类**：判断一条评论是好评还是差评
- **机器翻译**：中文 → 英文
- **情感分析**：判断用户是满意还是生气
- **信息抽取**：从“张三在美团工作”中提取“张三、美团”
- **文本生成/问答**：ChatGPT 这类大模型应用
- **语义检索**：通过 Embedding 找语义相近的文本



**一句话：**NLP（自然语言处理）是人工智能中专门研究如何让计算机理解、处理和生成人类语言的领域，Embedding 是 NLP 中用于将文本转换成语义向量的重要技术之一。



### Transformers是什么？解决了什么问题

> **Transformer 是现代 NLP 和大模型的核心神经网络架构，它利用 Attention 机制直接学习文本中不同位置之间的上下文关系，相比传统 RNN/LSTM 更擅长处理长距离依赖且更容易并行计算；如今 GPT、Qwen 等大语言模型以及 BGE 等现代 Embedding 模型，大多建立在 Transformer 架构之上。**



#### Transformers是什么

> **Transformer**：一种神经网络架构

**Transformer 是一种专门处理序列数据的神经网络架构，它通过 Attention（注意力机制）让模型能够理解一句话中不同词之间的关系，是现代大语言模型最核心的基础架构。**



比如：小明把手机给了小王，因为他买了新手机。

```python
“他”
 ↓
到底和前面的哪个人有关系？
 ↓
小明？小王？
```

理解一句话不能只看单个词，而要理解**词与词之间的上下文关系**。Transformer 最核心的能力之一，就是通过 **Attention** 建立这种关系。

```python
可以粗略想象成：
小明  把  手机  给了  小王  因为  他  买了  新手机
 ↑                         ↑
 └────── Attention ────────┘

模型处理“他”的时候，会关注句子里其他相关的词，并计算它们之间的关联程度。
```



#### Transformers解决了什么问题

Transformer 出现之前，NLP 经常使用 RNN、LSTM 等网络。这些有一个很直观的问题：**倾向于按顺序处理文本。**

比如：

```python
我 → 今天 → 下午 → 去 → 商场 → 买 → 手机
```

前面的信息一步一步传递到后面。这样有两个突出问题：

**第一，难以充分并行计算。**

​	因为后面的计算依赖前面的结果：第1步 → 第2步 → 第3步 → 第4步，文本越长，处理效率问题越明显。

**第二，长距离依赖比较困难。**

​	例如：

```python
我昨天在商场看到了一台手机，经过各种比较之后，虽然价格很贵，但最终还是把它买了。
```

​	“它”和前面很远的“手机”存在关系。传统 RNN/LSTM 要经过很多中间步骤传递信息，而 Transformer 的 Attention 可以更直接地建立：手机 ←────────────→ 它



所以 Transformer 的重要优势：

> **通过 Attention 更直接地建模文本中不同位置之间的关系，同时具有很强的并行计算能力，因此更适合处理大规模文本和长上下文。**



#### Transformers和大模型是什么关系

```python
                 Transformer
                神经网络架构
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
    Embedding模型               大语言模型
     BGE、E5等             GPT、Qwen、DeepSeek等
          ↓                       ↓
      生成向量                 生成文本
```

Embedding模型和大预言模型它们都可能基于 Transformer，但**训练目标和最终用途不同**。

> **Transformer 有点像一种底层架构设计，而 GPT、Qwen、BGE 等是在这种架构思想之上训练出来的不同模型。**



将知识串联起来：

```python
Transformer
│
├─ 一种神经网络架构
│
├─ BGE等Embedding模型基于它
│
├─ GPT/Qwen等LLM也基于它
│
└─ Hugging Face Transformers
      ↑
      名字很像，但这是“工具库”
      用来加载/训练/运行Transformer模型
```

