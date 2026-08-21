# Ollama 部署 Qwen3.5-9B 操作手册

适用设备：Apple M4、24GB 统一内存的 MacBook Pro  
当前模型存储：Mac 内置磁盘  
Hugging Face 模型名：`Qwen/Qwen3.5-9B`  
Ollama 模型名：`qwen3.5:9b`
Qwen3.5-9B 原生聊天接口：`POST http://127.0.0.1:11434/api/chat`  
Qwen3.5-9B OpenAI 兼容接口：`POST http://127.0.0.1:11434/v1/chat/completions`

## 1. 部署方案

本手册选择 Ollama 官方模型库中的：

```text
qwen3.5:9b
```

该版本约 9.65B 参数，采用 Q4_K_M 量化，模型下载约 6.6GB。它支持文本、图片输入、工具调用和思考能力。你的 24GB Mac 适合运行这个版本。

不要误选下面这个版本：

```text
qwen3.5:9b-mlx-bf16
```

该 BF16 版本约 19GB，会给 24GB 统一内存带来很大压力。第一次部署优先使用 `qwen3.5:9b`。

虽然模型标称最大上下文为 256K，但这不代表你的电脑适合实际使用 256K。第一次运行先使用 Ollama 默认上下文；后续建议控制在 4K～8K，确有需要再尝试 16K。

## 2. 确认 Ollama 正常

在“应用程序”中启动 Ollama，确认菜单栏出现 Ollama 图标。然后打开“终端”，执行：

```bash
ollama --version
```

再执行：

```bash
ollama list
```

如果显示表头且没有模型，属于正常情况，说明 Ollama 已启动，但尚未下载模型。

如果出现无法连接 `127.0.0.1:11434`，先执行：

```bash
open -a Ollama
```

等待几秒，再运行 `ollama list`。

## 3. 下载 Qwen3.5-9B

执行：

```bash
ollama pull qwen3.5:9b
```

下载量约 6.6GB。下载过程中保持网络连接，不要让 Mac 进入深度睡眠。如果下载中断，重新执行同一个命令即可。

下载完成后验证：

```bash
ollama list
```

正常情况下会看到类似信息：

```text
NAME           SIZE
qwen3.5:9b     6.6 GB
```

查看 Ollama 模型目录占用：

```bash
du -sh ~/.ollama/models
```

## 4. 第一次运行

执行：

```bash
ollama run qwen3.5:9b
```

出现输入提示符后，可以输入：

```text
你好，请介绍你自己，并说明你是否运行在我的本地 Mac 上。
```

再测试中文总结：

```text
请用五个要点解释什么是大语言模型，面向没有技术基础的读者。
```

退出当前对话：

```text
/bye
```

也可以按 `Control + D` 退出。

## 5. 检查 GPU 和内存状态

模型正在生成或刚完成回答时，另开一个终端窗口执行：

```bash
ollama ps
```

它会显示：

- 当前加载的模型；
- 模型占用大小；
- CPU/GPU 使用比例；
- 上下文长度；
- 模型预计何时从内存卸载。

在 Apple Silicon 上，如果模型正常使用 GPU，`PROCESSOR` 一栏通常会显示 GPU 占比较高或 `100% GPU`。

还可以打开 macOS 的“活动监视器”，在“内存”页面查看 Ollama 的实际占用。

## 6. 停止模型并释放运行内存

退出聊天界面后，模型可能继续在统一内存中保留几分钟，以便下次快速启动。这属于正常行为。

要立即卸载模型并释放内存，执行：

```bash
ollama stop qwen3.5:9b
```

然后确认：

```bash
ollama ps
```

如果列表为空，说明模型已从运行内存卸载。模型文件仍保留在磁盘，下次不需要重新下载。

## 7. 再次使用模型

以后启动 Ollama 后，直接执行：

```bash
ollama run qwen3.5:9b
```

模型已经存在时，Ollama 会直接加载，不会再次完整下载。

## 8. 端口与 RESTful API

### 8.1 Qwen3.5-9B 的实际访问地址

重要说明：`qwen3.5:9b` 在 Ollama 中没有独立端口。

Ollama 与 Docker 的这一点不同：Docker 容器可以分别映射不同端口；Ollama 的模型不是独立网络服务，而是由同一个 Ollama 推理服务加载和管理。程序向 Ollama 发送请求，Ollama 再根据 `model` 字段选择 `qwen3.5:9b` 执行推理。

```text
Python / Java 程序
        ↓
POST http://127.0.0.1:11434/api/chat
        ↓
Ollama 根据 model="qwen3.5:9b" 选择模型
        ↓
Qwen3.5-9B 在 M4 GPU 上推理
```

