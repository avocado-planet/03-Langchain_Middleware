# LangChain 1.0 ミドルウェア 完全ガイド

## はじめに

LangChain 1.0 で導入された**ミドルウェア（Middleware）**は、エージェントのコアループに対して「フック（介入ポイント）」を差し込む仕組みです。Web フレームワーク（Express や FastAPI など）のミドルウェアと同じ考え方で、リクエスト処理の前後にロジックを挿入します。

従来の LangChain では、エージェントの挙動を細かく制御しようとするとパラメータが爆発的に増え、管理が困難でした。ミドルウェアは、これらの横断的な関心事（ロギング、リトライ、セキュリティ等）を**再利用可能なコンポーネント**として切り出します。

---

## エージェントのコアループ

LangChain のエージェントは以下のループで動作します:

```
ユーザー入力
    ↓
┌─── エージェントループ ───┐
│  モデル呼び出し          │
│    ↓                    │
│  ツール呼び出し?         │
│    YES → ツール実行      │
│       → ループ先頭に戻る │
│    NO  → 完了            │
└─────────────────────────┘
    ↓
最終応答をユーザーに返す
```

ミドルウェアはこのループの**各段階**にフック（介入）できます:

```
before_agent  ← エージェント開始前（1回だけ）
    ↓
┌─── ループ ──────────────────────┐
│  before_model  ← 毎回のモデル前  │
│    ↓                            │
│  wrap_model_call ← モデルを包む  │
│    ↓                            │
│  after_model   ← 毎回のモデル後  │
│    ↓                            │
│  wrap_tool_call ← ツールを包む   │
└─────────────────────────────────┘
    ↓
after_agent   ← エージェント完了後（1回だけ）
```

---

## フックの種類

### ノードスタイル（Node-style）

特定のタイミングで**順番に**実行されるフックです。`dict` を返すとエージェントの State にマージされます。`None` を返すと何も変更しません。

| フック | 実行タイミング | 実行回数 |
|---|---|---|
| `before_agent` | エージェント開始前 | 1回 |
| `before_model` | モデル呼び出し前 | 毎ループ |
| `after_model` | モデル応答後 | 毎ループ |
| `after_agent` | エージェント完了後 | 1回 |

### ラップスタイル（Wrap-style）

モデル呼び出しやツール実行を**包み込む**フックです。`handler` を呼ぶ/呼ばないを自分で決められるため、リトライ・キャッシュ・ショートサーキットが可能です。

| フック | 対象 | 主な用途 |
|---|---|---|
| `wrap_model_call` | モデル呼び出し | リトライ、計測、モデル切替 |
| `wrap_tool_call` | ツール実行 | 監視、リトライ、権限チェック |

### 便利デコレータ

| デコレータ | 説明 |
|---|---|
| `@dynamic_prompt` | 実行時に動的なシステムプロンプトを生成 |

---

## ミドルウェアの書き方

### 1. デコレータ形式（単機能向け）

```python
from langchain.agents.middleware import before_model, AgentState
from langgraph.runtime import Runtime
from typing import Any

@before_model
def log_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"メッセージ数: {len(state['messages'])}")
    return None  # State を変更しない
```

**メリット**: 簡潔で読みやすい。プロトタイプに最適。

### 2. クラス形式（複数フック・設定あり向け）

```python
from langchain.agents.middleware import AgentMiddleware

class LoggingMiddleware(AgentMiddleware):
    def __init__(self, verbose: bool = True):
        super().__init__()
        self.verbose = verbose

    def before_model(self, state, runtime):
        if self.verbose:
            print(f"メッセージ数: {len(state['messages'])}")
        return None

    def after_model(self, state, runtime):
        if self.verbose:
            print(f"応答: {state['messages'][-1].content[:50]}")
        return None
```

**メリット**: 複数フック・初期化パラメータ・sync/async の両方をまとめられる。

### 使い分けの目安

| 条件 | 推奨 |
|---|---|
| フック1つだけ | デコレータ形式 |
| 設定パラメータあり | クラス形式 |
| 複数フックを1つにまとめたい | クラス形式 |
| プロジェクト間で再利用 | クラス形式 |
| 素早くプロトタイプ | デコレータ形式 |

---

## カスタム State

ミドルウェア専用のフィールドを State に追加できます。

```python
from langchain.agents.middleware import AgentState
from typing_extensions import NotRequired

class TrackingState(AgentState):
    model_call_count: NotRequired[int]  # 初期値なしでもOK
    user_id: NotRequired[str]
```

`state_schema` パラメータで指定:

```python
@after_model(state_schema=TrackingState)
def increment(state: TrackingState, runtime) -> dict[str, Any] | None:
    return {"model_call_count": state.get("model_call_count", 0) + 1}
```

---

## 実行順序

```python
agent = create_agent(
    model="...",
    middleware=[A, B, C],
    tools=[...],
)
```

### before_* フック → 登録順（A → B → C）

```
A.before_agent → B.before_agent → C.before_agent
A.before_model → B.before_model → C.before_model
```

### wrap_* フック → ネスト（A が B を包み、B が C を包む）

```
A.wrap_model_call(
    B.wrap_model_call(
        C.wrap_model_call(
            実際のモデル呼び出し
        )
    )
)
```

### after_* フック → 逆順（C → B → A）

```
C.after_model → B.after_model → A.after_model
C.after_agent → B.after_agent → A.after_agent
```

この「before は順、after は逆」のパターンは、Web フレームワークのミドルウェアと同じです。

---

## Agent Jumps（フロー制御）

`jump_to` を返すことで、エージェントの実行フローを変更できます。

| jump_to 値 | 効果 |
|---|---|
| `"end"` | エージェントを即座に終了 |
| `"tools"` | ツールノードにジャンプ |
| `"model"` | モデルノードにジャンプ |

