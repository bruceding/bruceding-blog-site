+++
date = '2025-04-04T22:16:16+08:00'
draft = false 
title = 'Mcp Go Client开发示例'
searchHidden = true
ShowReadingTime =  true
ShowBreadCrumbs =  true
ShowPostNavLinks =  true
ShowWordCount =  true
ShowRssButtonInSectionTermList =  true
UseHugoToc = true
showToc = true
TocOpen = false
hidemeta = false
comments = false
description = ''
disableHLJS = true 
disableShare = false
hideSummary = false
+++

在[mcp协议解析](../mcp协议解析)文章中，我们已经了解了mcp协议的基本概念，以及如何使用mcp协议来开发一个mcp server。本文将介绍如何使用mcp协议来开发一个mcp client。
我们使用 [mcp-go](github.com/mark3labs/mcp-go)来开发，并且模型调用基于阿里云百炼平台。
源码可以参考[这里](https://github.com/bruceding/pairec-mcp-demo/tree/main/client)。

整体的步骤可以描述如下：
1. 初始化 mcp client, 由于使用 stdio transport 方式，通过 command 方式，启动一个mcp server。  
2. mcp client 初始化之后，通过 list tools 接口获取所有的工具信息。 
3. 处理用户请求的信息，把 prompt 和 tools 信息传递给大模型。我们使用了qwen-plus 模型，本身支持 function call 功能，我们直接使用 tools 把工具信息传递给大模型。
4. 大模型返回结果后，解析结果，判断是否调用了工具。如果没有使用工具，直接返回大模型的结果。如果使用了工具，大模型返回的 ToolCalls 信息包含了工具名称以及调用工具需要的参数信息。
5. 根据ToolCalls 返回的信息，mcp client 通过 call tool 接口，调用工具。
6. 拿到相应的工具返回的结果后，把用户的 prompt 和工具的结果传递给大模型。
7. 拿到大模型的结果后，返回给用户。

首先在 pairec-mcp-demo 目录下进行编译，生成 pairec-mcp-demo 二进制文件, 然后 copy 到 client 目录下。 

## 初始化mcp client
这里 serverPath 就是 pairec-mcp-demo 二进制文件的路径。 
首先通过 Initialize 接口初始化mcp client。 
```go
func connectToServer(serverPath string) (*client.StdioMCPClient, error) {
	mcpClient, err := client.NewStdioMCPClient(serverPath, nil)
	if err != nil {
		return nil, fmt.Errorf("failed to connect to server: %v", err)
	}
	_, err = mcpClient.Initialize(context.Background(), mcp.InitializeRequest{
		Params: struct {
			ProtocolVersion string                 "json:\"protocolVersion\""
			Capabilities    mcp.ClientCapabilities "json:\"capabilities\""
			ClientInfo      mcp.Implementation     "json:\"clientInfo\""
		}{
			ProtocolVersion: "2024-11-05",
			Capabilities:    mcp.ClientCapabilities{},
			ClientInfo: mcp.Implementation{
				Name:    "mcp-go",
				Version: "0.1.0",
			},
		},
	})

	if err != nil {
		return nil, fmt.Errorf("failed to initialize client: %v", err)
	}
	return mcpClient, nil
}
```

## 列出所有的工具
通过 ListTools 接口列出所有的工具信息。
使用了 openai 的接口，把工具信息转化成 tools 列表。
```go
	toolsResult, err := mcpClient.ListTools(context.Background(), mcp.ListToolsRequest{})
	if err != nil {
		fmt.Println("list tools fail:", err)
		return
	}
	fmt.Println("list tools result:", toolsResult)

	var tools []openai.Tool
	for _, tool := range toolsResult.Tools {
		tools = append(tools, openai.Tool{
			Type: "function",
			Function: &openai.FunctionDefinition{
				Name:        tool.Name,
				Description: tool.Description,
				Parameters:  tool.InputSchema.Properties,
			},
		})
	}
```
## 处理用户请求
这里使用了 qwen-plus 模型，本身支持 function call 功能。我们直接使用 tools 把工具信息传递给大模型。
下面的 API_KEY 通过阿里云的百炼平台申请。
```go
    config := openai.DefaultConfig(os.Getenv("API_KEY"))
	config.BaseURL = "https://dashscope.aliyuncs.com/compatible-mode/v1"
	// 初始化客户端
	client := openai.NewClientWithConfig(config)

    
// 创建聊天请求
	resp, err := client.CreateChatCompletion(
		context.Background(),
		openai.ChatCompletionRequest{
			Model: "qwen-plus",
			Messages: []openai.ChatCompletionMessage{
				{
					Role:    openai.ChatMessageRoleUser,
					Content: query,
				},
			},
			Tools:      tools,
			ToolChoice: "auto",
		},
	)
```
## 调用工具
如果大模型返回的结果中包含了 ToolCalls 信息，说明大模型调用了工具。我们根据 ToolCalls 返回的信息，调用工具。
由于大模型返回的ToolCalls 接口的调用参数是一个 string ，我们需要把参数转化成 map[string]interface{} 类型。
```go
    toolCall := resp.Choices[0].Message.ToolCalls[0].Function
	callTooRequest := mcp.CallToolRequest{}
	callTooRequest.Params.Name = toolCall.Name
	callTooRequest.Params.Arguments = make(map[string]interface{})
	if err := json.Unmarshal([]byte(toolCall.Arguments), &callTooRequest.Params.Arguments); err != nil {
		fmt.Println("json unmarshal fail:", err)
		return
	}
	toolCallResult, err := mcpClient.CallTool(context.Background(), callTooRequest)
	if err != nil {
		fmt.Println("call tool fail:", err)
		return
	}
```
## 处理工具返回的结果
拿到工具返回的结果后，把用户的 prompt 和工具的结果传递给大模型。
```go
messages := []openai.ChatCompletionMessage{
		{
			Role:    openai.ChatMessageRoleUser,
			Content: query,
		},
		{
			Role:    openai.ChatMessageRoleAssistant,
			Content: "",
			ToolCalls: resp.Choices[0].Message.ToolCalls,
		},
		{
			Role:       openai.ChatMessageRoleTool,
			ToolCallID: resp.Choices[0].Message.ToolCalls[0].ID,
			Content:    toolCallResult.Content[0].(mcp.TextContent).Text,
		},
	}

	resp, err = client.CreateChatCompletion(
		context.Background(),
		openai.ChatCompletionRequest{
			Model:    "qwen-plus",
			Messages: messages,
		},
	)
```
## 运行示例
在 client 目录，直接运行
```bash
go run . 
```