# MGscore モバイルUI リデザイン + QRコード

## Overview

`index_iOS.html` をCalmi (CalMG) のデザインシステムに合わせてリニューアル。テーブルベースのUIを問診フロー型（1問ずつ→自動進行）に刷新し、結果テキストのQRコード生成を追加する。完全オフライン・単一HTMLで完結。

## Base File

`index_iOS.html`（MG-ADL + MGC 2タブ、iOS対応済み）

## Design System — Calmi Tokens (Web CSS変換)

### Colors
```css
--mg-primary: #4A9B3F;       /* Calmi mgPrimary */
--mg-primary-light: rgba(74, 155, 63, 0.1);
--mg-primary-border: rgba(74, 155, 63, 0.3);
--mg-bg: #F5F7F2;            /* Calmi mgBackground */
--mg-surface: #FFFFFF;        /* Calmi mgSurface */
--mg-text: #1c1c1e;
--mg-text-grey: #63666A;      /* Calmi brandGrey */
--mg-divider: #E5E5EA;
--mg-error: #DC2626;          /* Calmi mgError */
--mg-warning: #D97706;        /* Calmi mgWarning */
--mg-success: #2D8A4E;        /* Calmi mgSuccess */
```

### Spacing & Sizing
```css
--radius-card: 16px;          /* Calmi corner.card */
--radius-medium: 12px;        /* Calmi corner.medium */
--radius-small: 8px;
--shadow-card: 0 2px 8px rgba(0, 0, 0, 0.06);
--shadow-elevated: 0 4px 16px rgba(0, 0, 0, 0.1);
--min-tap-target: 44px;       /* Apple HIG */
--font-body: 16px;            /* 最小フォントサイズ */
```

## UI Architecture — 問診フロー型

### Flow (ADL: 8問)

```
[会話] → [咀嚼] → [嚥下] → [呼吸] → [歯磨き] → [立ち上がり] → [複視] → [眼瞼下垂]
   ↓
[ADL結果画面] → [MGCに進む] or [コピー/QR]
```

### Flow (MGC: 10問, うち4問はADL自動反映)

```
[眼瞼下垂(医師)] → [複視(医師)] → [閉眼筋力] → [会話※] → [咬む※] → [飲み込み※] → [呼吸※] → [頸筋力] → [上肢] → [下肢]
※ ADL重複項目: 自動反映済み、確認のみ or スキップ
   ↓
[MGC結果画面] → [最終結果] or [コピー/QR]
```

### 各画面の構成

#### 1. 項目画面（1問ずつ）
- **プログレスバー**: 全問数分のセグメント（現在地=グリーン、未回答=グレー）
- **カウンター**: 右上に「n / N」
- **項目名**: 中央、26px Bold
- **補足テキスト**: 14px、グレー（「該当する状態をタップしてください」）
- **選択肢**: 縦並びカード
  - 左: 点数バッジ（36x36px、角丸10px、グリーン背景）
  - 右: 選択肢テキスト（16px）
  - 全体: 白背景、1.5px border、角丸14px、min-height 52px
- **戻るリンク**: 画面下部「← 前の項目に戻る」（2問目以降）

#### 2. 選択時の挙動
- タップ → 選択肢がグリーンボーダー+背景に変化
- 300ms後 → スライドアニメーションで次の項目へ自動遷移
- スコアは内部で即座に記録

#### 3. ADL結果画面
- **スコア円**: Calmi MGScoreDisplay風
  - 140x140px、6px border グリーン
  - 中央に大きなスコア数字（48px Bold）
  - 下に「/ 24点」（14px）
- **項目一覧**: カード内リスト
  - 項目名（左）+ 点数（右、色分け: 0=グリーン, 1-2=オレンジ, 3=レッド）
- **ボタン**:
  - Primary: 「MGCに進む →」（フル幅、グリーン）
  - Secondary 2つ横並び: 「コピー」「QRコード」
  - Ghost: 「やり直す」（テキストリンク、赤）

#### 4. MGC結果画面
- ADL結果画面と同構造
- Primary: 「最終結果を見る」

#### 5. 最終結果画面
- ADL + MGC 両方のスコア円を横並び
- 統合結果テキスト
- ボタン: 「コピー」「QRコード」「最初からやり直す」

## QR Code Implementation

### 方式
- **完全インライン**: QR生成JSをHTML内に埋め込み（~3KB minified）
- **Canvas API**: `<canvas>` 要素に直接描画
- **外部依存ゼロ**: CDN・外部ライブラリ不使用

### QRコード生成ロジック
- 入力: 結果テキスト（textarea.value と同じ内容）
- エンコーディング: UTF-8 Byte mode
- エラー訂正レベル: M（15%）
- バージョン: テキスト長に応じて自動選択（v1-v10程度）
- 出力: Canvas 200x200px、白背景+黒モジュール

### 表示方法
- 「QRコード」ボタンタップ → ボタン下にCanvasが表示/非表示トグル
- Canvas長押しで画像保存可能（iOS Safari対応）
- 下部に「長押しで画像を保存できます」ヒントテキスト

### 各タブのQR内容
- **ADL結果**: MG-ADLスケール結果テキスト（日付+全項目+合計）
- **MGC結果**: MG Composite Scale結果テキスト
- **最終結果**: 両スケール統合テキスト

## ADL → MGC マッピング

既存ロジックを維持:
| ADL項目 | MGC項目 |
|---------|---------|
| 会話 (index 0) | 会話、発音 (index 3) |
| 咀嚼 (index 1) | 咬む動作 (index 4) |
| 嚥下 (index 2) | 飲み込み動作 (index 5) |
| 呼吸 (index 3) | MGによる呼吸状態 (index 6) |

MGCフローでは重複4項目をADLから自動反映。ユーザーは確認画面をスキップまたは変更可能。

## Technical Constraints

- **単一HTML**: CSS/JS全てインライン。ファイル分割なし
- **完全オフライン**: 外部リソース一切不使用
- **iOS Safari最適化**: viewport meta、-webkit-overflow-scrolling、touch events
- **既存機能維持**: スコア計算ロジック、結果テキストフォーマット、コピー機能

## Out of Scope

- ダークモード対応（将来検討）
- MG-QOL15タブ（mg-scale-fixed.htmlにのみ存在、index_iOS.htmlには不要）
- デスクトップ最適化（モバイルファースト）
- データ永続化/履歴
