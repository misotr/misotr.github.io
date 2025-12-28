# Nostr kind:30000 ユーザリスト編集アプリ 詳細設計（実装者向け）

## 目的
外部仕様（別ドキュメント）を実装として成立させるための、構成・状態管理・通信フロー・関数責務・安全設計・リファレンスをまとめる。

本書は PoC 実装（単一 HTML）を前提とし、将来の分割（モジュール化・ビルド導入）も見越した「引き継ぎ可能な粒度」を目標とする。

---

## 実装の前提
- 実行環境：PC ブラウザ
- 依存：nostr-tools（ESM）
- ログイン：nsec をユーザ入力 → `nip19.decode` → secretKey（Uint8Array）
- 通信：複数リレーへ WebSocket（SimplePool）
- 永続化：なし（メモリのみ）

---

## フォルダ/ファイル構成（現状）
- `index.html`（単一ファイル）
  - `<style>`：ライト調の UI
  - `<body>`：screenList / screenEdit と各 modal
  - `<script type="module">`：状態管理・Nostr 通信・レンダリング

将来の推奨分割（任意）：
- `ui/`：DOMレンダリング、モーダル制御
- `nostr/`：リレー通信、イベント構築、暗号化
- `state/`：状態・reducers
- `security/`：URLサニタイズ、CSP管理

---

## 画面構成（DOM）

### Screen
- `screenList`：リスト一覧
  - `listCards`：d ごとに最新の kind:30000 をカード表示
  - `settingsPanelList`：リレー設定、デバッグログ
- `screenEdit`：リスト編集
  - `entryList`：エントリ行（プロフィール + petname + 非公開チェック + 削除）
  - `settingsPanelEdit`：title/description/d とデバッグログ
  - `footerBar`：追加・publish（常時表示）

### Modals
- `loginModal`：nsec 入力 + 初期 d
- `newListModal`：新規作成（d 必須、title/description 任意）
- `addModal`：npub 追加（kind:0 プレビュー → 追加）
- `publishModal`：リレー別 publish 結果
- `deleteListModal`：kind:5 削除（a/k タグ）

---

## 状態管理（in-memory）

### グローバル
- `sk`：secret key（Uint8Array）。メモリのみ。
- `pkHex`：自分の pubkey（hex）
- `npub`：自分の npub

### キャッシュ
- `profileCache: Map<hex, profileJson>`
  - kind:0 を JSON parse して保持
  - 取得できない場合は未登録のまま
- `listIndex: Map<d, {event, d, title, description, created_at, id}>`
  - 一覧表示用（d ごとに最新 event）
- `existingDs: Set<d>`
  - 新規リスト作成の重複チェック用

### 編集ステート
- `dirty: boolean`：未反映変更あり
- `state`：現在編集中のリスト
  - `d: string`
  - `title: string`
  - `description: string`
  - `entries: Entry[]`

`Entry` 例：
- `hex: string`（必須、ユニーク）
- `npub: string`（表示用）
- `display: string`（kind:0 由来）
- `name: string`（kind:0 由来）
- `avatar: string`（kind:0 由来、https/data:image のみ許可）
- `petname: string`（任意。未入力は空欄維持）
- `relayHint: string`（現状は "" 固定、将来拡張余地）
- `isPrivate: boolean`
- `pendingDelete: boolean`（削除予定グレー表示）

不変条件：
- `entries.hex` は重複しない（追加時に弾く）
- `petname` は自動補完しない
- `pendingDelete` は publish で除去される

---

## Nostr 通信設計

### 使用ライブラリ
- `SimplePool`：複数リレー購読・publish
- `finalizeEvent`：署名 + id 計算
- `getPublicKey`：pubkey 算出
- `nip19`：npub/nsec decode
- `nip04`：暗号化/復号（自分宛）

### リレー選定
- 初期：bootstrap relay を `relayInput` に設定
- ログイン後：kind:10002 を `fetchOne` で取得
  - `tags: ["r", <url>]` を抽出
  - `normalizeRelayUrls()` にかけて最大20件
  - 取得できた場合は `relayInput` を置換

### 購読（read）
- `fetchMany(filter, relays, timeoutMs)`
  - `pool.subscribeMany(relays, [filter], {onevent, oneose})`
  - timeout でも close して結果を返す
  - event.id で重複排除
- `fetchOne(...)`
  - `fetchMany` の結果を `created_at desc` でソートして先頭

