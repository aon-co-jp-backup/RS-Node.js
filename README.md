# RS-Node.js

Node.js のコア概念を、既存の Node.js 実装コードを一切流用せず Rust で
一から再実装するプロジェクト([RFrontEnd](https://github.com/aon-co-jp)
エコシステム傘下)。

## リポジトリ改称(2026-09-13)

`RNode.js`→`RS-Node.js`へGitHub上でrename済み。`aruaru.pro`向けの
フロント基盤整備に合わせた`RFrontEnd`傘下のネーミング統一の一環
(`RS-HTML`・`RS-CSS`・`RS-GraphQL`も同時に改称)。crate名(`rnodejs`)は
依存元が無いため変更しても影響は無いが、他の3リポジトリと揃えて
今回は見送り。以下の記述内の`RNode.js`表記は改称前の履歴として残す。

## これは何か(v0.1.0 の正直なスコープ)

Node.js は巨大であり、JavaScript エンジン(V8 相当)を一から作るのは
非現実的なため、v0.1.0 は **「Node.js 全体」ではなく次の 2 つの概念モデルの
骨格だけ**を実装している:

- **モジュール解決**(`resolve`): CommonJS 的な `require(path)` の
  パス解決アルゴリズム——相対パス・拡張子補完(`.js` → `.json`)・
  ディレクトリの `index`・`package.json` の `main`・`node_modules` の
  祖先方向探索——を、実ファイルシステムアクセスから切り離した純粋関数
  として実装。存在判定は `FileSystem` トレイトで抽象化され、テストは
  インメモリの `MapFileSystem` で実 I/O 無しに解決規則そのものを検証する。
- **イベントループ順序モデル**(`event_loop`): next-tick キュー →
  マイクロタスクキュー → タイマーフェーズという実行順序規則を、実時間や
  tokio に載せる前の**論理時刻ベースの決定的な純粋キューロジック**として
  実装。`setTimeout(0)` でも Promise `then` より後、タイマー 1 件ごとに
  tick/microtask を完全排出、といった Node の代表的挙動をテストで固定。
- **ランタイムアダプタ層**(`runtime`): 純粋モデルと実ランタイム
  (tokio/RPoem)の境界となる `LoopDriver` トレイト。テスト用
  `ManualDriver` 同梱。RPoem 統合は feature `rpoem-adapter` の設計
  スタブのみ(実装は未着手)。
- **`fs` モジュール**(`fs`、v0.2.0 で追加): Node の `fs`(同期 API)
  相当。`read_file`/`read_to_string`/`write_file`/`exists`/`mkdir`
  (`_recursive`)/`read_dir`/`stat`/`remove_file`/`remove_dir`
  (`_all`) を `std::fs` の薄いラッパーとして実装。実ファイルシステムに
  対する実 I/O(一時ディレクトリを使った読み書き往復テストで検証済み)。

**未実装(スコープ外)**: JavaScript の実行そのもの、`http`/`path` 等
それ以外のコアモジュール、`setImmediate`/I/O ポーリング/close フェーズ、
`package.json` の `exports`、ESM、`fs` の非同期版(`fs.promises`)。
RTypeScript との将来連携構想は `CLAUDE.md` の設計メモを参照。

## 使用例

```rust
use rnodejs::{resolve, MapFileSystem, EventLoop};

// モジュール解決(実 I/O 無し)
let fs = MapFileSystem::new()
    .with_file("/app/node_modules/lodash/index.js")
    .with_file("/app/src/util.js");
assert_eq!(resolve("lodash", "/app/src", &fs).unwrap(),
           "/app/node_modules/lodash/index.js");
assert_eq!(resolve("./util", "/app/src", &fs).unwrap(),
           "/app/src/util.js");

// イベントループ順序モデル(論理時刻・決定的)
let mut el = EventLoop::new();
el.schedule_timer(3, 0);      // setTimeout(0) 相当
el.schedule_microtask(2);     // Promise.then 相当
el.schedule_next_tick(1);     // process.nextTick 相当
let trace = el.run_until_idle();
assert_eq!(trace.iter().map(|e| e.id).collect::<Vec<_>>(), vec![1, 2, 3]);
```

## ビルド・テスト

```bash
cargo test
cargo test --features rpoem-adapter,tokio-driver
```

既定 42 テスト、全 feature 有効時 46 テスト green(2026-07-19時点)。

依存クレートゼロ(`unsafe_code = "deny"`)。追加のセットアップは不要。

## 関連プロジェクト

- [RPoem](https://github.com/aon-co-jp/RPoem) — tokio/hyper ベースの
  サーバー側実行基盤。I/O・HTTP は RPoem が担い、RNode.js は
  「Node.js 的な JavaScript ランタイム/モジュールシステムの土台」に
  焦点を当てる(役割重複を避ける分担)。
- [RTypeScript](https://github.com/aon-co-jp/RTypeScript) — トークナイザ+
  型注釈除去トランスパイラ。将来の JS 実行層の連携候補(`CLAUDE.md` 参照)。
- [RJSON](https://github.com/aon-co-jp/RJSON) — 本リポジトリの構成・
  ドキュメント体裁の雛形。

## ライセンス

Apache-2.0 OR MIT

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, redaktionell überarbeitet; noch nicht von Muttersprachlern geprüft)
- **REST wird vollständig abgeschafft; neue APIs entstehen nicht mehr als REST.** APIs, die aus dem Browser oder per `curl` aufgerufen werden, verwenden **unäre Connect-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body). Zwischen Diensten kommen **gRPC bzw. gRPC-web** zum Einsatz, im selben Stil wie die Admin-API von RPoem (`open_runo.admin.v1.Admin`). Streaming wird erst ergänzt, wenn es tatsächlich gebraucht wird.
- **Bestehende REST-Endpunkte werden schrittweise umgestellt, und zwar immer dann, wenn man ohnehin an ihnen arbeitet.** Sobald ein Endpunkt umgestellt ist, wird die alte REST-Route entfernt. Bis zum Abschluss der Migration bleiben die übrigen bestehen, es werden aber **keine neuen mehr hinzugefügt**.
- **JSON wird einheitlich über RS-JSON verarbeitet**: Gelesen wird mit `from_str_secure` bzw. `parse_secure` (mit Obergrenzen für Größe, Verschachtelungstiefe und doppelte Schlüssel), geschrieben mit `rust_json::fast`. `prost` wird nicht in das Browser-WASM aufgenommen, da es die Datei vergrößern würde.
- **Ausnahmen (Schnittstellen, deren Form ein fremder Standard vorgibt und die nicht zu unserer eigenen API gehören)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Drittanbietern (z. B. GitHub), Health-Checks, die Auslieferung statischer Dateien, OTLP und Ähnliches. Diese nehmen, wie vom jeweiligen Standard verlangt, gewöhnliches HTTP entgegen.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines einzelnen Repositorys diese Richtlinie bereits erfüllt, hängt vom jeweiligen Repository ab. Dieser Abschnitt dokumentiert lediglich die Richtlinie; er besagt nicht, dass der Code schon angepasst wurde.

### Русский (перевод выполнен с помощью ИИ и отредактирован; носителями языка пока не проверен)
- **От REST мы полностью отказываемся; новые API больше не создаются в стиле REST.** API, к которым обращаются из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело запроса — JSON). Для взаимодействия между сервисами применяются **gRPC и gRPC-web** — в том же стиле, что и административный API RPoem (`open_runo.admin.v1.Admin`). Потоковая передача данных добавляется лишь тогда, когда в ней действительно возникает необходимость.
- **Существующие REST-эндпоинты переводятся поэтапно — по мере того, как с ними приходится работать.** После перевода старый REST-маршрут удаляется. Пока миграция не завершена, оставшиеся эндпоинты сохраняются, но **новые не добавляются**.
- **Работа с JSON унифицирована на основе RS-JSON**: чтение — через `from_str_secure` или `parse_secure` (с ограничениями на размер, глубину вложенности и повторяющиеся ключи), запись — через `rust_json::fast`. `prost` в браузерный WASM не включается, поскольку это увеличило бы размер файла.
- **Исключения (интерфейсы, форму которых определяет чужой стандарт, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и тому подобное. Такие интерфейсы принимают обычный HTTP, как того требует соответствующий стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Данный раздел лишь фиксирует политику и не означает, что код уже изменён.
