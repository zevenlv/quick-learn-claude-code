# quick-learn-claude-code

# S01核心机制详解
1. 最简单的代理循环
def agent_loop(messages: list):
    while True:
        # 1. 调用LLM
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        
        # 2. 保存助手回复
        messages.append({"role": "assistant", "content": response.content})
        
        # 3. 检查是否要继续循环
        if response.stop_reason != "tool_use":
            return  # 模型不再调用工具，结束循环
        
        # 4. 执行工具调用
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        
        # 5. 将工具结果加入对话历史
        messages.append({"role": "user", "content": results})

2. 核心原理
循环条件：只要模型调用工具(stop_reason == "tool_use")，就继续循环
工具执行：bash命令执行，结果返回给模型
消息累积：所有对话历史都保存在messages数组中
这就是整个AI代理的核心秘密！30行代码实现了一个能执行命令的AI助手。

你理解了这个基础循环吗？我们继续学习s02的工具扩展机制。
