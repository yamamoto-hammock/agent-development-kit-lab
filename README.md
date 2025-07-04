

# **Agent Development Kitの調査用リポジトリ**

# 構築手順
## python仮想環境構築＆有効化
```bash
python -m venv .venv
source .venv/bin/activate
```

## SDKインストール
```bash
pip install google-adk
```

## 必須構成の作成
```bash
mkdir multi_tool_agent/
echo "from . import agent" > multi_tool_agent/__init__.py
touch multi_tool_agent/agent.py
touch multi_tool_agent/.env
```

## OpenAIのLLMを使うためのライブラリをインストール（geminiとかなら不要）
```bash
pip install litellm
```

## 環境変数設定
```bash
export OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
```

## OpenAIのLLMを使うためにagent.pyを編集

```python
# ファイル先頭辺りに追記
from google.adk.agents import LlmAgent
from google.adk.models.lite_llm import LiteLlm

# 元のroot_agentはコメントアウトする
# root_agent = Agent(
#     name="weather_time_agent",
#     model="gemini-2.0-flash",
#     description=(
#         "Agent to answer questions about the time and weather in a city."
#     ),
#     instruction=(
#         "You are a helpful agent who can answer user questions about the time and weather in a city."
#     ),
#     tools=[get_weather, get_current_time],
# )

# 追記
root_agent = LlmAgent(
    model=LiteLlm(model="openai/gpt-4o"), # LiteLLM model string format
    name="weather_time_agent",
    description=(
        "Agent to answer questions about the time and weather in a city."
    ),
    instruction=(
        "You are a helpful agent who can answer user questions about the time and weather in a city."
    ),
    tools=[get_weather, get_current_time],
)
```

## エージェントをWebで起動
```bash
adk web
```
ブラウザで表示=>http://127.0.0.1:8000/

ドキュメント：https://google.github.io/adk-docs/get-started/quickstart/#dev-ui-adk-web

## エージェントをターミナルで起動
```bash
adk run multi_tool_agent
```

ドキュメント：https://google.github.io/adk-docs/get-started/quickstart/#terminal-adk-run

## APIサーバーとして起動
```bash
adk api_server
```

ドキュメント：https://google.github.io/adk-docs/get-started/quickstart/#api-server-adk-api_server