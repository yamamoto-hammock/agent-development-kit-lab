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

