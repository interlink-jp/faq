---
layout: default
permalink: /ai-debate-bot
---

# ローカル AI 3 体を Slack で議論させる

API 課金ゼロ、Mac 1 台で動く、AI 同士のディベート Bot を作ります。

`/debate AI に重要な経営判断を任せていいか` と Slack で打つだけで、3 体の AI が議論を始めます。クラウド API を使わないので、何回議論させても追加コストはかかりません。

社内ブレストの種、新企画の壁打ち相手、息抜きのエンタメとして、用途は自由です。

---

## なぜこれを作るのか

「AI 同士に議論させる」は、ChatGPT や Claude の API を3つ並べれば作れます。ただ、AI 同士の会話は **会話履歴がどんどん蓄積される** という性質上、トークン課金との相性が最悪です。1 議論で 200〜500 円かかることもあります。

ローカル LLM を使えば、何ターン回しても電気代だけ。Apple Silicon Mac なら 16GB のメモリがあれば動きます。

社外秘の議題でも安心して投げられるのもメリットです。

---

## 完成形のイメージ

Slack のチャンネルで:

```
/debate リモートワークは生産性を上げるか
```

と打つと、スレッドの中で 3 体の AI が順番に発言します。

- **gemma-san** 🌱（慎重派 / Gemma 2 9B）
- **qwen-san** ⚡（革新派 / Qwen 2.5 7B）
- **llama-san** ⚖️（中立派 / Llama 3.1 8B）

それぞれが違う立場から意見を述べ、3 ターン（計 9 発言）で議論を完結させます。1 議論あたり 3〜5 分。

---

## 動作環境

| 項目 | 必要要件 |
|---|---|
| ハードウェア | Apple Silicon Mac（M1 以降） |
| メモリ | 16GB / 32GB / 64GB（本記事は3パターン対応） |
| OS | macOS 14 以降 |
| Slack | ワークスペースの管理者権限（または App インストール権限） |
| 知識 | ターミナル操作の経験があれば理想、なくても本記事の通りに進めれば動きます |

所要時間: 約 90 分（モデルダウンロード時間を除く）

ランニングコスト: 0 円（電気代を除く）

---

## ターミナルとは

この記事では「ターミナル」というアプリを使います。Mac に最初から入っている、文字でコマンドを打つ画面のことです。

**起動方法**: アプリケーション → ユーティリティ → ターミナル.app を開く（または Spotlight で「ターミナル」と検索）。

**この記事の使い方**: 本文中のグレーの枠の中（例: `brew --version` のような表記）が、ターミナルに入力するコマンドです。

```bash
このようなブロックがコマンドです
```

**操作の基本**: コマンドをコピーして、ターミナルに貼り付けて、**Enter キー**を押します。それだけです。複数行のコマンドでも、まとめてコピー&貼り付けて Enter で実行できます。

---

## 全体像

```
  Slack チャンネル
    ├─ @gemma-san  ← Gemma（慎重派）
    ├─ @qwen-san   ← Qwen（革新派）
    └─ @llama-san  ← Llama（中立派）
            ↑
       Python ボット (Slack Bolt + Socket Mode)
            ↓
       Ollama (http://localhost:11434)
            ↓
       Mac 内でモデル稼働
```

Socket Mode を使うため、外部公開エンドポイント不要で、自分の Mac から直接 Slack に接続できます。

---

## Phase 0: 事前準備チェック

まずシェルの種類を確認します。後で設定ファイルを書き込む先が変わるので、この情報はメモしておいてください。

```bash
echo $SHELL
```

結果が `/bin/bash` なら **bash**、`/bin/zsh` なら **zsh** です。新しめの Mac ではほぼ zsh のはずです。

続いて、必要なソフトが入っているか確認します。

```bash
sw_vers
brew --version
python3 --version
```

`brew --version` で「command not found」と出たら、Homebrew をインストールします:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Python は 3.9 以上であれば動きますが、3.11 以上を推奨します。

