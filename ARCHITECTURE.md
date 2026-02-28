# tmuxcc アーキテクチャガイド

## 概要

**tmuxcc** (v0.1.7) は、複数のAIコーディングエージェント（Claude Code, OpenCode, Codex CLI, Gemini CLI）をtmuxペイン内で一括監視・操作するRust製TUIアプリケーションです。

| 項目 | 内容 |
|------|------|
| 総コード行数 | ~5,220行 |
| 言語 | Rust (Edition 2021) |
| TUIフレームワーク | Ratatui 0.29 + Crossterm 0.28 |
| 非同期ランタイム | Tokio 1.x |
| 設定ファイル | TOML形式 |

---

## ディレクトリ構造

```
src/
├── main.rs              # エントリーポイント、CLI引数解析
├── lib.rs               # ライブラリルート
├── agents/              # エージェント型定義
│   ├── types.rs         # AgentType, AgentStatus, MonitoredAgent
│   └── subagent.rs      # Subagent, SubagentType
├── app/                 # アプリケーション状態
│   ├── state.rs         # AppState, AgentTree
│   ├── actions.rs       # Action enum (40+種類)
│   └── config.rs        # TOML設定
├── monitor/             # バックグラウンド監視
│   ├── task.rs          # MonitorTask (非同期ポーリング)
│   └── system_stats.rs  # CPU/メモリ監視
├── parsers/             # エージェント別パーサー
│   ├── mod.rs           # AgentParser trait, ParserRegistry
│   ├── claude_code.rs   # Claude Code用 (600行、最複雑)
│   ├── opencode.rs
│   ├── codex_cli.rs
│   └── gemini_cli.rs
├── tmux/                # tmux連携
│   ├── client.rs        # TmuxClient (コマンドラッパー)
│   └── pane.rs          # PaneInfo, プロセスキャッシュ
└── ui/                  # TUI描画
    ├── app.rs           # メインイベントループ
    ├── layout.rs        # レイアウト定義
    ├── styles.rs        # スタイル定義
    └── components/      # 各UIウィジェット
        ├── agent_tree.rs
        ├── header.rs
        ├── footer.rs
        ├── pane_preview.rs
        └── ...
```

---

## アーキテクチャ全体図

```
┌─────────────────────────────────────────────────────────────┐
│                      main.rs                                │
│  CLI解析 → Config読込 → Terminal初期化 → run_app()          │
└─────────────────────────────┬───────────────────────────────┘
                              │
         ┌────────────────────┴────────────────────┐
         │                                         │
         ▼                                         ▼
┌─────────────────────┐              ┌─────────────────────────┐
│  MonitorTask        │   mpsc      │  UIイベントループ        │
│  (バックグラウンド)   │ ─────────▶ │  (メインスレッド)        │
│                     │  channel    │                         │
│  500ms毎にポーリング │              │  キーボード/マウス処理   │
│  • list_panes()     │              │  Action dispatch        │
│  • capture_pane()   │              │  Ratatui描画            │
│  • パーサーで解析    │              │                         │
└─────────────────────┘              └─────────────────────────┘
         ▲                                         │
         │                                         ▼
┌─────────────────────┐              ┌─────────────────────────┐
│  TmuxClient         │              │  send_keys()で承認      │
│  • tmux CLI wrapper │ ◀────────────│  focus_pane()でフォーカス│
└─────────────────────┘              └─────────────────────────┘
```

---

## コアデータ構造

### MonitoredAgent (監視対象エージェント)

```rust
struct MonitoredAgent {
    id: String,                    // 一意ID
    target: String,                // "session:window.pane"
    agent_type: AgentType,         // ClaudeCode, OpenCode等
    status: AgentStatus,           // Idle, Processing, AwaitingApproval
    subagents: Vec<Subagent>,      // 子タスク
    last_content: String,          // キャプチャ内容
    context_remaining: Option<u8>, // コンテキスト残量%
}
```

### AgentStatus (状態管理)

