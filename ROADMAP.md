# Roadmap — Aster-Launcher

Rust + GPUI 製 Minecraft ランチャーの開発ロードマップ。根拠は [DESIGN.md](DESIGN.md)（確定方針 D1–D32、懸念点レジスタ R1–R33）で、本文と矛盾する場合は DESIGN.md が正。

- ステータス: **Draft**（2026-11-08 作成。DESIGN.md §ロードマップ案 を具体化したもの）
- 工数見通し: 未確定（DESIGN.md R11）。Milestone 間の順序は確定、各 M の期間は未定
- 完了条件（全 M 共通）: aster-core の単体テスト＋CLI による E2E（シェルスクリプト＋JSON アサート）がパス。UI の自動検証は初期見送り（DESIGN.md §実装 constraints）

## 全体像

```
M0 先行検証 ──> M1 骨格 ──> M2 プラグイン実証 ──> M3 配布 ──> M4 周辺 ──> M5 以降（並列拡張）──> 公開前ゲート
```

順序の根拠:

1. **コンテンツアドレスストアは最初から作る**（DESIGN.md §14）。後付けはデータ構造の破壊的変更になるため M1 で導入
2. **プラグイン ABI は早期に実証する**（R2/R24 は設計上最大のリスク）。M2 で fabric 解決器を通じて WASM ゲストが起動パイプラインを表現できるか答え合わせする
3. **GPUI on Linux の検証が全作業のブロッカー**（R20）。M1 着手前に M0 で潰す
4. M5 以降は互いに独立して並列化可能（CF / Microsoft 認証 / BTA・Cleanroom / 競合リスト / Windows）

## M0 先行検証（ブロッカー解消）

目的: M1 の技術的前提を壊さないことだけを確認する。コードは書かない。

| 検証項 | 対応懸念 | 受け入れ条件 |
|---|---|---|
| GPUI on Linux（X11/Wayland、Vulkan 要否） | R20, R12 | 検証用サンプルが両セッションでウィンドウ描画＋日本語テキスト表示＋IME 入力ができることが確認済みであること |
| CI（GitHub Actions）での Linux ビルド・起動 smoke | R28 | ヘッドレス環境でソフトウェアレンダリング（mesa/swiftshader 等）を使ったビルド＋起動 smoke が通ることが確認済みであること |
| GPUI バージョン固定運用の確度 | R1 | crates.io 版 GPUI の現行バージョンで上記が成立し、Zed リポジトリ追従の頻度・手順が把握されていること |

完了したら GPUI バージョンを Cargo.lock 起点で固定し、M1 へ。

## M1 骨格

目的: 「インスタンスを作って vanilla をオフライン認証で起動する」最小単位を、core と CLI と最小 UI で完走する。

スコープ:

- ワークスペース構成: `aster-core` / `aster-cli` / `aster-ui`（DESIGN.md §アーキテクチャ）
- aster-core: インスタンス定義 TOML＋状態 DB（redb、D17）＋コンテンツアドレスストアの基盤（store ディレクトリ＋redb インデックス、§データモデル）
- 起動パイプラインの vanilla 実装: Resolver（公式 piston-meta、D25）→ Fetcher（artifacts/assets 取得＋sha256 検証）→ Installer（ハードリンク配置）→ Launcher（JVM 起動）→ Runner（終了コード・ログ保持）
- offline 認証プラグイン（M1 時点では WASM ではなく core 内の直接実装でよい。WASM 化は M2 で ABI と一緒に移行）
- CLI 最小セット（DESIGN.md §CLI）: `aster instance create|list|show|launch|delete`、`aster account list|add --kind offline`、`aster java list|install`（Temurin API、D8）、全て `--format json` 対応・非対話・終了コード意味論あり
- 単一インスタンスロック（D29）。2 個目プロセスは「既に起動中」を識別可能な JSON エラー＋終了コードで拒否
- GPUI 最小画面: インスタンス一覧＋起動ボタン

受け入れ条件（テスト可能）:

- `aster instance create --name test --version 1.21.1` が TOML 定義を生成し、`aster instance launch test` が JVM プロセスを起動して終了コード 0 で戻る（E2E シェルスクリプト＋JSON アサート）
- 同一マシンで 2 個目の `aster` プロセス（GUI/CLI 問わず）が規定の終了コードで即拒否される
- 取得アーティファクトが sha256 検証済みで store に配置され、インスタンス mods/ からハードリンクで参照される（リンク数・ハッシュの照合テスト）
- `aster java install` で Temurin から JDK が store に取得される（対象: Linux x64/aarch64 の Java 8/17/21。提供有無は M0/M1 で確認、DESIGN.md §2 要検証）
- core 単体テスト: コンテンツアドレスストアの参照カウント・起動時 sweep、名前ベース UUID v3（D23。v3/v5 は実装時に業界慣行を確認）

対応懸念・検証: R16（D25 で解決、要検証は M1 内）, R17, R18, R20, R22（基本形は M1 で定義）, R23（EXDEV フォールバック方式を実装時決定）

## M2 プラグイン実証

目的: プラグインホスト（wasmtime/WASI preview2＋wit、D1）を立ち上げ、バンドルプラグイン方式で modded 起動まで実証する。設計リスク R2/R24 の答え合わせが最優先。

スコープ:

- `aster-plugin-host`: wasmtime ランタイム、wit 世界定義（プラグイン種別 4 面、§プラグイン契約）、能力モデル（`net-fetch` allowlist / `fs` scope / `secrets` / `spawn` 原則禁止）
- wasi-http 外向き通信の host プロキシ＋allowlist ゲート
- M1 の offline 認証を WASM ゲストへ移行（ABI の最初の実用例）
- modloader 解決器プラグイン: **fabric を最初に**（R2 の方針通り動的実行を必要としない）。必要ライブラリ取得・起動引数・パッチ適用まで
- バンドルプラグインのビルド・読み込み基盤（ビルド時 wasm 化、実行時バンドル読み込み）

受け入れ条件:

- 能力なしの guest は外向き HTTP・任意 fs 書き込み・プロセス起動のいずれも拒否される（plugin-host 単体テストで allowlist 検査を網羅、R24）
- `net-fetch { allowlist }` で allowlist 外ドメインへのリクエストがホスト側で遮断される（同上）
- fabric インスタンスが作成→起動→終了コード 0 で戻る（CLI E2E）
- プラグイン ABI v1（メジャー 1 のみ運用、D31）として wit 世界が文書化されている

対応懸念・検証: R2（legacyforge は翻訳メタデータの保守前提で M5 以降、本 M では fabric のみで判断）, R24（必須検証）, R6（設計は secrets capability で切り離し済み、実装はまだ）

## M3 配布

目的: mod の探し・入れ・更新というランチャーの中核体験を完成させる。

スコープ:

- Modrinth 配布サイトプラグイン: 検索・プロジェクト/バージョン情報・取得（§4）
- 依存解決の実運用: mod 追加・更新・バージョン変更時に Resolver 再走（R22）。愚直に、SatSolver なし
- コンテンツアドレスストア＋ハードリンクの完全動作: 重複 jar の物理共有、参照カウント、sweep、手動編集検出（リンク数・ハッシュ照合、R5）
- 起動時 sweep（単純マーク&スイープ）
- mclo.gs 投稿（#9）: マスキング＋投稿前プレビュー＋同意。CLI からも実行可

受け入れ条件:

- `aster mod search|add`（命名は実装時）で Modrinth から mod を検索・追加し、依存が再解決されて起動可能な構成になる（E2E）
- 同一 jar を持つ 2 インスタンス間で store オブジェクトが 1 つに共有され、一方を削除しても他方が壊れない（リンク数テスト）
- mods/ 内 jar をユーザーが手動で壊した場合、起動時に検出され store から再リンクされる
- mclo.gs 投稿で既知のトークンパターンがマスキングされ、同意なしでは送信されない（マスキング網羅性テスト、R25）

