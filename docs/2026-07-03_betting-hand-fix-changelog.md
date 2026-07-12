# 変更点まとめ (2026-07-03) — 掛け金フェーズ手牌消失バグ ほか

## 症状と根本原因

**症状**: Betting（掛け金）フェーズ遷移時に自分の手牌が減り、オブジェクトプールに牌が戻る。

**根本原因**: `BoardStateManager.SetLocalState / SetEnemyState` が `hand` 引数を「壁配列のインデックス」として解釈していた（`hand.Contains(i)`）。
一方、全呼び出し元（`HandSelectionMessageHandler`, `StatusMessageHandler`）は「牌の値」を渡す設計。
サーバー仕様も確認済み:
- `hand_selection_completed` の `hand` はサーバー側でインデックス→牌IDに変換済みの **値**（`game_session.py:819`）
- `status` の `player_state.hand` は **インデックス** だが、`StatusMessageHandler` が値に変換してから渡す（65-71行）

結果、手牌13枚の値がインデックスとして解釈され（重複除去で5個に減り）、壁の無関係な位置から5枚拾った壊れた盤面（手牌5/壁29）になる。
Betting遷移時のリビルドがそれを忠実に描画して「手牌が減る・余剰牌がプールへ」となっていた。
NetworkMessageHandler 分割リファクタリング時の回帰と推定。

**副次効果**: 以前から未解決だった「東など特定牌が両者の手牌に偏って表示される」問題も同根の可能性が高い。

## 変更ファイル

### 1. Assets/Scripts/Managers/BoardStateManager.cs（根本修正）
- `SetLocalState` / `SetEnemyState` を「hand は牌の値」を前提とする実装に変更。
- 壁スロットを値一致で1枚ずつ消費して手牌に割り当てる（重複牌対応）。
  照合は完全一致を優先し、無ければ `& 0x1F`（baseId）でドラフラグ差を許容（`MoveTileToHand` と同じ規約）。
- `LocalHandIndexes` / `EnemyHandIndexes` は正しい壁インデックスとして算出。
- 照合できない牌が残った場合は `Debug.LogWarning` を出す。

### 2. Assets/Scripts/UI/HandBaseUI.cs（防御ガード）
- `UpdateLayout` の冒頭に Betting フェーズの早期 return を追加。
  Betting 中は `discardPhaseContainer` / `handSlotContainer` の切り替えを一切行わない（どの呼び出し元経由でも安全）。

### 3. Assets/Scripts/UI/GameUIVisualController.cs（防御ガード / 指示書の修正2）
- `RebuildAllTilesFromState` に Betting フェーズガードを追加（IsTransitioning チェックの直後）。
  Betting 中のフルリビルドをスキップ（スキップ時に Debug.Log 出力）。

### 4. Assets/Scripts/UI/GameUIPhaseController.cs
- 指示書の修正1（`UpdatePhaseStatus` で Betting 時に `HandUI.UpdateLayout` をスキップ）は一度適用後、**撤去**。
  スキップすると「選び直す」等のボタン表示が Betting 遷移時に再評価されず表示が残るため。
  コンテナ保護は 2. の HandBaseUI 内ガードが担う（その旨のコメントを追記済み）。

### 5. Assets/Scripts/UI/GameUIManager.cs（第2局で敵手牌が増えるバグ）
- `ClearAllTiles()` で敵手牌の GameObject を `ReturnTileToPool` してから `ClearHand()` を呼ぶよう修正。
  従来は `ClearHand()`（リストのクリアのみ）だけで、前局の敵手牌 GameObject が孤児化して画面に残り続けていた。

### 6. Assets/Scripts/UI/TilePoolManager.cs（プール返却の整列・表向き化）
- `_slotTileIds`（各スロットの本来の牌ID、初期化時の deck 順）を記録するフィールドを追加。
- `ReturnTileToPool` の返却先スロット選択を「赤ドラ0x40込み完全一致 → baseId一致 → 任意の空き」の優先順に変更（元の位置に戻る）。
- 返却時に RectTransform をリセット（アンカー/ピボット中央・anchoredPosition3D=0・回転/スケール初期化）。
  従来は直前レイアウトのローカル座標が残り、Scene view でプール周辺に牌が散らばって見えていた。
- `TileResourceManager` から表向きスプライトを当て直し（従来は `SetTile(id, null)` で敵の裏牌表示のまま残存）、
  ホバー/フリテン/公開ハイライトもリセット。
- `_uiManager` フィールドを追加（`InitializePool` でキャッシュ、スプライト取得に使用）。

### 7. シーン変更（メインシーン）
- `Canvas/EnemyHandCanvas/EnemyHandPanel/HandSlotContainer` の anchoredPosition.y を **60 → 25**（敵手牌の表示位置を計35px下げ）。

## 備考
- Python サーバーコード（mahjong_engine/）は無変更（プロジェクトルール遵守。仕様確認のための読み取りのみ実施）。
- デバッグ用に一時追加した調査ログ（フェーズ遷移・プール返却・コンテナ切替・リビルド開始）は全て削除済み。
- 動作確認済み: Betting遷移で手牌13枚維持 / 「選び直す」ボタン非表示 / 第2局で敵手牌が増えない。
