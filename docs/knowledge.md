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

## セッション
過去の会話や設定を保持するには`memory`が必要。
ADKは`Session State`を通じてこれを提供する。

### `Session State`とは？
`Session State` は、**特定のユーザーセッション（`APP_NAME`、`USER_ID`、`SESSION_ID` によって識別）に紐づく Python の辞書オブジェクト** 。

- セッション内の複数の会話ターンをまたいで情報を保持。
- エージェントやツールは、この状態（state）から情報を読み取ったり、書き込んだりすることが可能。
- これにより、エージェントは詳細を記憶し、挙動を適応させ、応答をパーソナライズすることが可能。

## コールバック

### `before_model_callback`
エージェントが基盤となるLLMへ送信する前に行う処理を定義できる。
LLMへ送られたくないテキストのフィルタリングとかに使う感じっぽい。

```text
目的:リクエストを検査し、必要に応じて変更するか、事前定義されたルールに基づいてリクエストを完全にブロックします。
一般的な使用例:
入力検証/フィルタリング:ユーザー入力が基準を満たしているか、または許可されていないコンテンツ (PII やキーワードなど) が含まれているかどうかを確認します。
ガードレール:有害、トピック外、またはポリシー違反のリクエストが LLM によって処理されるのを防ぎます。
動的プロンプト変更:送信直前に、LLM 要求コンテキストにタイムリーな情報 (セッション状態からなど) を追加します。
```

```log
# `BLOCK`という文字列がユーザー入力に存在した場合、LLMへの送信をブロックしている。

--- Testing Model Input Guardrail ---
--- Turn 1: Requesting weather in London (expect allowed, Fahrenheit) ---

>>> User Query: What is the weather in London?
--- Callback: block_keyword_guardrail running for agent: weather_agent_v5_model_guardrail ---
--- Callback: Inspecting last user message: 'What is the weather in London?...' ---
--- Callback: Keyword not found. Allowing LLM call for weather_agent_v5_model_guardrail. ---
--- Tool: get_weather_stateful called for London ---
--- Tool: Reading state 'user_preference_temperature_unit': Celsius ---
--- Tool: Generated report in Celsius. Result: {'status': 'success', 'report': 'The weather in London is cloudy with a temperature of 15°C.'} ---
--- Tool: Updated state 'last_city_checked_stateful': London ---
--- Callback: block_keyword_guardrail running for agent: weather_agent_v5_model_guardrail ---
--- Callback: Inspecting last user message: 'What is the weather in London?...' ---
--- Callback: Keyword not found. Allowing LLM call for weather_agent_v5_model_guardrail. ---
<<< Agent Response: The weather in London is cloudy with a temperature of 15°C.

--- Turn 2: Requesting with blocked keyword (expect blocked) ---

>>> User Query: BLOCK the request for weather in Tokyo
--- Callback: block_keyword_guardrail running for agent: weather_agent_v5_model_guardrail ---
--- Callback: Inspecting last user message: 'BLOCK the request for weather in Tokyo...' ---
--- Callback: Found 'BLOCK'. Blocking LLM call! ---
--- Callback: Set state 'guardrail_block_keyword_triggered': True ---
<<< Agent Response: I cannot process this request because it contains the blocked keyword 'BLOCK'.

--- Turn 3: Sending a greeting (expect allowed) ---

>>> User Query: Hello again
--- Callback: block_keyword_guardrail running for agent: weather_agent_v5_model_guardrail ---
--- Callback: Inspecting last user message: 'Hello again...' ---
--- Callback: Keyword not found. Allowing LLM call for weather_agent_v5_model_guardrail. ---
--- Tool: say_hello called without a specific name (name_arg_value: None) ---
<<< Agent Response: Hello there!
```

### `before_tool_callback`
ツールを実行する前の処理を定義できる。ツールの使用履歴をDB保存するときとかに便利そう。

```text
目的:ツールの引数を検証し、特定の入力に基づいてツールの実行を防止し、引数を動的に変更し、リソース使用ポリシーを適用します。
一般的な使用例:
引数の検証: LLM によって提供された引数が有効であるか、許容範囲内であるか、または予期される形式に準拠しているかどうかを確認します。
リソース保護:コストがかかったり、制限されたデータにアクセスしたり、望ましくない副作用 (特定のパラメータに対する API 呼び出しをブロックするなど) を引き起こす可能性のある入力でツールが呼び出されないようにします。
動的な引数の変更:ツールを実行する前に、セッション状態またはその他のコンテキスト情報に基づいて引数を調整します。
```

`before_model_callback`も`before_tool_callback`もafter版はないの？と思ったけどあるらしい。まあそりゃそうか。

## 他

### 参考になりそうなもの
Agent Development Kit（邦訳）
https://zenn.dev/uxoxu/books/adk-docs-japanese/viewer/index