利用箇所：
- `loadAllLists()`：kind:30000 を複数取得 → d ごとに latest を index
- `fetchProfilesForEntries()`：kind:0 を必要分だけ取得
- `previewBtn`：kind:0 取得 → 追加のプレビュー

### publish（write）
- `publishToRelays(ev, relays)`
  - `pool.publish(relays, ev)` が返す Promise 群を個別に timeout 監視
  - 結果を `{url, ok, note}` に正規化

バグ対応の経験則：
- タイムアウト時は `pool.publish` の Promise が resolve/reject しない場合があるため `withTimeout` を必ず噛ませる。
- リレーURL末尾スラッシュ等は正規化し、同一接続を避ける。

---

## イベント構築

### kind:30000
- `buildKind30000Event()`
  - `kept = entries.filter(!pendingDelete)`
  - `publicTags`：`!isPrivate` の p タグ
  - `privateTags`：`isPrivate` の p タグ配列
  - tags：
    - `["d", state.d]`
    - `["title", titleOrD]`
    - `description` は任意
    - `...publicTags`
  - content：
    - `privateTags.length ? nip04.encrypt(sk, pkHex, JSON.stringify(privateTags)) : ""`

### kind:5
- `buildDeleteEvent()`
  - `a = "30000:<pkHex>:<state.d>"`
  - tags：`[["a", a], ["k", "30000"]]`
  - content：""

---

## UI レンダリング設計

### 原則
- XSS対策：**DOM への挿入は `textContent` / 属性セットのみ**。`innerHTML` は固定SVG以外使わない。
- 画像URLは `sanitizeImageUrl()` を通す（https と data:image のみ）
- bech32/hex は表示用に短縮（ホバーで hex 全体表示）

### 主要関数
- `switchScreen(which)`：list/edit の切替、戻るボタン制御
- `renderListCards()`：listIndex から一覧カード生成
- `renderEditAll()`：編集画面全体再描画
- `renderRow(entry)`：エントリ行を生成
- `markDirty(true/false)` / `updateDirtyUI()`：変更表示

レンダリング方針：
- PoC は「全体再描画」中心（シンプル優先）。
- パフォーマンスが問題になったら差分更新へ。

---

## 入力検証・正規化

### 文字数制限
- `LIMITS` に集約。
- `clampStr(s, max)` で truncate。

### リレーURL
- `normalizeRelayUrls(raw)`
  - `new URL()` で parse
  - scheme は `wss:` / `ws:` のみ許可
  - `hash/search` を削除、末尾スラッシュ除去
  - 重複排除、最大20件

### npub/nsec
- `bech32ToSk(nsec)`：`nip19.decode` type チェック
- `decodeNpubToHex(npub)`：同上

---

## セキュリティ設計（nsec入力以外）

### 守るべきこと
- nsec を event content/tags に混入しない
- nsec を debug log に出さない
- 不正なプロファイル（kind:0 content）で挙動を変えない（XSS, DOM injection）

### 実装している対策
- 画面表示：`textContent` を使用
- kind:0 JSON parse：try/catch、欠損値は空扱い
- 画像：`sanitizeImageUrl`（https と data:image のみ）
- CSP：connect-src に `wss:` を許可、object-src/ base-uri/ frame-ancestors を制限

### 注意点
- data:image は許可しているため、理論上は巨大 data URI によるメモリ圧迫の余地がある。
  - 必要ならサイズ上限を追加する。

---

## ログ/デバッグ
- `debugLog(msg, obj)`
  - console 出力 + textarea に追記
  - publish の summary（id/kind/tags_count/content_len）を出す
  - relay 別結果を記録

推奨ログ追加ポイント：
- `fetchMany` の onevent / oneose 回数
- relay ごとの接続開始/終了（必要になった場合）

---

## テスト観点（実装者向け）
- bech32 decode（npub/nsec）異常系
- 重複追加の拒否
- 1,000件制限（既存>1000、追加拒否、削除のみ）
- 公開/非公開切替で tags/content が期待通りになる
- publish の relay 別結果（成功/失敗/timeout）
- kind:5 発行と、UI上の警告・再送

---

## 残課題（TODO）
- **未来 created_at の除外**：
  - 「現在時刻 + 許容差（例：5分/10分）」を超える created_at は採用しない。
  - 実装箇所：`loadAllLists()` の latest 選定、`fetchOne()` のソート後フィルタ
- リレー間不整合の可視化：
  - 同一 d のイベントがリレーごとに異なる場合の表示（取得元リレー、差分、警告）