---

## Phase 1: Ollama のインストールとモデル準備

### 1-1. Ollama インストール

```bash
brew install ollama
```

### 1-2. Ollama 起動

```bash
brew services start ollama
```

起動確認:

```bash
curl http://localhost:11434/api/tags
```

`{"models":[]}` と返れば OK です。

### 1-3. お使いの Mac に合わせてモデルを選ぶ

メモリサイズによって、選ぶモデルが変わります。**該当するパターンの 3 つだけダウンロードしてください**。

#### パターン A: メモリ 32GB 以上 → 7〜9B 級モデル（推奨）

議論の質が高く、論理的な応答が期待できます。

```bash
ollama pull gemma2:9b      # 約 5.5GB
ollama pull qwen2.5:7b     # 約 4.7GB
ollama pull llama3.1:8b    # 約 4.9GB
```

#### パターン B: メモリ 16GB → 3〜4B 級モデル

応答のテンポが良く、メモリへの負担も軽いです。議論の深みは A よりやや劣ります。

```bash
ollama pull gemma3:4b      # 約 3.3GB
ollama pull qwen2.5:3b     # 約 2.0GB
ollama pull phi4-mini      # 約 2.5GB
```

#### パターン C: メモリ 64GB 以上 → 14B 級でもう一段上の質に

応答は遅めですが、複雑な議論にも対応できます。

```bash
ollama pull qwen2.5:14b    # 約 9GB
ollama pull gemma2:9b      # 約 5.5GB
ollama pull phi4:14b       # 約 9GB
```

それぞれのパターンで合計約 8〜30GB のダウンロードです。回線速度次第で 30〜90 分かかります。

### 1-4. 動作確認

ダウンロードしたモデルが日本語で動くかテストします。**自分のパターンに合わせて**コマンド内のモデル名を読み替えてください。

```bash
ollama run gemma2:9b "こんにちは。日本語で短く自己紹介してください。"
```

応答を確認したら **Ctrl+D** で終了。同じく残り 2 つのモデルもテスト。3 つとも日本語で応答すれば Phase 1 完了です。

### 1-5. メモリ常駐の設定

デフォルトでは使っていないモデルが 5 分でメモリから降りてしまい、次回応答が遅くなります。長時間常駐させるための設定です。

**Phase 0 で確認したシェルに合わせて**、以下のいずれかを実行してください。

bash の場合:
```bash
echo 'export OLLAMA_KEEP_ALIVE=24h' >> ~/.bash_profile
source ~/.bash_profile
```

zsh の場合:
```bash
echo 'export OLLAMA_KEEP_ALIVE=24h' >> ~/.zshrc
source ~/.zshrc
```

確認:
```bash
echo $OLLAMA_KEEP_ALIVE
```

`24h` と表示されれば成功です。

---

## Phase 2: Slack App を 3 つ作成

各 AI ごとに別の Slack App を作ります。アイコンと名前で個性が出るので、楽しい工程です。

### 2-1. Slack API ダッシュボードへ

ブラウザで <https://api.slack.com/apps> を開き、自分のワークスペースにログインします。

### 2-2. 1 つ目: gemma-san を作成

1. **Create New App** → **From scratch** をクリック
2. App Name に `gemma-san` と入力、Workspace は自分のワークスペースを選択 → **Create App**
3. 左メニューの **App Manifest** をクリック → **JSON タブ** に切り替え → 既存内容を全削除して以下を貼り付け:

```json
{
    "display_information": {
        "name": "gemma-san",
        "description": "AI Debate Bot - 慎重派",
        "background_color": "#4A8CB7"
    },
    "features": {
        "bot_user": {
            "display_name": "gemma-san",
            "always_online": true
        },
        "slash_commands": [
            {
                "command": "/debate",
                "description": "AI 3体に議論させる",
                "usage_hint": "[議題]",
                "should_escape": false
            }
        ]
    },
    "oauth_config": {
        "scopes": {
            "bot": [
                "app_mentions:read",
                "channels:history",
                "chat:write",
                "chat:write.customize",
                "commands"
            ]
        }
    },
    "settings": {
        "event_subscriptions": {
            "bot_events": [
                "app_mention",
                "message.channels"
            ]
        },
        "interactivity": {
            "is_enabled": true
        },
        "org_deploy_enabled": false,
        "socket_mode_enabled": true,
        "token_rotation_enabled": false
    }
}
```

4. **Save Changes** をクリック
5. 上部に出る黄色い注意「App level token missing」内の **Click here to generate** を押し、Token Name に `gemma-socket` と入力、Scope に `connections:write` を選んで **Generate**
6. 表示された `xapp-` で始まるトークンをコピー保存（これは**一度しか表示されません**。テキストエディタなどに必ず貼り付けて保存）
7. 左メニュー **Install App** → **Install to Workspace** → **Allow**
8. 表示された `xoxb-` で始まる Bot Token をコピー保存

### 2-3. 2 つ目: qwen-san を作成

**Your Apps** に戻って **Create New App** → **From scratch** で `qwen-san` を作成し、Manifest に以下を貼り付け:

```json
{
    "display_information": {
        "name": "qwen-san",
        "description": "AI Debate Bot - 革新派",
        "background_color": "#A04B7C"
    },
    "features": {
        "bot_user": {
            "display_name": "qwen-san",
            "always_online": true
        }
    },
    "oauth_config": {
        "scopes": {
            "bot": [
                "app_mentions:read",
                "channels:history",
                "chat:write",
                "chat:write.customize"
            ]
        }
    },
    "settings": {
        "event_subscriptions": {
            "bot_events": [
                "app_mention",
                "message.channels"
            ]
        },
        "interactivity": {
            "is_enabled": true
        },
        "org_deploy_enabled": false,
        "socket_mode_enabled": true,
        "token_rotation_enabled": false
    }
}
```

slash_commands と commands スコープが無いことに注意。スラッシュコマンドは gemma-san が司令塔として一括で受けます。

App Token (`xapp-`) と Bot Token (`xoxb-`) を取得して保存。

### 2-4. 3 つ目: llama-san を作成

同じく `llama-san` を作り、Manifest:

```json
{
    "display_information": {
        "name": "llama-san",
        "description": "AI Debate Bot - 中立派",
        "background_color": "#5B8C5A"
    },
    "features": {
        "bot_user": {
            "display_name": "llama-san",
            "always_online": true
        }
    },
    "oauth_config": {
        "scopes": {
            "bot": [
                "app_mentions:read",
                "channels:history",
                "chat:write",
                "chat:write.customize"
            ]
        }
    },
    "settings": {
        "event_subscriptions": {
            "bot_events": [
                "app_mention",
                "message.channels"
            ]
        },
        "interactivity": {
            "is_enabled": true
        },
        "org_deploy_enabled": false,
        "socket_mode_enabled": true,
        "token_rotation_enabled": false
    }
}
```

App Token と Bot Token を取得して保存。

### 2-5. トークンを 6 つ揃える

最終的に手元に 6 つのトークンが揃います。テキストエディタなどにこのような形でメモしておいてください:

```
gemma-san
  Bot Token: xoxb-...
  App Token: xapp-...

qwen-san
  Bot Token: xoxb-...
  App Token: xapp-...

llama-san
  Bot Token: xoxb-...
  App Token: xapp-...
```

`xoxb-` が **Bot Token**、`xapp-` が **App Token** です。混ざりやすいので注意してください。

---

## Phase 3: ボット本体の実装

### 3-1. プロジェクトディレクトリを作る

ターミナルで以下を実行（コピーして Enter）:

```bash
mkdir -p ~/ai-debate-bot
cd ~/ai-debate-bot
python3 -m venv venv
source venv/bin/activate
pip install slack-bolt slack-sdk requests python-dotenv
```