```rust
enum AgentStatus {
    Idle,
    Processing { activity: String },
    AwaitingApproval { approval_type: ApprovalType, details: String },
    Error { message: String },
    Unknown,
}
```

### ApprovalType (承認種別)

```rust
enum ApprovalType {
    FileEdit,
    FileCreate,
    FileDelete,
    ShellCommand,
    McpTool,
    UserQuestion { choices: Vec<String>, multi_select: bool },
}
```

---

## tmux連携の仕組み

**コントロールモードは使用せず**、CLIコマンドを直接実行:

| コマンド | 用途 |
|----------|------|
| `tmux list-panes -a -F ...` | 全ペイン列挙 |
| `tmux capture-pane -p -t <target> -S -100` | 内容キャプチャ |
| `tmux send-keys -t <target> "y" Enter` | 承認送信 |
| `tmux select-pane -t <target>` | フォーカス移動 |

### なぜコントロールモードを使わないか

- シンプルさ（標準的なコマンドライン呼び出し）
- 互換性（どのtmuxバージョンでも動作）
- デバッグの容易さ（手動でコマンドをテスト可能）

---

## パーサーシステム

`AgentParser` traitで拡張可能な設計:

```rust
trait AgentParser {
    fn matches(&self, detection_strings: &[&str]) -> bool;
    fn parse_status(&self, content: &str) -> AgentStatus;
    fn parse_subagents(&self, content: &str) -> Vec<Subagent>;
    fn parse_context_remaining(&self, content: &str) -> Option<u8>;
}
```

### 対応パーサー

| パーサー | 検出方法 |
|---------|---------|
| ClaudeCodeParser | `claude`コマンド、`✳`アイコン、バージョン番号 |
| OpenCodeParser | `opencode`コマンド |
| CodexCliParser | `codex`コマンド |
| GeminiCliParser | `gemini`コマンド |

### Claude Codeパーサーの特徴 (最も複雑、600行)

- ペインタイトルの`✳`アイコン検出
- スピナー文字検出: `⠿⠇⠋⠙⠸⠴⠦⠧⠖⠏`
- 承認プロンプトのパターンマッチ
- Subagent検出: `⏺ Task(subagent_type="Explore"...)`

---

## 主要ワークフロー

### 1. 継続監視

```
MonitorTask (500ms間隔)
  → list_panes() でペイン取得
  → ParserRegistry でパーサー選択
  → capture_pane() で内容取得
  → parse_status() で状態解析
  → MonitorUpdate を UIに送信
```

### 2. 承認処理

```
ユーザーが 'y' を押す
  → Action::Approve 発火
  → tmux_client.send_keys(target, "y")
  → tmux_client.send_keys(target, "Enter")
  → 次のポーリングで状態更新確認
```

### 3. ユーザー質問 (番号選択)

```
パーサーが選択肢を検出
  → AgentStatus::AwaitingApproval { UserQuestion { choices } }
  → UIに選択肢表示
  → ユーザーが '1'-'9' を押す
  → send_keys で番号送信
```

---

## UI構成

| コンポーネント | ファイル | 役割 |
|---------------|---------|------|
| HeaderWidget | `ui/components/header.rs` | エージェント数、CPU/メモリ表示 |
| AgentTreeWidget | `ui/components/agent_tree.rs` | 階層的エージェントリスト |
| PanePreviewWidget | `ui/components/pane_preview.rs` | ペイン内容プレビュー |
| FooterWidget | `ui/components/footer.rs` | アクションヒント |
| SubagentLogWidget | `ui/components/subagent_log.rs` | 子タスク一覧 |
| HelpWidget | `ui/components/help.rs` | ヘルプオーバーレイ |
| InputWidget | `ui/components/input.rs` | テキスト入力 |

### レイアウト構成

```
┌─────────────────────────────────────────────────┐
│ Header (3行)                                    │
├──────────────────┬──────────────────────────────┤
│ AgentTree        │ PanePreview                  │
│ (サイドバー)      │ (コンテンツエリア)            │
│ 15-70%幅         │                              │
├──────────────────┴──────────────────────────────┤
│ Footer (1行)                                    │
└─────────────────────────────────────────────────┘
```