- kind:10002 の read/write 区別対応（将来）：
  - rタグの追加要素を解釈し、read/write を分離する UI
- パフォーマンス：
  - 大量 entries 時の差分レンダリング
  - kind:0 の取得のバッチ最適化/更新頻度
- セキュリティ：
  - data:image のサイズ制限
  - nsec入力を本番方式へ置換（NIP-07 等、PoC外）

---

## 関数リファレンス

本節では、PoC 実装内で主要となる関数について、役割・引数・戻り値・例外・注意点を整理する。

> 記載の「例外」は JavaScript の `throw` によるものに加え、`Promise` の reject（非同期例外）も含む。

---

### `clampStr(s, max)`
- 概要：文字列を最大長 `max` に切り詰めて返す。非文字列は空文字として扱う。
- 引数：
  - `s`: 任意（通常は string）。`typeof s !== "string"` の場合は ""。
  - `max`: number（>0 を想定）。
- 戻り値：string（長さが `max` 以下）。
- 例外：基本的に発生しない。
- 補足：UI入力・タグ・プロファイルなど、外部由来文字列を一律で制限するために使用。

### `safeShort(s, n)`
- 概要：長い文字列を `...` 付きで短縮表示する（表示用途）。
- 引数：
  - `s`: string（空/undefined は "" 扱い）
  - `n`: number（表示目標長。最小 6 程度を想定）
- 戻り値：string（短縮済みまたは元の文字列）。
- 例外：基本的に発生しない。
- 補足：npub/hex/id の表示に使用。値そのものは保持し、表示のみ短縮する。

### `sanitizeImageUrl(url)`
- 概要：プロフィール画像URLを安全に使用できる形式に制限する。
- 引数：
  - `url`: string（kind:0 の `picture` 等）。
- 戻り値：string（許可されたURL）または ""。
- 例外：`new URL()` 失敗を内部で catch し、例外は外に出さない。
- 補足：
  - 許可：`https:`、および `data:image/(png|jpeg|jpg|webp|gif)` のみ。
  - 不許可：`http:`、`javascript:`、その他のスキーム。
  - data URI は巨大化によるメモリ圧迫余地があるため、将来サイズ上限を入れる候補。

### `normalizeRelayUrls(raw)`
- 概要：ユーザ入力や kind:10002 から得たリレーURLを正規化して配列化する。
- 引数：
  - `raw`: string（改行区切りを想定）。
- 戻り値：string[]（最大 20 件、重複排除、末尾スラッシュ除去）。
- 例外：`new URL()` 失敗は内部で握りつぶし、その行を無視。
- 補足：
  - 許可スキーム：`wss:` / `ws:`。
  - `hash` / `search` は削除。
  - 末尾 `/` を削除して接続の重複を減らす。

### `setRelayUrls(urls)` / `getRelayUrls()`
- 概要：リレー入力欄（textarea）への反映、および現在のリレー集合取得。
- 引数：
  - `setRelayUrls(urls)`: string[]（未正規化でも可。内部で正規化）。
- 戻り値：
  - `setRelayUrls`: なし。
  - `getRelayUrls`: string[]（空なら bootstrap relay を返す）。
- 例外：基本的に発生しない。
- 補足：
  - `getRelayUrls()` は「空なら bootstrap」を保証し、以降の読取/書込が必ず何かのリレー集合を持つ。

---

### `bech32ToSk(nsec)`
- 概要：nsec（bech32）を secret key（Uint8Array）に変換する。
- 引数：
  - `nsec`: string（`nsec1...`）。
- 戻り値：Uint8Array（secret key）。
- 例外：
  - `nip19.decode` が失敗した場合。
  - decode の `type !== "nsec"` の場合（"nsec ではありません"）。
- 補足：戻り値は極めて機密。ログ出力や DOM 反映禁止。

### `decodeNpubToHex(npubStr)`
- 概要：npub（bech32）を hex 公開鍵に変換する。
- 引数：
  - `npubStr`: string（`npub1...`）。
- 戻り値：string（hex 公開鍵）。
- 例外：
  - `nip19.decode` 失敗。
  - decode の `type !== "npub"` の場合（"npub ではありません"）。
- 補足：ユーザ入力は npub のみ受け付ける方針。

### `encryptPrivatePTags(privatePTags)`
- 概要：非公開エントリ（pタグ配列）を NIP-04 で暗号化し、content 用の文字列を生成する。
- 引数：
  - `privatePTags`: any[]（期待値：`[["p", hex, relayHint, petname], ...]`）。