**重要**: `jump_to` を使うには `@hook_config(can_jump_to=[...])` で事前に宣言が必要です。

```python
from langchain.agents.middleware import hook_config

class GuardMiddleware(AgentMiddleware):
    @hook_config(can_jump_to=["end"])
    def before_model(self, state, runtime):
        if len(state["messages"]) >= 50:
            return {
                "messages": [AIMessage("上限に達しました。")],
                "jump_to": "end",
            }
        return None
```

---

## ビルトインミドルウェア一覧

LangChain には以下のビルトインミドルウェアが用意されています:

| ミドルウェア | フック | 説明 |
|---|---|---|
| `SummarizationMiddleware` | `before_model` | トークン上限接近時に古いメッセージを自動要約 |
| `HumanInTheLoopMiddleware` | `after_model` | 危険なツール実行前に人間の承認を要求 |
| `ModelCallLimitMiddleware` | — | モデル呼び出し回数の上限を設定 |
| `ToolCallLimitMiddleware` | — | ツール実行回数の上限を設定 |
| `ModelFallbackMiddleware` | `wrap_model_call` | エラー時に代替モデルへ自動フォールバック |
| `PIIMiddleware` | `before_agent` / `after_agent` | 個人情報（PII）の検出と隠蔽 |
| `ToolRetryMiddleware` | `wrap_tool_call` | ツール実行失敗時のバックオフ付きリトライ |
| `ToolSelectionMiddleware` | `wrap_model_call` | LLMで関連ツールを事前選択 |
| `LLMToolEmulator` | `wrap_tool_call` | テスト用にツール実行をLLMでエミュレート |
| `ShellToolMiddleware` | — | 永続的シェルツールを登録 |
| `TodoMiddleware` | — | ToDoリスト管理機能を提供 |

### SummarizationMiddleware の例

```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4.1",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4.1-mini",
            trigger=("tokens", 4000),    # 4000トークンで発動
            keep=("messages", 20),       # 最新20メッセージは保持
        ),
    ],
)
```

### HumanInTheLoopMiddleware の例

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="openai:gpt-4.1",
    tools=[read_email, send_email],
    checkpointer=InMemorySaver(),  # 必須
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "read_email": False,  # 承認不要
            }
        ),
    ],
)
```

---

## 本番構成のベストプラクティス

### 1. ミドルウェアの順序

```python
middleware = [
    # 1. ロギング（最初に配置 → before は最初、after は最後に実行）
    ProductionLoggingMiddleware(),

    # 2. セキュリティ / バリデーション
    PIIMiddleware("email", strategy="redact"),

    # 3. リトライ / フォールバック
    ModelFallbackMiddleware(fallback_models=["gpt-4.1-mini"]),

    # 4. コンテキスト管理
    SummarizationMiddleware(model="gpt-4.1-mini", trigger=("tokens", 4000)),

    # 5. HITL（必要な場合）
    HumanInTheLoopMiddleware(interrupt_on={...}),
]
```

### 2. 設計原則

- **単一責任**: 1つのミドルウェアは1つの仕事だけ
- **エラーハンドリング**: ミドルウェアの例外でエージェントが止まらないように
- **テスト可能性**: ミドルウェアは独立してユニットテスト
- **設定可能**: ハードコードを避け、コンストラクタで設定を渡す

### 3. create_agent の注意点

```python
# ❌ tools は省略不可
agent = create_agent(model="...", middleware=[...])

# ✅ ツールがなくても空リストを渡す
agent = create_agent(model="...", tools=[], middleware=[...])

# ❌ HumanInTheLoopMiddleware には checkpointer が必須
agent = create_agent(model="...", tools=[...], middleware=[hitl])

# ✅
agent = create_agent(
    model="...", tools=[...],
    middleware=[hitl],
    checkpointer=InMemorySaver(),
)
```

---

## フック全パターン早見表

| # | パターン | 形式 | フック | 用途 |
|---|---|---|---|---|
| 1 | ログ出力 | デコレータ | `@before_model` | メッセージ数の記録 |
| 2 | 応答ログ | デコレータ | `@after_model` | モデル応答の記録 |
| 3 | 初期化 | デコレータ | `@before_agent` | リソース準備 |
| 4 | クリーンアップ | デコレータ | `@after_agent` | 集計・後処理 |
| 5 | リトライ | デコレータ | `@wrap_model_call` | エラー時の再試行 |
| 6 | ツール監視 | デコレータ | `@wrap_tool_call` | 実行時間・結果の記録 |
| 7 | 動的プロンプト | デコレータ | `@dynamic_prompt` | 時刻やコンテキスト注入 |
| 8 | 全ライフサイクル | クラス | 全フック | 本番用ロギング |
| 9 | カスタムState | デコレータ | `state_schema=...` | 呼び出し回数追跡 |
| 10 | フロー制御 | クラス | `jump_to` | 上限チェック・ガード |
| 11 | モデル切替 | デコレータ | `@wrap_model_call` | 動的モデル選択 |
| 12 | プロンプト加工 | デコレータ | `@wrap_model_call` | SystemMessage 編集 |

---

## インストール

```bash
pip install --pre -U langchain langgraph langchain-openai langchain-anthropic
```

> LangChain 1.0 はまだアルファ/ベータ段階のため `--pre` フラグが必要です。

---

## 参考リンク

- [LangChain Middleware 公式ドキュメント](https://docs.langchain.com/oss/python/langchain/middleware/overview)
- [カスタムミドルウェア](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [ビルトインミドルウェア](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Middleware API リファレンス](https://reference.langchain.com/python/langchain/middleware)
- [LangChain ブログ: Agent Middleware](https://blog.langchain.com/agent-middleware/)