最後の `pip install` は数十秒〜1分ほどかかります。プロンプトの先頭に `(venv)` が付くようになれば、Python の仮想環境に入っている状態です。

### 3-2. 環境変数ファイル(.env)を作る

トークンを保存するファイルを作成します。

ターミナルで以下を実行:

```bash
nano ~/ai-debate-bot/.env
```

`nano` というシンプルなエディタが開きます。以下のテンプレートを貼り付けて、`xxx...` の部分を **Phase 2 で保存した実際のトークン**で置き換えてください:

```
GEMMA_BOT_TOKEN=xoxb-xxx
GEMMA_APP_TOKEN=xapp-xxx

QWEN_BOT_TOKEN=xoxb-xxx
QWEN_APP_TOKEN=xapp-xxx

LLAMA_BOT_TOKEN=xoxb-xxx
LLAMA_APP_TOKEN=xapp-xxx

OLLAMA_URL=http://localhost:11434
MAX_TURNS_PER_AI=3
```

入力できたら、保存して終了します:

1. **Control + O**（オー）を押す → 画面下に確認が出るので **Enter**（保存）
2. **Control + X** を押す → nano が終了

確認:

```bash
cat ~/ai-debate-bot/.env
```

6 行のトークン行が正しく書かれているか目視で確認してください。

> **重要**: `.env` ファイルは絶対に Git に commit したり、SNS にスクショで載せたりしないでください。トークンが漏れると bot が乗っ取られます。

### 3-3. 環境変数ファイルが正しく書けているか自動チェック

トークンの prefix（`xoxb-` と `xapp-`）が逆になっていると、よくあるトラブルになります。確認用のコマンドを実行します:

```bash
grep "BOT_TOKEN" ~/ai-debate-bot/.env | grep -v "xoxb-" && echo "❌ BOT_TOKEN が xoxb- で始まっていません" || echo "✅ BOT_TOKEN は OK"
grep "APP_TOKEN" ~/ai-debate-bot/.env | grep -v "xapp-" && echo "❌ APP_TOKEN が xapp- で始まっていません" || echo "✅ APP_TOKEN は OK"
```

両方とも ✅ が出れば OK です。

### 3-4. メインスクリプト（bot.py）を作る

このファイルが bot の本体です。長いコードですが、以下のコマンドを **そのまま全部コピーしてターミナルに貼り付けて Enter** すれば、ファイルが一気に作成されます。

