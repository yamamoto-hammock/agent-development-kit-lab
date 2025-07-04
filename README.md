# **Agent Development Kitの調査用リポジトリ**
## 構築手順

### python仮想環境構築＆有効化
```bash
python -m venv .venv
source .venv/bin/activate
```

### SDKインストール
```bash
pip install google-adk
```

### 必須構成の作成
```bash
mkdir multi_tool_agent/
echo "from . import agent" > multi_tool_agent/__init__.py
touch multi_tool_agent/agent.py
touch multi_tool_agent/.env
```

### マルチモデル対応のためのライブラリをインストール
```bash
pip install litellm
```

### 環境変数設定
`.env.example`ファイルを参考に、`.env`ファイルを作成し、必要なAPIキーを設定します。

```bash
# .envファイルの例
OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
GOOGLE_API_KEY="YOUR_GOOGLE_API_KEY"
ANTHROPIC_API_KEY="YOUR_ANTHROPIC_API_KEY"
```

または、直接環境変数を設定することもできます：

```bash
export OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
export GOOGLE_API_KEY="YOUR_GOOGLE_API_KEY"
export ANTHROPIC_API_KEY="YOUR_ANTHROPIC_API_KEY"
```

## 使用できるモデル

このエージェントでは以下のモデルを使用できます：

- **OpenAI**: GPT-4o, GPT-4-turbo など
- **Google**: Gemini-2.0-flash など
- **Anthropic**: Claude-sonnet-4, Claude-opus-4 など

デフォルトではOpenAIのモデルが使用されますが、`agent.py`内で簡単に切り替えることができます。

## エージェントの実行方法
```python
python3 multi_tool_agent/agent.py
```