- 戻り値：Promise<string>（暗号化結果）。
- 例外：
  - `nip04.encrypt` が reject した場合（鍵不整合など）。
- 補足：
  - 宛先は自分自身（`pkHex`）。「自分だけが復号可能」固定。
  - 事前に `JSON.stringify` するため、循環参照は想定しない（privatePTags は単純配列のみ）。

### `decryptPrivatePTags(content)`
- 概要：kind:30000 の `content` を復号し、非公開 p タグ配列を得る。
- 引数：
  - `content`: string（暗号化された文字列）。
- 戻り値：Promise<any[]>（期待：pタグ配列。失敗時は空配列）。
- 例外：外には投げず、失敗時は `[]` を返す。
- 補足：
  - 失敗要因：鍵不一致、暗号文破損、JSON parse 失敗、巨大 content（上限超過）など。
  - content が無い/長すぎる場合は即 `[]`。

---

### `fetchMany(filter, relays, timeoutMs)`
- 概要：複数リレーに同一フィルタで購読を張り、イベントを収集して配列で返す。
- 引数：
  - `filter`: object（Nostr filter。例：`{kinds:[30000], authors:[pkHex], limit:200}`）。
  - `relays`: string[]（対象リレーURL群）。
  - `timeoutMs`: number（購読の最大待ち時間）。
- 戻り値：Promise<Event[]>（イベント配列。0件もあり得る）。
- 例外：外には投げにくい設計（購読失敗は結果0件になりやすい）。
- 補足：
  - `oneose` で終了するが、リレーが EOSE を返さない場合に備え timeout を必ず設定。
  - event.id で重複排除。
  - 返却後に `sub.close()` を試みる。

### `fetchOne(filter, relays, timeoutMs)`
- 概要：`fetchMany` の結果から created_at 最大の1件を返す。
- 引数：`fetchMany` と同じ。
- 戻り値：Promise<Event|null>（0件なら null）。
- 例外：`fetchMany` 同様、外には投げにくい。
- 補足：
  - future created_at を弾く仕様を入れる場合、ここ（または呼び出し側）でフィルタリングする。

### `withTimeout(promise, ms, label)`
- 概要：Promise を一定時間でタイムアウトさせるラッパ。
- 引数：
  - `promise`: Promise<any>
  - `ms`: number
  - `label`: string（ログ用）
- 戻り値：Promise<any>
- 例外：タイムアウト時に `Error("timeout(label)")` で reject。
- 補足：publish 結果が返らないリレー/実装差の吸収に重要。

### `publishToRelays(ev, relays)`
- 概要：イベントを複数リレーへ publish し、リレー別の成否を整形して返す。
- 引数：
  - `ev`: Event（`finalizeEvent` 済み）
  - `relays`: string[]
- 戻り値：Promise<Array<{url:string, ok:boolean, note:string}>>
- 例外：通常は結果配列として吸収するが、想定外の例外があれば reject し得る。
- 補足：
  - `pool.publish` が返す Promise 群に `withTimeout` を噛ませる。
  - 返却は `Promise.allSettled` で安全に正規化。

### `loadKind10002Relays()`
- 概要：kind:10002 を取得し、推奨リレー（rタグ）を relayInput に適用する。
- 引数：なし（内部で `pkHex` と `getRelayUrls()` を参照）。
- 戻り値：Promise<void>
- 例外：外には投げない設計（未取得なら何もしない）。
- 補足：
  - rタグの read/write 区別は現状未対応（将来TODO）。
  - 取得できたら bootstrap を置換する。

### `loadAllLists()`
- 概要：自分の kind:30000 を収集し、d ごとに最新を `listIndex` に構築して一覧表示を更新する。
- 引数：なし（内部で `pkHex`, `getRelayUrls()` を参照）。
- 戻り値：Promise<void>
- 例外：外には投げない設計。
- 補足：
  - `existingDs` を併せて構築し、新規作成時の重複チェックに使う。
  - 「未来 created_at の除外」仕様を入れる場合、latest 選定時に除外する。

---

### `hydrateStateFromEvent(ev)`
- 概要：kind:30000 イベントから編集用 `state` を構築する。
- 引数：
  - `ev`: Event（kind:30000 を想定）。