```bash
cat > ~/ai-debate-bot/bot.py << 'PYEOF'
import os
import re
import threading
import time
import requests
from dotenv import load_dotenv
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler
from slack_sdk import WebClient

load_dotenv(os.path.expanduser("~/ai-debate-bot/.env"))

OLLAMA_URL = os.environ["OLLAMA_URL"]
MAX_TURNS = int(os.environ.get("MAX_TURNS_PER_AI", "3"))

ANTI_ROLLEPLAY_RULE = (
    "\n\n【厳守ルール】\n"
    "1. あなたが書いていいのは、あなた自身の発言「1人分」だけです。\n"
    "2. 他の登場人物のセリフを絶対に書いてはいけません。\n"
    "3. 「【役名】」のような見出しは絶対に書かないでください。\n"
    "4. 「〜さん:」「〜様:」のような自己紹介ラベルも書かないでください。\n"
    "5. あなたの発言を1回書いたら、すぐに発言を終了してください。"
)

AGENTS = [
    {
        "name": "gemma-san",
        "model": "gemma2:9b",
        "bot_token": os.environ["GEMMA_BOT_TOKEN"],
        "app_token": os.environ["GEMMA_APP_TOKEN"],
        "persona": "あなたは慎重派の論者「ジェマ」です。リスクや前提を丁寧に検討してから意見を述べます。日本語で200字以内で返答してください。他の論者の意見も踏まえて議論を深めてください。",
        "icon_emoji": ":seedling:",
    },
    {
        "name": "qwen-san",
        "model": "qwen2.5:7b",
        "bot_token": os.environ["QWEN_BOT_TOKEN"],
        "app_token": os.environ["QWEN_APP_TOKEN"],
        "persona": "あなたは革新派の論者「クウェン」です。新しい可能性や大胆な提案を積極的に出します。日本語で200字以内で返答してください。他の論者の意見も踏まえて議論を深めてください。",
        "icon_emoji": ":zap:",
    },
    {
        "name": "llama-san",
        "model": "llama3.1:8b",
        "bot_token": os.environ["LLAMA_BOT_TOKEN"],
        "app_token": os.environ["LLAMA_APP_TOKEN"],
        "persona": "あなたは中立派の論者「ラマ」です。両論を整理し、論点を明確にする役回りです。日本語で200字以内で返答してください。他の論者の意見も踏まえて議論を深めてください。",
        "icon_emoji": ":scales:",
    },
]

gemma_app = App(token=AGENTS[0]["bot_token"])

for agent in AGENTS:
    agent["client"] = WebClient(token=agent["bot_token"])


def call_ollama(model, system_prompt, history):
    full_prompt = system_prompt + ANTI_ROLLEPLAY_RULE
    messages = [{"role": "system", "content": full_prompt}] + history
    resp = requests.post(
        f"{OLLAMA_URL}/api/chat",
        json={
            "model": model,
            "messages": messages,
            "stream": False,
            "options": {"num_ctx": 4096, "temperature": 0.9},
        },
        timeout=180,
    )
    resp.raise_for_status()
    return resp.json()["message"]["content"].strip()


def clean_response(text, all_names):
    for name in all_names:
        text = re.sub(rf"^[【\[]?{re.escape(name)}[】\]]?\s*[::]?\s*", "", text, count=1)
    earliest_cut = len(text)
    for name in all_names:
        for pat in [rf"【{re.escape(name)}】", rf"\[{re.escape(name)}\]", rf"\n{re.escape(name)}\s*[::]"]:
            m = re.search(pat, text)
            if m and m.start() < earliest_cut:
                earliest_cut = m.start()
    return text[:earliest_cut].strip()


def post_as(agent, channel, text, thread_ts=None):
    agent["client"].chat_postMessage(
        channel=channel,
        text=text,
        username=agent["name"],
        icon_emoji=agent["icon_emoji"],
        thread_ts=thread_ts,
    )


def run_debate(channel, topic, thread_ts):
    history = []
    history.append({
        "role": "user",
        "content": f"議題: {topic}\n\nこの議題について各論者で議論してください。発言は自分の分だけ書き、他者のセリフは書かないでください。"
    })

    all_names = [a["name"] for a in AGENTS]

    for turn in range(MAX_TURNS):
        for agent in AGENTS:
            try:
                print(f"[Turn {turn+1}] {agent['name']} thinking...")
                raw = call_ollama(agent["model"], agent["persona"], history)
                response = clean_response(raw, all_names)
                if not response.strip():
                    response = raw
                post_as(agent, channel, response, thread_ts=thread_ts)
                history.append({
                    "role": "user",
                    "content": f"# {agent['name']} の発言:\n{response}",
                })
                time.sleep(2)
            except Exception as e:
                print(f"Error: {agent['name']}: {e}")

    post_as(AGENTS[0], channel, "💬 議論を終了しました。", thread_ts=thread_ts)


@gemma_app.command("/debate")
def handle_debate(ack, command, client):
    ack()
    topic = command.get("text", "").strip()
    channel = command["channel_id"]

    if not topic:
        client.chat_postMessage(
            channel=channel,
            text="議題を指定してください。例: `/debate リモートワークは生産性を上げるか`"
        )
        return

    parent = client.chat_postMessage(
        channel=channel,
        text=f"🎤 *議題*: {topic}\n3体の AI が議論します...",
    )
    thread_ts = parent["ts"]

    threading.Thread(
        target=run_debate,
        args=(channel, topic, thread_ts),
        daemon=True,
    ).start()


@gemma_app.event("message")
def handle_message_events(body, logger):
    pass


if __name__ == "__main__":
    print("AI Debate Bot を起動します...")
    print(f"参加 AI: {[a['name'] for a in AGENTS]}")
    print(f"各 AI 最大ターン数: {MAX_TURNS}")
    handler = SocketModeHandler(gemma_app, AGENTS[0]["app_token"])
    handler.start()
PYEOF
```