対応懸念・検証: R13（M3 着手前に mclo.gs が Cleanroom/BTA 系ログを受理するか動作確認）, R5, R22, R25

## M4 周辺

目的: 体験を整える周辺機能を一括投入する。相互依存が薄く並列化しやすい。

スコープ（各項は独立して取り込み可能）:

- テーマ: トークンセット確定（D26、UI 実装前にロック）＋ Mokka/Dracula バンドル＋matugen 外部バイナリ連携（D9）。UI はトークン参照のみ
- スキンライブラリ: ローカル PNG 管理＋新旧形式両対応・Prism 式修復変換（D22）。適用は認証プロバイダのサポート範囲で動的決定
- 内蔵ターミナル: `aster-term`（libghostty-rs、D19）でゲーム出力表示＋runclient 的一時環境（開発者モード）
- Modpack エクスポート: mrpack（公式仕様準拠）／CF zip／MMC zip（§5。CF/MMC は非公式知識の要検証あり）
- i18n 基盤: .po（gettext）＋LLM 翻訳＋CI 原文変更検知→自動 PR（D30）。D15 対象言語の投入タイミング（一括 or 段階）は実装時判断

受け入れ条件（例）:

- トークンセットが文書化され、UI コード内に色リテラルが存在しない（grep による CI チェック）
- 旧形式 64x32 スキンが読み込まれ 64x64 に修復変換される（画像比較テスト）
- インスタンスから mrpack が公式仕様に準拠してエクスポートされ、Modrinth 側で受理される形式検証ができる
- CLI E2E: エクスポート 3 形式が生成され、構造アサートを通過する

対応懸念・検証: R19（LLM 翻訳品質は残る）, R12（GPUI フォント品質、M0 で基本確認済みのはず。未解決ならここで再判定）

## M5 以降（順序不固定・並列可）

各項目は独立。優先度は公開時の訴求とユーザビリティで再判断してよい。

| 項目 | 内容 | 主な懸念 |
|---|---|---|
| CurseForge 対応 | 配布サイトプラグイン。key はユーザー各自取得（D6）、オプトイン外 mod は取得不可を UI 明示 | R3（要検証・継続監視） |
| Microsoft 認証 | account-microsoft プラグイン（D4）。完成で公式スキン API も自然に効く | R6（client ID 戦略） |
| BTA! / Cleanroom | 独自メタデータリポジトリ（GitHub、D24）＋modloader プラグイン | R4, R30（BTA リリース取得の機械化） |
| legacyforge 等 Forge 系 | 静的メタデータ翻訳の保守運用を前提に | R2 |
| 競合リスト | クライアント照合＋配信インフラ運用開始（GitHub 正本、D7/D18） | R8（インフラ運用開始の判断） |
| macOS 実機対応 | 作者が実機を持たないため後回し（D10）。dmg/notarize は U7 残項 | — |
| Windows | 段階的対応（D3）。更新ボタン（D32）。OS 分岐は `aster-ui/src/platform/` に集中 | — |

## 公開前ゲート（リリース条件）

順序の外にあるが、公開リリースには以下の完了が必須:

- R32: 「Aster」名称重複調査の完了。調査結果次第で改名判断
- R33: AGPL-3.0 と WASM プラグインの法的境界の方針明文化（wit 越し分離の明記）
- R14: 外部プラグイン配布開始時の案内（無管理＋能力表示の説明）
- 配布形式: Nix / AUR / AppImage（Flatpak は R10/R26 の検証後、D10）
- 競合リスト配信インフラの運用開始（公開と同時運用が自然）

## 判断保留事項（ロードマップ上の未確定）

- M4 の i18n 言語投入タイミング（一括 or 段階、D15 関連）
- R23 EXDEV フォールバックの最終方式（コピー or 同居強制）
- D23 UUID v3/v5 の最終確認（M1 実装時）
- M5 以降のマイルストーン分割（工数見通し確定後に M5/M6…へ具体化）