因此，调用 Qwen3.5-9B 时真正需要使用的是下面的完整接口，而不是单独寻找一个“模型端口”：

| 调用方式 | Qwen3.5-9B 完整地址 |
|---|---|
| Ollama 原生聊天 API | `POST http://127.0.0.1:11434/api/chat` |
| Ollama 原生单轮生成 API | `POST http://127.0.0.1:11434/api/generate` |
| OpenAI 兼容聊天 API | `POST http://127.0.0.1:11434/v1/chat/completions` |
| 查看本地模型列表 | `GET http://127.0.0.1:11434/api/tags` |
| 查看正在运行的模型 | `GET http://127.0.0.1:11434/api/ps` |

每次调用都必须在 JSON 中指定：

```json
{
  "model": "qwen3.5:9b"
}
```

如果 Ollama 中同时安装 Qwen、DeepSeek 等多个模型，它们仍然共用这些接口，不会分别变成 `11435`、`11436`。模型名称才是路由标识。

### 8.2 Ollama 的监听信息

Ollama 在 Mac 上默认监听：

```text
主机：127.0.0.1（等同于 localhost）
端口：11434
Ollama 原生 API：http://127.0.0.1:11434/api
OpenAI 兼容 API：http://127.0.0.1:11434/v1
```

默认的 `127.0.0.1` 只能从这台 Mac 本机访问，局域网中的其他电脑不能直接访问。

### 8.3 在浏览器中检查服务和模型列表

先确保 Ollama 已启动：

```bash
open -a Ollama
```

可以在 Safari 或 Chrome 地址栏依次访问：

```text
http://127.0.0.1:11434/
```

正常时会显示 Ollama 正在运行。

查看 Ollama 原生格式的本地模型列表：

```text
http://127.0.0.1:11434/api/tags
```

查看 OpenAI 兼容格式的本地模型列表：

```text
http://127.0.0.1:11434/v1/models
```

下载 `qwen3.5:9b` 后，返回的 JSON 中应能看到该模型名称。

Ollama 本地服务不自带 Swagger UI 或图形化 API 调试页面。浏览器地址栏适合访问上述 GET 接口；聊天等 POST 接口需要通过 cURL、Postman、Python 或 Java 调用。完整接口文档位于 Ollama 官方 API 文档网站。

### 8.4 常用原生 REST API

| 方法 | 地址 | 作用 |
|---|---|---|
| `GET` | `/api/tags` | 查看已经下载的模型 |
| `GET` | `/api/ps` | 查看当前加载到内存的模型 |
| `POST` | `/api/chat` | 多轮聊天，推荐应用开发使用 |
| `POST` | `/api/generate` | 根据单个 prompt 生成文本 |
| `POST` | `/api/show` | 查看指定模型的详细信息 |
| `POST` | `/api/embed` | 生成向量，通常配合专用嵌入模型使用 |

完整地址的组合方式是：

```text
http://127.0.0.1:11434 + 接口路径
```

例如聊天接口的完整地址：

```text
http://127.0.0.1:11434/api/chat
```

### 8.5 使用 cURL 调用 Qwen3.5-9B

发送一次非流式请求：

```bash
curl http://127.0.0.1:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3.5:9b","messages":[{"role":"user","content":"请用三句话解释本地大模型的优点。"}],"stream":false}'
```

`stream:false` 表示等待回答完成后返回一个完整 JSON，最适合第一次调试。如果省略该字段，某些生成接口会默认以 NDJSON 流式返回多个 JSON 片段。

响应中的主要内容位于：

```text
message.content
```

### 8.6 Python：不安装第三方库的 REST API 示例

创建文件 `ollama_rest_demo.py`：

```python
import json
import urllib.error
import urllib.request

URL = "http://127.0.0.1:11434/api/chat"

payload = {
    "model": "qwen3.5:9b",
    "messages": [
        {
            "role": "user",
            "content": "请用三句话解释什么是本地大模型。",
        }
    ],
    "stream": False,
}

request = urllib.request.Request(
    URL,
    data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
    headers={"Content-Type": "application/json"},
    method="POST",
)

try:
    with urllib.request.urlopen(request, timeout=300) as response:
        result = json.load(response)
    print(result["message"]["content"])
except urllib.error.URLError as exc:
    print(f"无法连接 Ollama：{exc}")
```

运行：

```bash
python3 ollama_rest_demo.py
```

该示例只使用 Python 标准库，不需要 `pip install`。

### 8.7 Python：使用 Ollama 官方客户端