「`PYEOF`」の行までを **すべて含めて** コピーしてください。最後の `PYEOF` までが 1 つのコマンドです。

実行後、ファイルが正しく作成されたか確認:

```bash
wc -l ~/ai-debate-bot/bot.py
```

`130` 前後の数字が出れば成功です。

### 3-5. 16GB Mac の方は model 名を書き換える

Phase 1-3 で **パターン B（16GB）** を選んだ方は、bot.py の中の model 名を書き換える必要があります。

```bash
sed -i '' 's|"gemma2:9b"|"gemma3:4b"|; s|"qwen2.5:7b"|"qwen2.5:3b"|; s|"llama3.1:8b"|"phi4-mini"|' ~/ai-debate-bot/bot.py
```

確認:
```bash
grep '"model"' ~/ai-debate-bot/bot.py
```

`gemma3:4b`、`qwen2.5:3b`、`phi4-mini` が表示されれば OK。

（パターン A・C の方はそのままで大丈夫です）

### 3-6. 起動

```bash
cd ~/ai-debate-bot
source venv/bin/activate
python bot.py
```

以下のメッセージが出れば成功です:

```
AI Debate Bot を起動します...
参加 AI: ['gemma-san', 'qwen-san', 'llama-san']
各 AI 最大ターン数: 3
⚡️ Bolt app is running!
```

**このターミナルウィンドウは閉じないでください**。閉じると bot が止まります。Slack 操作は別のウィンドウやアプリで行います。

---

## Phase 4: Slack で動作確認

### 4-1. ボットをチャンネルに招待

Slack で任意のチャンネル（新規に `#ai-debate` などを作るのがおすすめ）を開き、3 つの bot を招待します:

```
/invite @gemma-san
/invite @qwen-san
/invite @llama-san
```

### 4-2. 議論開始

```
/debate リモートワークは生産性を上げるか
```

スレッド内で 3 体が順番に発言します。各 AI が 3 ターン発言するので、合計 9 発言。1 議論で約 3〜5 分かかります（7〜9B 級の場合）。

応答開始まで 15〜30 秒待つことがあります。Slack のリアルタイムチャットというより、「ゆったりとした議論」の体感です。

---

## おすすめの議題

対立軸が明確な議題ほど議論が盛り上がります。

```
/debate リモートワークと出社、どちらが生産的か
/debate AI に重要な経営判断を任せていいか
/debate 終身雇用は維持すべきか
/debate 副業解禁は本業を蝕むか
/debate 生成 AI に著作権を認めるべきか
/debate ホームページを人間用に作る時代は終わったか
```

最後の議題は、Interlink が 2026 年 4 月から実践している「人間用ホームページをやめました」を AI に議論させる、自社批評の試みです。よかったら試してみてください。

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `curl http://localhost:11434/api/tags` が応答しない | Ollama が起動していない | `brew services restart ollama` |
| Slack に発言が出ない | Bot がチャンネルに招待されていない | `/invite @bot名` |
| `not_in_channel` エラー | 同上 | 同上 |
| 応答が極端に遅い | モデルがメモリから降りた | `OLLAMA_KEEP_ALIVE=24h` 設定を再確認 |
| `Invalid auth` エラー | トークンの取り違え | `.env` の Bot Token と App Token が逆になっていないか確認（Phase 3-3 のチェック実行） |
| 文字化け・英語混じり | モデルが日本語苦手 | bot.py 内の persona の最後に「必ず日本語で」を追記 |
| `command not found: brew` | Homebrew が未インストール | Phase 0 の Homebrew インストールから実行 |

