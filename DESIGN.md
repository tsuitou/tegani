# tegani アーキテクチャ設計書（v2 / コアモデル再設計）

## 0. この文書の目的

tegani（レイヤー型ラスタ・アニメーションエディタ）の内部設計を、現行の
「base64 文字列を中心に据えた設計」から、**ドキュメントモデルを中心に据えた設計**へ
置き換える。本書は実装前に責務・データ構造・層間 API・データフローを確定させるための
合意ドキュメントである。

- 配布形態: **単一 `index.html` を維持**（ビルド工程を増やさない）
- 最適化ターゲット: **タブレット / 低スペック端末**（メモリと main-thread CPU が制約）
- 移行方針: コアモデルから再設計し、ツール / UI をその上に載せ替える

---

## 1. 現行設計の構造的欠陥（再設計の動機）

1. **ドキュメントとビューが未分離。** 可視 `<canvas>` が事実上の編集対象＝ドキュメント
   になっており、`toDataURL()` で `frame.layers[id]`(base64) に焼き戻して同期している。
   → 真実の源が二重化し、操作ごとに encode/decode/clone が不可避。
2. **ピクセルの内部表現が文字列。** base64 PNG は I/O 用の直列化形式であるべきだが、
   メモリモデル・履歴・レンダリング入力すべてに流用されている。しかも `reactive()`
   配下に置かれ、UI フレームワークが数 MB の文字列を監視している。
3. **関心の分離がない。** 約 3650 行が単一の Vue `setup()` クロージャに同居し、
   描画ロジックが UI 状態を直接変更し、ついでに永続化フォーマットを生成している。

**設計指針（3つの境界を引き直す）:**
- **canvas はビュー**であり、ドキュメントではない。
- **ピクセルはラスタバッファ**であり、文字列ではない。
- **base64 / PNG / zip / webm は I/O 境界にのみ存在する。**

---

## 2. レイヤー構成（責務）

```
┌──────────────────────────────────────────────────────────┐
│ UI Layer (Vue)                                            │
│   reactive: 構造 + メタのみ（pixels は持たない）            │
│   frameOrder / layerMeta / 選択 / current index / fps /    │
│   dirty フラグ / ツール状態                                 │
└──────────▲──────────────────────────────────┬────────────┘
           │ subscribe(描画要求)               │ dispatch(Command)
┌──────────┴──────────┐          ┌─────────────▼────────────┐
│ Compositor          │          │ ToolController            │
│  Model→viewCanvas へ │          │  pointer入力→Command生成   │
│  合成。canvas枚数は   │          └─────────────┬────────────┘
│  レイヤー数で有界     │                        │ apply / invert
└──────────▲──────────┘          ┌─────────────▼────────────┐
           │ read(surface)        │ History (command stack)   │
┌──────────┴───────────────────┐ │  領域差分(bbox)を保持       │
│ DocumentModel (真実の源・非reactive)                        │
│   CellStore: (frameId,layerId) → Cell{surface|bytes}       │
│   近傍のみ decoded、遠方は圧縮バイナリで退避(LRU)            │
└──────────▲───────────────────┬────────────────────────────┘
           │ serialize          │ deserialize
┌──────────┴──────┐   ┌─────────▼───────────────────────────┐
│ Persistence      │   │ Exporter                            │
│  IndexedDB,       │   │  各frameをModelから合成→PNG/zip/webm  │
│  dirty cellのみput │   │  UPNG/fflate は Worker にオフロード    │
└──────────────────┘   └─────────────────────────────────────┘
```

中核ルール: **ビューはモデルから派生し、モデルがビューから派生することはない。**

---

## 3. DocumentModel

### 3.1 reactive に置くもの（UI が監視する構造・メタ）