建议为练习项目创建虚拟环境：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install ollama
```

创建文件 `ollama_sdk_demo.py`：

```python
from ollama import Client

client = Client(host="http://127.0.0.1:11434")

response = client.chat(
    model="qwen3.5:9b",
    messages=[
        {
            "role": "user",
            "content": "请列出学习大模型应用开发的五个步骤。",
        }
    ],
)

print(response.message.content)
```

运行：

```bash
python ollama_sdk_demo.py
```

使用结束后退出虚拟环境：

```bash
deactivate
```

### 8.8 Python：OpenAI 兼容接口示例

如果现有项目已经使用 OpenAI Python 客户端，可以只修改 `base_url` 和模型名：

```bash
source .venv/bin/activate
python -m pip install openai
```

创建文件 `ollama_openai_demo.py`：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:11434/v1/",
    api_key="ollama",  # 本地接口要求客户端字段非空，但 Ollama 会忽略它
)

response = client.chat.completions.create(
    model="qwen3.5:9b",
    messages=[
        {
            "role": "user",
            "content": "请用 Java 和 Python 各举一个 REST API 调用场景。",
        }
    ],
)

print(response.choices[0].message.content)
```

运行：

```bash
python ollama_openai_demo.py
```

### 8.9 Java：使用 JDK HttpClient 调用 REST API

下面的示例使用 JDK 自带的 `java.net.http.HttpClient`，无需添加第三方依赖。建议使用 JDK 17 或更高版本。

创建文件 `OllamaDemo.java`：

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.Duration;

public class OllamaDemo {
    private static final String API_URL =
            "http://127.0.0.1:11434/api/chat";

    public static void main(String[] args) throws Exception {
        String jsonBody = """
                {
                  "model": "qwen3.5:9b",
                  "messages": [
                    {
                      "role": "user",
                      "content": "请用三句话解释什么是本地大模型。"
                    }
                  ],
                  "stream": false
                }
                """;

        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(API_URL))
                .timeout(Duration.ofMinutes(5))
                .header("Content-Type", "application/json; charset=UTF-8")
                .POST(HttpRequest.BodyPublishers.ofString(
                        jsonBody, StandardCharsets.UTF_8))
                .build();

        HttpResponse<String> response = client.send(
                request,
                HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8)
        );

        System.out.println("HTTP 状态码：" + response.statusCode());
        System.out.println("响应 JSON：");
        System.out.println(response.body());
    }
}
```

编译和运行：

```bash
javac OllamaDemo.java
java OllamaDemo
```

这个零依赖示例直接打印完整 JSON。正式 Java 项目可以使用 Jackson 或 Gson，将 `message.content` 映射成 Java 对象。

### 8.10 调用前检查清单

本地代码调用失败时，依次检查：

1. Ollama 应用是否启动；
2. `ollama list` 中是否存在 `qwen3.5:9b`；
3. 浏览器能否打开 `http://127.0.0.1:11434/api/tags`；
4. 请求地址是否包含正确端口 `11434`；
5. 请求中的模型名是否严格写成 `qwen3.5:9b`；
6. 是否设置了 `Content-Type: application/json`；
7. 是否使用 `stream:false` 简化首次调试。

如果 Python 或 Java 程序与 Ollama 在同一台 Mac 上，始终优先使用 `127.0.0.1`，不要为了本地调用而开放局域网监听。

## 9. AppKey / API Key 说明

### 9.1 当前本地模型不需要 API Key

本手册部署的是保存在本机并由 M4 GPU 运行的 `qwen3.5:9b`。访问下面的本地地址不需要 AppKey 或 API Key：

```text
http://127.0.0.1:11434
```

因此，本手册前面的 cURL 示例没有 `Authorization` 请求头，这是正确的用法。Qwen 的模型权重已经下载到本地，推理过程中不需要阿里云百炼或 DashScope API Key。

Ollama 默认只监听本机回环地址。不要在没有认证、防火墙和访问控制的情况下把 `11434` 端口直接暴露到局域网或公网，因为 Ollama 本地接口本身不校验 API Key。

### 9.2 第三方软件强制要求填写 API Key

有些兼容 OpenAI API 的软件即使连接本地 Ollama，也不允许 API Key 留空。这时可以填写一个没有实际权限的占位字符串，例如：

```text
Base URL: http://127.0.0.1:11434/v1
API Key:  ollama
Model:    qwen3.5:9b
```

这里的 `ollama` 只是为了通过第三方软件的必填项检查，不是真实密钥，本地 Ollama 会忽略它。不要把真实的云服务密钥当作占位符填写。

如果第三方软件原生支持 Ollama，而不是通过 OpenAI 兼容接口连接，一般填写：

