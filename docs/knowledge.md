## Toolの定義においてdocstringの記述が重要

エージェントのLLMは、関数の **docstring** を強く参考にして、以下を理解する。
- このツールが何をするのか
- どのような場面で使うべきか
- どんな引数（例：city: str）が必要か
- どんな情報を返すか

##### get_weatherツールで言うと下記部分

```python
"""Retrieves the current weather report for a specified city.
Args:
    city (str): The name of the city (e.g., "New York", "London", "Tokyo").
Returns:
    dict: A dictionary containing the weather information.
          Includes a 'status' key ('success' or 'error').
          If 'success', includes a 'report' key with weather details.
          If 'error', includes an 'error_message' key.
"""
```

docstring は単なるコメントではなく、「LLMが読む説明文」として書くのがポイント。

- ユーザーの視点で何をする関数かを書く
- 引数と戻り値を丁寧に説明する
- 曖昧な表現を避ける

チュートリアルページ曰く、

```text
明確で説明的かつ正確なものにしてください。これは、LLMがツールを正しく使用するために不可欠です。
```
とのこと。

## Agentの定義について

```python
weather_agent = LlmAgent(
    model=LiteLlm(model=MODEL_GPT_4O),
    name="weather_agent_v1",
    description="Provides weather information for specific cities.",
    instruction="You are a helpful weather assistant. "
                "When the user asks for the weather in a specific city, "
                "use the 'get_weather' tool to find the information. "
                "If the tool returns an error, inform the user politely. "
                "If the tool is successful, present the weather report clearly.",
    tools=[get_weather], # Pass the function directly
)
```

`name`: このエージェントの一意の識別子。

`model`: 使用するLLMを指定。

`description`: エージェントの全体的な目的を簡潔にまとめたもの。これは他のエージェントがこのエージェントにタスクを委任するかどうかを決定する際に重要になる。

`instruction`: LLM の行動方法、人格、目標、割り当てられたリソースを具体的にどのように、いつ活用するかについての詳細なガイダンス。

`tools`: エージェントが実際に使用できる Python ツール関数を含むリスト。

チュートリアルページ曰く、

```text
`instruction`には明確かつ具体的に指示をすること。指示が詳細であればあるほど、LLMは自身の役割とツールの効果的な使用方法をより深く理解できます。必要に応じて、エラー処理についても明確に説明してください。
```

```text
`name`や`description`にはわかりやすく具体的な表現を使うこと。
これらの値はADK内部で使用され、自動的な処理の委任などの機能にとって重要な情報となります。
```

## サブエージェントの定義について

チュートリアルページ曰く、
```text
サブエージェントの`description`には、それぞれの具体的な能力を正確かつ簡潔にまとめる必要があります。
これは、効果的な自動委任を行う上で非常に重要です。
```

```text
サブエージェントinstructionフィールドは、その限定された範囲に合わせてカスタマイズし、
実行する内容と実行しない内容を正確に伝える必要があります(例: 「唯一のタスクは...」)。
```

とのこと。

ルートエージェントにサブエージェントを使わせるためには、ルートエージェント側に定義が必要。
```python
weather_agent_team = LlmAgent(
    name="weather_agent_v2",
    model=root_agent_model,
    description="The main coordinator agent. Handles weather requests and delegates greetings/farewells to specialists.",
    instruction="You are the main Weather Agent coordinating a team. Your primary responsibility is to provide weather information. "
                "Use the 'get_weather' tool ONLY for specific weather requests (e.g., 'weather in London'). "
                "You have specialized sub-agents: "
                "1. 'greeting_agent': Handles simple greetings like 'Hi', 'Hello'. Delegate to it for these. "
                "2. 'farewell_agent': Handles simple farewells like 'Bye', 'See you'. Delegate to it for these. "
                "Analyze the user's query. If it's a greeting, delegate to 'greeting_agent'. If it's a farewell, delegate to 'farewell_agent'. "
                "If it's a weather request, handle it yourself using 'get_weather'. "
                "For anything else, respond appropriately or state you cannot handle it.",
    tools=[get_weather],
    sub_agents=[greeting_agent, farewell_agent]  # ここに使用するサブエージェントをリスト形式で定義
)
```

`sub_agents`: 使用するサブエージェントをリスト形式で定義。

`instruction`: ルート エージェントにサブエージェントについて、またサブエージェントにタスクを委任するタイミングを明示的に記載。

```text
キー概念：自動的な委任（Auto Flow）
`sub_agents` リストを指定することで、ADKは自動的な委任（Automatic Delegation）を可能にします。

ルートエージェントがユーザーからのクエリを受け取ると、
そのエージェントの LLM は 自身のインストラクションやツールだけでなく、各サブエージェントの説明文（description）も考慮します。

そして、もしそのクエリがあるサブエージェントの説明内容（例：「シンプルな挨拶を処理する」）により合致していると判断された場合、
そのターンに限り、内部的に特別な「制御移譲アクション（内部アクション）」を生成し、処理をそのサブエージェントに委任します。

委任されたサブエージェントは、自身のモデル・インストラクション・ツールを使って、そのクエリを処理します。

ルートエージェントの指示が委任の決定を明確に導くようにしてください。サブエージェントの名前を挙げ、委任が行われる条件を記述してください。
```
とのこと。面白い。

現時点のコードを動作確認したログが下記。
正しくルートエージェントが各サブエージェントにタスクを委任して動作できている。

```log
❯ python3 multi_tool_agent/agent.py
get_weather tools defined.
say_hello tools defined.
say_goodbye tools defined.
✅ Agent 'greeting_agent' created using model 'model='openai/gpt-4o' llm_client=<google.adk.models.lite_llm.LiteLLMClient object at 0x121966270>'.
✅ Agent 'farewell_agent' created using model 'model='openai/gpt-4o' llm_client=<google.adk.models.lite_llm.LiteLLMClient object at 0x1219f82d0>'.
✅ Root Agent 'weather_agent_v2' created using model 'openai/gpt-4o' with sub-agents: ['greeting_agent', 'farewell_agent']
Executing using 'asyncio.run()' (for standard Python scripts)...

--- Testing Agent Team Delegation ---
Session created: App='weather_tutorial_agent_team', User='user_1_agent_team', Session='session_001_agent_team'
Runner created for agent 'weather_agent_v2'.

>>> User Query: Hello there!
--- Tool: say_hello called without a specific name (name_arg_value: None) ---
<<< Agent Response: Hello there!

>>> User Query: What is the weather in New York?
--- Tool: get_weather called for city: New York ---
<<< Agent Response: The weather in New York is sunny with a temperature of 25°C.

>>> User Query: Thanks, bye!
--- Tool: say_goodbye called ---
<<< Agent Response: Goodbye! Have a great day.
```