### Bot Token と App Token の違い

混同しやすいので注意:

- **Bot Token (`xoxb-` で始まる)**: bot として発言するためのトークン → `*_BOT_TOKEN` に入れる
- **App Token (`xapp-` で始まる)**: Socket Mode の接続用トークン → `*_APP_TOKEN` に入れる

---

## 拡張アイデア

シンプル版が動いたら、以下のような拡張が楽しめます。

### 1. 形式を変える

3 体の役割を変えるだけで、まったく違う体験になります。

- **法廷バトル**: 検察 / 弁護 / 裁判官の役を割り当て、最終ターンで判決を出す
- **インタビュー形式**: 1 体を質問者、2 体を当事者にして、矛盾を突きながら掘り下げる
- **偉人憑依**: 各 AI に「あなたは Steve Jobs です」「あなたは ドラッカー です」のように歴史上の人物を演じさせる
- **哲学カフェ**: 対立ではなく協力関係にして、議題を一緒に深掘りさせる

### 2. ロールジャック対策の強化

AI が他のキャラのセリフまで書いてしまう「ロールジャック」現象が起きることがあります。本記事のコードには簡易対策（`clean_response` 関数）を入れていますが、persona の指示を強化したり、最終出力に正規表現フィルタをかけるなど、改善の余地があります。

### 3. 役割を最終ターンだけ変える

法廷型のように「途中までは質問だけ、最後だけ判決を出す」という時系列の役割変化を実装する場合、ターン番号を見て persona を切り替える仕組みが必要です。

### 4. 人間の参加を取り込む

スレッド内で人間が発言したら、それも履歴に取り込んで AI が応答するように改造すれば、人間 + AI の混合議論が可能です。

### 5. 議事録の自動保存

議論ログを Markdown で書き出して社内 Wiki に蓄積する、という運用も面白いです。

### 6. モデルの組み合わせ

日本語特化モデル（ELYZA、Swallow など）や、より大きい 14B 級モデルに置き換えると、議論の質感が変わります。

---

## なぜローカル LLM を選ぶか

冒頭に書いた通り、AI 同士の議論は会話履歴がどんどん蓄積される性質があり、クラウド API のトークン課金との相性が悪いです。

具体的なコスト試算: 3 体の AI が議題について 3 ターンずつ議論する場合、各 AI は毎回過去の全履歴を入力として受け取ります。1 議論で累積入力トークンは数十万トークン、合計コストは 1 議論あたり 200〜500 円になることもあります。

ローカル LLM の利点:

- 何ターンでも何議論でも追加コストゼロ
- データが外部に出ないため、機密性の高い議題も扱える
- 動作が安定している（レート制限を気にしなくていい）

欠点は応答速度とモデル品質ですが、議論の楽しさを優先するなら十分実用的です。

---

## 関連記事

Interlink では、ローカル LLM や AI ツールの活用について継続的に記事を公開しています。本記事と合わせて読むと理解が深まります。

- Claude Code 入門: [interlink-jp.github.io/faq/](https://interlink-jp.github.io/faq/)
- AI-Private（ローカル LLM ホスティング）について

---

## まとめ

ローカル LLM 3 体を Slack に住まわせて、好きなだけ議論させる仕組みが完成しました。

- 初期構築は 90 分程度
- ランニングコストはゼロ
- 議題を変えるだけで、毎回違う体験ができる

社内コミュニケーションのアクセントとしても、ブレストの種としても使えます。ぜひ自分の好みに合わせてカスタマイズしてみてください。

---

本記事のコードは自由に利用・改変いただけます。動作については自己責任でお願いします。
