+++
date = '2025-03-02T17:29:08+08:00'
draft = false 
title = 'ollama 使用'
+++

## 安装

```shell
curl -fsSL https://ollama.com/install.sh | sh
```

运行模型示例

```
ollama run llama3.2
```

类似docker 命令，可以使用 `ollama list`  和 `ollama ps` 命令查看已安装的模型和正在运行的模型。

## 自定义模型

类似 dockerfile ， 可以创建Modelfile 来自定义模型的行为及输出。 

比如定义一个翻译助手，从英文翻译到中文。 Modelfile 可以创建如下：

```
FROM llama3.2

# set the system message
SYSTEM """
你的任务是把用户提供的{{.Prompt}} 直接翻译成中文。
"""
```

然后可以创建模型

```
ollama create fanyi-llama3.2 -f Modelfile
```

之后，可以运行

```
ollama run fanyi-llama3.2
```

Modelfile 的使用说明参考 [https://github.com/ollama/ollama/blob/main/docs/modelfile.md](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)。

## 使用api 接口服务

启动 server 服务

```
ollama serve
```

在本地起一个 11434 端口。

然后另一个终端，运行模型

```
ollama run fanyi-llama3.2
```

接口请求

```
curl http://localhost:11434/api/generate -d '{
  "model": "fanyi-llama3.2",
  "prompt": "The SYSTEM instruction specifies the system message to be used in the template, if applicable.",
  "stream": false
}'
```