---

## 設計パターン

| パターン | 適用箇所 | 説明 |
|---------|---------|------|
| Registry | ParserRegistry | パーサーの動的ディスパッチ |
| Trait Object | dyn AgentParser | ポリモーフィズム |
| State Machine | AgentStatus enum | 有限状態管理 |
| Observer | mpsc channel | Monitor→UIの通知 |
| Hysteresis | STATUS_HYSTERESIS_MS | 2秒間の状態保持でフリッカー防止 |
| Builder | MonitoredAgent::new() | オブジェクト構築 |

---

## スレッドモデル

```
┌──────────────────────────────────────────────────────────┐
│ Main Thread                                              │
│ • UIイベントループ (Ratatui/Crossterm)                    │
│ • キーボード/マウスイベント処理                            │
│ • 描画                                                   │
└──────────────────────────────────────────────────────────┘
                         ▲
                         │ mpsc channel (MonitorUpdate)
                         │
┌──────────────────────────────────────────────────────────┐
│ Background Thread (Tokio)                                │
│ • MonitorTask: tmuxポーリング                            │
│ • SystemStatsCollector: リソース監視                     │
└──────────────────────────────────────────────────────────┘
```

---

## パフォーマンス最適化

1. **プロセスキャッシュ**: ProcessTreeCacheで500ms毎に一括更新
2. **遅延Regex**: パーサー初期化時にコンパイル、再利用
3. **コンテンツ切り詰め**: `safe_tail()`で直近行のみ検索
4. **統計スロットリング**: システム統計は1秒間隔で更新
5. **効率的なツリー構築**: BTreeMapでソート済みセッション/ウィンドウ管理

---

## 依存クレート

| クレート | バージョン | 用途 |
|---------|-----------|------|
| ratatui | 0.29 | TUIフレームワーク |
| crossterm | 0.28 | ターミナル制御 |
| tokio | 1.x | 非同期ランタイム |
| clap | 4.x | CLI引数解析 |
| regex | 1.x | パターンマッチング |
| serde | 1.x | シリアライズ |
| toml | 0.8 | 設定ファイル |
| chrono | 0.4 | 日時処理 |
| dirs | 5.x | 設定パス解決 |
| parking_lot | 0.12 | 高性能Mutex |
| unicode-width | 0.1 | 文字幅計算 |
| sysinfo | 0.32 | システムリソース監視 |
| anyhow | 1.x | エラーハンドリング |
| tracing | 0.1 | ロギング |

---

## 設定ファイル

パス: `~/.config/tmuxcc/config.toml` (Linux) / `~/Library/Application Support/tmuxcc/config.toml` (macOS)

```toml
poll_interval_ms = 500
capture_lines = 100

# カスタムエージェントパターン (オプション)
[[agent_patterns]]
pattern = "my-custom-agent"
agent_type = "CustomAgent"
```

---

## 新しいエージェントの追加方法

1. `src/parsers/`に新しいパーサーファイルを作成
2. `AgentParser` traitを実装
3. `ParserRegistry::new()`に追加
4. `AgentType` enumに新しい種類を追加

```rust
// 例: src/parsers/my_agent.rs
pub struct MyAgentParser;

impl AgentParser for MyAgentParser {
    fn matches(&self, detection_strings: &[&str]) -> bool {
        detection_strings.iter().any(|s| s.contains("my-agent"))
    }

    fn parse_status(&self, content: &str) -> AgentStatus {
        // 実装
    }

    // ...
}
```

---

## キーバインド

| キー | アクション |
|-----|----------|
| `j` / `↓` | 次のエージェント |
| `k` / `↑` | 前のエージェント |
| `Space` | 選択トグル |
| `y` | 承認 |
| `n` | 拒否 |
| `a` | 全て承認 |
| `f` | ペインにフォーカス |
| `s` | サブエージェントログ表示 |
| `t` | TODO/Tools表示トグル |
| `h` / `?` | ヘルプ |
| `q` | 終了 |
| `1-9` | 番号選択 |
