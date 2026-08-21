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



**发展：**由静态的Word Embedding（如Word2Vec、GloVe和FastText） -> 动态预训练模型（如 ELMo、BERT、GPT、GPT-2、GPT-3、ALBERT、XLNet等）。大型语言模型可以生成上下文相关的 embedding 表示，可以更好地捕捉单词的语义和上下文信息。



**向量空间**

所有的数据都变成向量，这些向量组成一个庞大的矩阵。在这个世界里，每个词、句子、图片、用 户…都被表示成一个“点”（即向量），大家都有自己的“坐标”。 我们可以通过“距离”和“方向”来理解它们的关系。



Embedding 向量放在向量空间里，有啥用？ 

距离表示相似度 

向量之间越近：意义越相似 

向量之间越远：意义越不同 

比如： “苹果 🍎” 和 “香蕉 🍌” 的向量夹角小（近） → 都是水果 

“苹果 🍎” 和 “MacBook 💻” 的向量略远 → 一个是水果，一个是电子产品



### 什么是embedding模型

> **Embedding 模型的核心作用：把文本、图片等复杂数据转换成能表达其“语义/特征”的向量，使原本难以直接计算的数据变得可以进行数学计算，尤其可以通过向量距离/相似度判断数据之间的语义相关性。**



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



#### embedding的应用场景

* **检索：从大量数据中“找相似的”**

​	这也是RAG的核心，例如：知识库里有 10 万篇文档，用户问：

```python
用户："Redis 怎么防止缓存击穿？"
        ↓
     Embedding
        ↓
    Query向量
        ↓
去向量数据库寻找最相似的文档向量
        ↓
"使用互斥锁解决缓存击穿问题..."
        ↓
交给LLM回答
```

​	本质：**检索 = 给我一个东西，帮我找到最相似的东西。**



* **推荐：找到“和你兴趣相似”的东西**

  例如用户最近经常浏览：红酒、白酒、啤酒，系统可以形成用户兴趣的向量表示，再寻找相关商品/内容推荐给用户

  会发现它和检索非常像，其本质都是：**找相似。**

  ```python
  检索：问题   → 找相似文档
  推荐：兴趣   → 找相似商品	
  ```

* **聚类：把“相似的东西自动放在一起”**

  假设有 100 万条用户投诉，但是没有分类：

  ```python
  "骑手一直没有送到"
  "配送太慢了"
  "一个小时还没到"
  
  "优惠券不能使用"
  "满减没有生效"
  "活动券失效了"
  
  "商品包装破损"
  "瓶子碎了"
  "商品漏液"
  ```

  全部 Embedding 后：

  ```python
                      Embedding
                         ↓
  
          根据向量距离自动聚在一起
  
  第一堆                 第二堆              第三堆
  
  配送太慢               优惠券不能用         包装破损
  骑手没到               满减没生效           瓶子碎了
  一小时没到             活动券失效           商品漏液
  
     ↓                     ↓                   ↓
  配送问题               营销问题             商品问题
  ```

  其本质还是：**相似的数据，它们的向量比较接近，因此可以自动聚成一组。**

​	

* **分类：判断“这个东西属于哪一类”**

  分类是**把一个向量判断到某个已有类别里。**

  比如电商里收到用户评价，将这些评价转为Embedding：

  ```python
  "这个耳机音质很好"
  "物流太慢了"
  "客服态度很差"
  ```

  准备几个类别描述：

  ```python
  商品质量
  物流
  客服
  ```

  让后将这几个类别描述也全部做 Embedding，然后让分类模型对比用户文本和哪个类别最接近。

  这里要注意一点：**Embedding 本身通常不等于分类器。**Embedding 负责「把这句话的语义转换成数学特征。」后面的分类算法再根据这些特征判断：它属于配送问题。

  ```python
  Embedding = 提供特征
  
  分类模型 = 根据特征做判断
  ```

  

这四个场景的核心：

**Embedding 模型将文本、图片等复杂数据映射成能够表达其语义或特征的向量，使这些数据可以进行数学计算；通过计算向量之间的距离或相似度，可以衡量原始数据之间的语义相关性，在此基础上可以实现语义检索、推荐、聚类、分类等任务。**

```tex
现实世界的数据
文本 / 图片 / 商品 / 用户……
        ↓
   Embedding模型
        ↓
   语义/特征向量
        ↓
原本的数据变得"可以计算"
        ↓
   比较向量之间的关系
        ↓
┌────────┬────────┬────────┬────────┐
↓        ↓        ↓        ↓
检索     推荐      聚类      分类
找相似   找相似    相似成组   判断类别
```



### 一个主意要点

**不要把 Embedding 单纯理解成“高维数据转向量”**。文本 `"Redis缓存击穿"` 本身并不是一个“高维数据”，更准确的说法是**“复杂/非结构化数据 → 向量表示”**。另外，Embedding 历史上确实常强调“降维”，但现代文本 Embedding 可能直接产生 768、1024、1536 甚至更高维向量，**现在更重要的价值是“语义表示”，而不是单纯降维。**



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

