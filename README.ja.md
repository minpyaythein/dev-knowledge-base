# dev-knowledge-base — 1ファイルに1つ、実戦で得たコーディングの学び

<div align="center">

[![English](https://img.shields.io/badge/README-English-lightgrey?style=for-the-badge)](README.md)
[![日本語](https://img.shields.io/badge/README-日本語-2563eb?style=for-the-badge)](README.ja.md)

</div>

実際のプロジェクトから抽出したコーディングの知見を集めた個人ナレッジベース。各 markdown ファイルは、プラットフォームの癖、競合状態、実際に役立った設計パターンなど、自明でない学びを**ちょうど1つ**だけ記録する。数ヶ月後にプロジェクトの文脈が一切なくても理解できるように書いてある。

---

## なぜ続けているか

微妙な競合状態のデバッグやプラットフォームの制約の発見には大きなコストがかかる。半年後に別のプロジェクトで同じことを再発見するのは純粋な無駄だ。だから作業セッションの終わりに、いま明確になったことを独立したノートとして抽出し、書き下ろす前に実際のコードと照合して検証している。

- **1ファイル1トピック** — 10個の疑問に曖昧に答えるのではなく、1つの疑問にきちんと答える。ファイル名だけで探せるようになる。
- **日記ではなく普遍的な記述** — 「今日学んだこと」ではなく「XはYだから動く」という真実を書く。出自のプロジェクトと一緒に腐らない。
- **記憶ではなく検証** — すべての主張は、読んだコードか観察した挙動に裏付けられる。曖昧な数値は推測せず省略する。

---

## ノートの形式

すべてのノートは同じ骨格に従い、1画面以内に収める:

```markdown
# タイトル — トピック名ではなく、洞察そのもの

1行の TL;DR。

## Why it matters
## How it works
## Example        （任意）
## Gotchas        （任意）
## See also       （任意 — 関連ノートへのリンク）
```

---

## ノート一覧

### Chrome 拡張機能（Manifest V3）

| ノート | 洞察 |
|---|---|
| [MV3 サービスワーカーの状態管理](mv3-service-worker-state-in-chrome-storage-session.md) | ランタイム状態はモジュール変数ではなく `chrome.storage.session` に置く — ワーカーは勝手に死ぬ。 |
| [30秒未満のタイマーは二経路で](chrome-alarms-sub-30s-settimeout-dual-path.md) | `chrome.alarms` は短い遅延で発火できない。`setTimeout` と定期アラームのバックストップを組み合わせる。 |
| [storage 用 Promise チェーン mutex](promise-chain-mutex-for-chrome-storage-rmw.md) | `chrome.storage` の並行 read-modify-write は5行の Promise チェーンで直列化する。 |
| [ランタイム状態の書き込みは単一箇所に](extension-background-single-writer-runtime-state.md) | ランタイム状態を変更するのは background だけ。UI コンテキストは書き込まずメッセージを送る。 |
| [注入してから再送するメッセージング](content-script-sendmessage-inject-fallback.md) | インストール前から開いていたページへの `tabs.sendMessage` は失敗する — スクリプトを注入して再送する。 |
| [孤立したコンテンツスクリプト](orphaned-content-script-safe-sendmessage.md) | 拡張機能のリロードは実行中のコンテンツスクリプトを孤立させる。`chrome.runtime` 呼び出しは try/catch で包む。 |
| [実行時の言語切り替え](chrome-i18n-runtime-language-toggle.md) | `chrome.i18n` はブラウザのロケールに固定される — 実行時切り替えには自前の辞書レイヤーが要る。 |

### Streamlit

| ノート | 洞察 |
|---|---|
| [`st.cache_resource` をサーバー状態に](st-cache-resource-as-process-global-server-state.md) | リソースキャッシュはプロセス全体で共有されるサーバーサイド状態としても使える。 |
| [iframe サンドボックスの（正当な）回避](streamlit-iframe-sandbox-parent-realm-injection.md) | `components.html` はページを操作できない iframe 内で動く — 親レルムにスクリプトを注入する。 |
| [rerun と reload とクライアント識別](streamlit-rerun-vs-reload-client-identity.md) | クライアント識別子は Python から設定する `st.query_params` に載せ、rerun も reload も生き延びさせる。 |

### Cloudflare・サーバーレス

| ノート | 洞察 |
|---|---|
| [cron の下限 + アプリ側ゲート](cloudflare-cron-floor-plus-app-gating.md) | 固定 cron 上でユーザー設定可能なスケジュールを実現する: 最短周期で発火し、コード側で間引く。 |
| [Worker 1つでフルスタック](single-worker-api-plus-static-spa.md) | API + SPA + cron を1つの Cloudflare Worker で配信するのが $0 フルスタック構成。 |

### 運用・信頼性

| ノート | 洞察 |
|---|---|
| [liveness とアプリの健全性は別物](healthz-edge-liveness-vs-in-app-error-monitoring.md) | プラットフォームのヘルスエンドポイントはホストの生存を示すだけで、アプリが動いている証明ではない — 両方の層を監視する。 |
| [Discord webhook の 403](discord-webhook-403-python-urllib-user-agent.md) | Discord は Python urllib のデフォルト User-Agent をブロックする — 自前の値を設定する。 |

### 設計・データパターン

| ノート | 洞察 |
|---|---|
| [アラートルールを純粋関数に](alert-rules-as-pure-decision-function.md) | アラート判定は状態遷移に対する純粋な決定関数としてモデル化し、リマインダーに上限を設ける。 |
| [追記専用スナップショット](append-only-snapshots-as-source-of-truth.md) | 追記専用スナップショットを唯一の情報源にし、それ以外はすべてビューとする。 |
| [金額は整数の最小単位で](money-as-integer-minor-units.md) | 金額は整数の最小単位（minor units）で保存し、ISO 4217 の通貨カラムを別に持つ。 |
| [抽出のカスケード](generic-product-extraction-cascade.md) | 商品データの抽出は、構造化ソースから推測ベースへと段階的にフォールバックする。 |
| [タイマーのフラッシュは整数単位で](timer-flush-whole-seconds-carry-remainder.md) | フラッシュのカーソルは `now()` ではなく計上した単位分だけ進める — さもないとタイマーが遅れていく。 |

### フロントエンド

| ノート | 洞察 |
|---|---|
| [ちらつかないダークモード](dark-mode-without-flash-inline-script.md) | 保存済みテーマは最初の描画前に、`<head>` 内のインラインスクリプトで適用する。 |

### LLM

| ノート | 洞察 |
|---|---|
| [バースト的なトークンストリームの平滑化](smoothing-bursty-llm-streams-reader-thread-pacer.md) | バースト的な LLM ストリームの平滑化には pull 型スロットルではなく、リーダースレッド + ペーサーが要る。 |

---

## ノートの追加方法

ノートは作業セッションの終わりに、自作の Claude Code `/learnings` スキルで記録する: いま明確になったことを1つ選び、すべての主張をコードと照合して検証し、狭いスコープの kebab-case ファイル名で書き下ろす。ハウスルール: 1ファイル1トピック、普遍的な記述、不確かなことに正直に、50行以内。