- 戻り値：Promise<void>
- 例外：外には投げにくい（復号失敗等は空配列扱い）。
- 補足：
  - tags から `d/title/description` を抽出。
  - 公開：`p` タグを読み取り。
  - 非公開：`decryptPrivatePTags(ev.content)` で復号。
  - 公開/非公開を同一 `entries` に統合し、hex 重複は排除。

### `fetchProfilesForEntries()` / `applyProfiles()`
- 概要：entries に必要な kind:0 を取得し、表示用情報（display/name/avatar）を埋める。
- 引数：なし。
- 戻り値：
  - `fetchProfilesForEntries`: Promise<void>
  - `applyProfiles`: void
- 例外：外には投げない設計。
- 補足：
  - `profileCache` に無い pubkey のみ取得。
  - kind:0 の content は JSON として parse（失敗は無視）。
  - avatar は `sanitizeImageUrl` を通す。

### `buildKind30000Event()`
- 概要：現在の `state` から kind:30000 の tags と、暗号化対象の privateTags を生成する。
- 引数：なし（内部で `state` を参照）。
- 戻り値：
  - `{ tags: string[][], privateTags: any[] }`
- 例外：基本的に発生しない。
- 補足：
  - `pendingDelete` は除外。
  - 公開：tags に p タグとして追加。
  - 非公開：privateTags として蓄積し、後段で暗号化して content に入れる。
  - p タグは NIP-51 相互運用のため 4要素固定（relay_hint が無くても "" を入れる）。

### `publishNow()`
- 概要：編集内容を検証し、kind:30000 をフルリスト再送として publish する。
- 引数：なし。
- 戻り値：Promise<void>
- 例外：
  - 入力不備（d 空、上限超過など）は `alert` で中断（throw ではない）。
  - publish 内部例外は reject し得る（ネットワーク/署名など）。
- 補足：
  - `finalizeEvent` で署名し、`publishToRelays` の結果をUIに表示。
  - publish 後に `pendingDelete` を entries から除去し、dirty を false に戻す。

### `buildDeleteEvent()`
- 概要：対象リストを削除要求する kind:5 イベントを構築する。
- 引数：なし（内部で `pkHex`, `state.d` を参照）。
- 戻り値：`{ ev: Event, a: string }`
- 例外：署名処理（finalizeEvent）で例外が起きれば throw/reject し得る。
- 補足：
  - a タグ：`30000:<pkHex>:<d>`
  - k タグ：`30000`

---

### `switchScreen(which)`
- 概要：screenList/screenEdit の表示切替、およびタイトル等のUI状態を同期する。
- 引数：
  - `which`: "list" | "edit"
- 戻り値：void
- 例外：基本的に発生しない。

### `renderListCards()`
- 概要：`listIndex` の内容から一覧カードDOMを生成して表示する。
- 引数：なし。
- 戻り値：void
- 例外：基本的に発生しない。
- 補足：
  - DOM 注入は `textContent` を使用。
  - created_at はローカル時刻に変換して表示。

### `renderEditAll()`
- 概要：編集画面（d/title/description/entries/制限表示）を全体再描画する。
- 引数：なし。
- 戻り値：void
- 例外：基本的に発生しない。
- 補足：
  - PoC は差分更新ではなく全体再描画。大量件数では重くなる可能性あり（TODO）。

### `renderRow(entry)`
- 概要：1エントリ分の行DOMを生成する（プロフィール表示、petname編集、非公開チェック、削除予定トグル）。
- 引数：
  - `entry`: Entry
- 戻り値：HTMLElement（row）
- 例外：基本的に発生しない。
- 補足：
  - hex は hover ツールチップで表示（data-hex 属性）。
  - petname 入力は `entry.petname` に即時反映し `dirty` を立てる。

### `markDirty(v)` / `updateDirtyUI()`
- 概要：未反映変更フラグ（dirty）を更新し、UI（ドット/文言）を同期する。
- 引数：
  - `markDirty(v)`: boolean
- 戻り値：void
- 例外：基本的に発生しない。
- 補足：
  - publish 成功後は `markDirty(false)`。

### `debugLog(msg, obj)`
- 概要：デバッグログを console と textarea に追記する。
- 引数：
  - `msg`: string
  - `obj`: any（任意、JSON化して追記）
- 戻り値：void
- 例外：JSON stringify 失敗は内部で吸収し "[unserializable]"。
- 補足：
  - 機密（nsec/sk）を渡さない運用ルールが前提。
  - publish の summary（content_len 等）を出して「秘密が混入していない」ことを目視しやすくする。

