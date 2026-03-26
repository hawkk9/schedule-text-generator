# リサーチ＆設計決定ログ

---
**Purpose**: ディスカバリフェーズの調査結果・アーキテクチャ検討・設計根拠を記録する。

---

## Summary

- **Feature**: `schedule-text-generator`
- **Discovery Scope**: Simple Addition（グリーンフィールド・フロントエンド単体アプリ）
- **Key Findings**:
  - フロントエンド専用のシングルページアプリであり、バックエンド・データストアは不要
  - 既存プロトタイプ（`index.html`）がバニラJS/HTMLで実装済み。フレームワーク導入の効果が薄く、依存ゼロのまま完結できる
  - 状態は `Map<dateKey, ScheduleEntry>` のインメモリ管理で十分。永続化は対象外

## Research Log

### フロントエンドフレームワーク選定

- **Context**: フレームワーク（React/Vue/Svelte）を使うべきか検討
- **Sources Consulted**: 既存プロトタイプ `index.html` のコード分析
- **Findings**:
  - コンポーネント数は 5 以下、状態もシンプルな Map 1 つ
  - ビルドツールなしで動作するバニラJS実装でプロトタイプが完成している
  - フレームワーク導入によるバンドルサイズ・ビルド設定コストがメリットを上回る
- **Implications**: バニラHTML/CSS/JS（型定義はJSDocで補完）を採用する

### 時刻入力UIの設計

- **Context**: 分として00/30のみを選択肢とし、その他は手動入力する要件（要件2.3）
- **Findings**:
  - `<input type="time">` はネイティブUIが環境依存で統一できない
  - 主要ライブラリ（flatpickr等）の `minuteIncrement` は手動入力フォールバックを持たない
  - 時セレクト＋分セレクト（00/30/その他）＋条件付きテキスト入力のカスタムUIが最適
- **Implications**: ライブラリ非依存のカスタム `TimePickerWidget` を設計する

### テキスト生成フォーマット

- **Context**: `2026/03/26(木) 20:00~21:00` 形式の確定
- **Findings**:
  - 曜日は `Date.prototype.getDay()` で 0（日）〜6（土）を取得し、`['日','月','火','水','木','金','土']` でマッピング
  - 月・日は ゼロパディング必須（`String.padStart(2,'0')`）
  - 複数エントリーは dateKey（YYYY-MM-DD）の辞書順昇順でソート可能
- **Implications**: `TextFormatter` は純粋関数として切り出し、単体テスト容易にする

### クリップボードAPI

- **Context**: 要件4.2〜4.4のコピー機能
- **Findings**:
  - `navigator.clipboard.writeText()` は HTTPS または localhost でのみ動作
  - フォールバック: `document.createRange()` + `window.getSelection()` でテキスト選択状態にしてユーザーに手動コピーを促す
- **Implications**: try/catch で両方のパスを実装する

## Architecture Pattern Evaluation

| Option | 説明 | 強み | リスク・制限 | 備考 |
|--------|------|------|------------|------|
| バニラHTML/JS | ビルドツールなしの単一ファイル | 依存ゼロ・即時動作・プロトタイプ資産流用可 | 大規模化時にメンテが難しい | 現スコープに最適 |
| Vue 3 (CDN) | CDN経由でフレームワーク利用 | リアクティビティが宣言的 | CDN依存・バンドル不要だが学習コスト追加 | 将来拡張時の候補 |
| React + Vite | フルビルド構成 | エコシステム豊富 | ビルド設定・依存管理が必要 | 現スコープには過剰 |

**選択**: バニラHTML/CSS/JS（単一ファイル構成）

## Design Decisions

### Decision: 単一HTMLファイル構成

- **Context**: デプロイ・配布の簡易性を最優先にする小規模ツール
- **Alternatives Considered**:
  1. モジュール分割＋ビルドツール — メンテ性は上がるがビルド設定が必要
  2. 単一HTMLファイル — ファイルを開くだけで動作
- **Selected Approach**: 単一HTMLファイル。CSS・JSをインラインで管理
- **Rationale**: ユーザーがサーバーなしでローカルに開けること、デプロイ先を選ばないこと
- **Trade-offs**: コードが大きくなるとファイル内ナビゲーションが難しい
- **Follow-up**: 500行を超えたらモジュール分割を検討

### Decision: インメモリ状態管理（Map）

- **Context**: 選択日付と時刻データの管理
- **Alternatives Considered**:
  1. グローバル変数の配列 — 並び替えが面倒
  2. `Map<dateKey, ScheduleEntry>` — キー操作・重複排除が容易
- **Selected Approach**: `Map<string, ScheduleEntry>`（キー: `YYYY-MM-DD`）
- **Rationale**: 日付の選択・解除トグルをO(1)で実現、ソートはキー昇順で可能
- **Trade-offs**: ページリロードで状態消失（要件範囲外）

## Risks & Mitigations

- クリップボードAPI非対応環境 — テキスト選択フォールバックで対応（要件4.4）
- 終了時刻≤開始時刻の不正入力 — エントリーごとにバリデーションし、該当行を出力から除外（要件2.4）
- モバイルでの時刻入力UX — セレクトボックス系UIはモバイルでも標準的に動作するため問題なし

## References

- Clipboard API MDN — `navigator.clipboard.writeText()` の環境制約
- `Date.prototype.getDay()` MDN — 曜日取得の仕様確認