```text
Base URL: http://127.0.0.1:11434
Model:    qwen3.5:9b
API Key:  留空
```

### 9.3 什么情况下才需要真实 API Key

以下情况可能需要登录或使用真实的 Ollama API Key：

- 直接调用 `https://ollama.com/api` 云端接口；
- 运行 Ollama Cloud 模型；
- 下载私有模型；
- 向 Ollama 发布模型。

通过本地 Ollama 使用云模型时，可以登录：

```bash
ollama signin
```

如果程序要直接访问 `https://ollama.com/api`，需要在 Ollama 网站创建 API Key，并在当前终端临时设置：

```bash
export OLLAMA_API_KEY="替换为自己的密钥"
```

请求时通过 Bearer 认证传递：

```bash
curl https://ollama.com/api/chat \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{"model":"云端模型名称","messages":[{"role":"user","content":"你好"}],"stream":false}'
```

安全注意事项：

- 不要把真实 API Key 写进本手册、代码仓库或截图；
- 不要把密钥直接提交到 Git；
- 不要在聊天消息中发送真实密钥；
- 怀疑泄露时，立即在密钥管理页面撤销并重新创建。

对于本手册的本地 `qwen3.5:9b`，以上云端配置全部不需要执行。

## 10. 创建自己的预设助手（可选）

在任意练习目录中创建一个名为 `Modelfile` 的文本文件，内容如下：

```text
FROM qwen3.5:9b
PARAMETER num_ctx 8192
PARAMETER temperature 0.7
SYSTEM 你是一名耐心、准确的中文学习助手。回答时先给结论，再解释原因；不确定时明确说明。
```

进入该目录后执行：

```bash
ollama create qwen-study -f Modelfile
```

运行自定义助手：

```bash
ollama run qwen-study
```

这个操作通常会复用原始模型权重，不会再完整复制一份 6.6GB 模型。

## 11. 更新模型

模型存在更新时，执行：

```bash
ollama pull qwen3.5:9b
```

Ollama 会检查并下载需要更新的内容。

通过 Homebrew 更新 Ollama：

```bash
brew upgrade --cask ollama-app
```

更新后重新启动 Ollama。

## 12. 删除模型并释放磁盘

先确认模型名称：

```bash
ollama list
```

停止模型：

```bash
ollama stop qwen3.5:9b
```

删除模型：

```bash
ollama rm qwen3.5:9b
```

再次检查：

```bash
ollama list
du -sh ~/.ollama/models
```

删除模型不会卸载 Ollama，也不会影响 macOS。以后仍可重新下载。

## 13. 磁盘与内存管理建议

针对这台 24GB Mac：

- 第一次只安装 `qwen3.5:9b`，不要同时下载多个量化版本；
- 日常上下文先保持默认值，需要时再提高到 8K；
- 避免同时加载多个模型；
- 使用结束后，如果要运行大型应用，执行 `ollama stop qwen3.5:9b`；
- 定期通过 `ollama list` 检查模型，通过 `du -sh ~/.ollama/models` 检查空间；
- 内置盘最好长期保留至少 100GB 可用空间；
- 将来购买外置 SSD 后，可以再迁移整个模型目录，不必现在重复配置。

## 14. 常见问题

### 无法连接 Ollama

症状：

```text
could not connect to a running Ollama instance
```

处理：

```bash
open -a Ollama
```

等待几秒后重试。

### 下载很慢或中断

重新执行：

```bash
ollama pull qwen3.5:9b
```

不要改用相似但来源不明的第三方模型名。

### 内存压力明显或系统出现 Swap

执行：

```bash
ollama stop qwen3.5:9b
```

关闭占用大量内存的软件，并避免使用超长上下文。不要改用 19GB 的 BF16 版本。

### 磁盘空间没有下降

先确认模型已删除：

```bash
ollama list
```

如果此前通过 Finder 手动删除过模型或缓存，还需要清空废纸篓。

## 15. 最简操作清单

第一次下载：

```bash
ollama pull qwen3.5:9b
```

开始聊天：

```bash
ollama run qwen3.5:9b
```

查看运行状态：

```bash
ollama ps
```

停止并释放内存：

```bash
ollama stop qwen3.5:9b
```

查看已安装模型：

```bash
ollama list
```

彻底删除模型：

```bash
ollama rm qwen3.5:9b
```

## 16. 本次暂不执行的操作

本手册只提供经过核对的操作步骤，目前没有自动执行 `ollama pull qwen3.5:9b`。只有执行下载命令后，才会使用约 6.6GB 网络流量和磁盘空间。