```
project = reactive({
  id, meta: { width, height, fps, createdAt, modifiedAt },
  layers: [ LayerMeta ],          // 全フレーム共通のレイヤー定義
  frames: [ { id } ],             // 構造のみ。pixels は持たない
})
LayerMeta = { id, name, type:'draw', visible, opacity, locked }
session  = reactive({ currentFrameIndex, currentLayerId, tool, colors, brush, selection, ... })
```

> ピクセルデータは `project` に**入れない**。これにより Vue は MB 級文字列の監視から解放される。

### 3.2 CellStore（非リアクティブ・ピクセルの実体）

セル = 1 つの (frame, layer) のピクセル面。

```
Cell = {
  state: 'empty' | 'decoded' | 'encoded',
  surface: Surface | null,   // decoded時: 直接描画可能なラスタ面
  bytes:   Uint8Array | null,// encoded時: PNG等の圧縮バイト列
  dirty:   boolean,          // 保存が必要（surfaceがbytesと不一致）
  lastUsed: number,          // LRU用
}
```

`Surface` 抽象: 描画可能なオフスクリーン面。実体は `OffscreenCanvas` を第一候補、
未対応端末では detached `<canvas>`(documentに非接続) にフォールバック（§8 互換性参照）。

CellStore API（同期/非同期は実装時確定、概念上）:

| メソッド | 役割 |
|---|---|
| `getSurface(frameId, layerId)` | 編集用に decoded 面を保証して返す（bytesからdecode-on-demand）|
| `peek(frameId, layerId)` | 合成用の読み取り（decoded面 or デコード）|
| `markDirty(frameId, layerId, bbox?)` | 編集後に dirty 化、変更領域を記録 |
| `newCell(frameId, layerId, init)` | 透明 / 単色などで初期化 |
| `evictIfNeeded()` | decoded 数が予算超過時、LRU を encode→surface解放 |
| `serializeDirty()` | dirty セルを PNG bytes 化（保存前）|

**メモリ戦略（タブレット対応の肝）:** decoded 面は「アクティブフレーム + オニオン近傍 +
再生先読み窓」だけに限定し、`DECODE_BUDGET` を超えたら LRU で `bytes` に退避して
`surface` を破棄（`OffscreenCanvas` 解放）。この戦略は CellStore の内部に閉じ込め、
他層へ漏らさない。

---

## 4. Compositor

モデルから**固定枚数**の DOM canvas へ合成する。canvas 枚数は**レイヤー数で有界**
（フレーム数には依存しない）。

- 出力 canvas: `bg`(チェッカー) / レイヤーごとの view canvas / `onion` / `temp`(カーソル・選択)。
  ※レイヤーごとの view canvas は CSS で opacity/visible を安価に切替できる利点を維持。
  重要なのは **これらが model から blit されるビューであり、編集対象ではない**点。
- `renderFrame(frameIndex)`: 各レイヤーの Cell 面を対応 view canvas へ `drawImage` で blit。
- `renderActiveLayer()`: ストローク中はアクティブレイヤーだけ rAF で再 blit（低遅延）。
- `renderOnion()`: オニオン面を合成（輝度マスクは canvas 合成演算で実装し、毎回の
  全画素 JS ループは持たない）。オニオン無効時・近傍未変化時はスキップ。

**ライブ描画の遅延設計:** ツールは Cell 面（＝モデル）に直接描く。view canvas へは
変更領域のみ blit。base64 は一切経由しない（現行の toDataURL 同期を構造的に廃止）。

---

## 5. ToolController と Command（履歴）

### 5.1 Command パターン

ツールのポインタ操作は **Command** を生成し、History 経由で適用する。

```
Command = {
  type,                       // 'stroke' | 'fill' | 'paste' | 'structural' ...
  apply(model),               // 前進適用
  invert() -> Command,        // 逆操作（undo 用）
}
```

ラスタ編集は**領域差分**で表現する:

```
RasterPatch = {
  frameId, layerId, bbox: {x,y,w,h},
  before: ImageData,   // bbox 範囲の編集前
  after:  ImageData,   // bbox 範囲の編集後
}
undo = putImageData(before, bbox);  redo = putImageData(after, bbox)
```

→ フレーム全体の base64 スナップショットを**完全に廃止**。undo 段数を増やしても
メモリは「変更矩形 × 段数」に比例するだけ。

### 5.2 ストロークのライフサイクル

1. pointerdown: アクティブ Cell の影響 bbox を見積もり、`before` を捕捉開始。
2. pointermove: Cell 面へ描画 + アクティブ view を blit（モデルが即時に真実）。
3. pointerup: 確定 bbox の `after` を捕捉 → `RasterPatch` を History へ push、Cell を dirty 化。
   （サムネは idle で 1 回、Cell 面から直接縮小生成。base64 再デコードなし）

### 5.3 構造編集

add/delete/insert/reorder frame, add/remove/reorder layer は構造 Command として
逆操作を保持（現行の `recordStructural` 相当だが Command に統一）。

---

## 6. Persistence

- IndexedDB ストア:
  - `projects`: meta + 構造（frameOrder, layerMeta）。pixels を含まない。
  - `cells`: key=(projectId, frameId, layerId) → `{ bytes }`（PNG 等）。
- 保存 = `serializeDirty()` で**dirty セルのみ** encode して put（増分保存）。
- 読み込み = 構造を reactive に流し込み、Cell は遅延デコード（必要時に `bytes`→`surface`）。

### マイグレーション
- `DB_VERSION` を上げる。旧形式（`frame.layers[id] = base64`）を検出したら
  起動時に一度だけ Cell へ変換するパスを用意し、既存プロジェクトを失わない。

---

## 7. Exporter

- 各フレームを **Model から** オフスクリーンへ合成（view canvas には依存しない）。
- UPNG / fflate / WebMWriter の重い encode は **Web Worker** にオフロード。
  単一 HTML を保つため、Worker は **Blob URL 経由でインライン生成**する。
- フレーム間に yield を入れ、進捗表示で UI を固めない。

---

## 8. 互換性・リスク

| 項目 | 対応 |
|---|---|
| `OffscreenCanvas` 未対応端末 | `Surface` 抽象でラップし detached `<canvas>` にフォールバック |
| `ImageBitmap` のメモリ圧 | 原則 detached canvas 面 + LRU eviction。常時全フレーム展開しない |
| 既存保存データ | DB_VERSION バンプ + 旧 base64→Cell マイグレーション |
| 単一 HTML 制約 | 層は ES import せず**ファクトリ関数**で分離。Worker は Blob インライン |
| 描画遅延の退行 | アクティブ Cell 面へ直接描画 + 部分 blit で現行と同等以上を担保 |

---

## 9. 段階移行（コアから載せ替え）

> いずれも「設計の置換」であって個別チューニングではない。各段で手動検証
> （描画→undo/redo→フレーム移動→再生→3形式エクスポート→保存して再読込）を行う。

1. **M1: DocumentModel + CellStore + Compositor** を新設し、既存 UI を新モデルに接続。
   描画・フレーム移動が base64 を一切経由しなくなる状態にする。
2. **M2: Command/History** を導入し、フルスナップショット履歴を領域差分へ置換。
3. **M3: Persistence** を増分（dirty cell）化 + マイグレーション。
4. **M4: Exporter** を Worker 化。
5. **M5: メモリ戦略**（LRU eviction / 再生先読み窓）を CellStore に実装し、低スペック実機で検証。

---

## 10. 完了の定義

- `project` reactive 配下に base64 ピクセル文字列が存在しない。
- 1 ストロークの確定で `toDataURL` / フレーム全クローンが発生しない。
- 履歴メモリが「変更矩形 × 段数」で有界。
- フレーム数を増やしても view canvas 枚数が増えない。
- エクスポート中に UI スレッドがブロックされない。
