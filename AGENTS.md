# AGENTS.md

yohaku（栞付きメモ帳エディタ）の開発リポジトリ。実装は **Qt6 / QML (C++)**。
開発中に見つけたクセ・罠・知見はこのファイルの該当セクションへ追記すること。

## 基本

- **名前**: プロジェクト / パッケージ / 実行ファイル / アプリIDは `Yohaku`（`yohaku`）。リポジトリは github:paleblue-2032/yohaku
- CLI: `yohaku [--new|--preview|--find TEXT] [files...]`。`--preview` は Markdown プレビューで起動、`--find` は検索をプリフィル。単一インスタンスで、起動中なら既存ウィンドウにタブ追加
- 設定は `~/.config/yohaku/settings.json` の **1ファイル**（keymap セクションを含む）。栞・セッション・ドラフトは `~/.local/state/yohaku/`
  - `Settings::save` は既存キーを消さないよう読み込み → 更新する
- 主な機能: タブ / 開く・保存 / 栞(自動位置・Ctrl+B の行栞・ファイル栞) / 検索(Ctrl+F) / Markdownプレビュー(Ctrl+P) / 多言語(英・日) / テーマとアクセント / 単一インスタンス
- **タブは復元しない**（起動時は常に新しいタブ）。`session.json` やドラフトは持たない（未保存は終了時の確認ダイアログで扱う）
- 文字サイズは `Ctrl+Shift+Up/Down`（JIS/USどちらでも押しやすい）と `Ctrl+=`/`Ctrl+-`/`Ctrl+0`

## ビルド

- `cmake` / `ninja` + Qt6（`qt6.qtbase` / `qt6.qtdeclarative` / `qt6.qtwayland`）。プロジェクトの flake が供給し、C++ コンパイラは既存の `gcc` で足りる
- **罠**: 同梱 QML のディスクキャッシュがビルド間で衝突し、更新後に古い UI が出ることがある。現在は `QML_DISK_CACHE_PATH` を実行ファイルのハッシュで分けて有効化して回避（古い UI を避けつつ起動を速く）。以前は `QML_DISABLE_DISK_CACHE=1` で無効化していた
- テストのキー送出には `wtype`（niri/Wayland）を使う

## 開発用フック

- `YOHAKU_SCREENSHOT=<path>` でウィンドウを PNG に保存して終了（見た目確認用）。`YOHAKU_SCREENSHOT_DELAY_MS=<ms>` で遅延
- ファイル選択: Linux で xdg-desktop-portal があれば `QT_QPA_PLATFORMTHEME=xdgdesktopportal` を自動設定し、その環境の標準ファイルアプリ（Nautilus 等）のダイアログを使う。無ければ従来の Qt ダイアログ。Windows/macOS はネイティブ

## 日本語入力・フォント

- niri 上の日本語入力は Qt6 Wayland の text-input-v3 経由（fcitx5-qt があればなお可）
- 本文は等幅（Noto Sans Mono CJK JP 優先。無ければ各OSの等幅）。UI はサンセリフ（`App.uiFont` / `App.monoFont`）

## QML の罠

- **メニュー**: Qt の `Menu`/`MenuBar` だとキー入力を受け取らず（↑↓/Enter が効かない）、`Menu` に custom `background` を付けると幅が 0 になって消える。→ `qml/AppMenu.qml`（`Popup` + `Column`）で自前実装し、`open()` + `parent`/`x`/`y` で開く（`Popup` に `popup()` は無い）。↑↓/Enter/Esc とホバー、角丸+影に対応
  - 自前実装にすると `Action{shortcut:...}` が持っていた実ショートカットが消える。`Shortcut { sequence: "Ctrl+S" }` 等を明示的に用意すること（表示だけでは効かない）
- **フォーカス**: 「起動時のエディタへの forceActiveFocus」を入れ忘れると、クリックするまで入力もカーソルキーも効かない（ウィンドウがアクティブでないと activeFocus が入らない）。`Window.onActiveChanged` + 起動時 `requestActivate()` + タブ切替時の focus で解決
- **モーダルな `Dialog`（Popup）ではウィンドウ側の `Shortcut` が発火しない**（非モーダルの `Popup` であるメニューでは発火する）。ダイアログのキー操作は footer などの Item の `Keys` で処理し、`onOpened` で `forceActiveFocus` + `Qt.callLater` で再フォーカスする
- QML が `model.xxx`（DelegateModel のロール）を見ている場合、C++ 側で独自シグナルを出しても再評価されない。`dataChanged(index, index, { Role... })` を必ず出すこと（「保存してもタブの未保存マークが消えない」不具合の原因だった）

## 透過

- Qt Quick は同じ領域を二重に塗ると α が合成されて不透明に近づく（例: 0.9 の下地 + 0.9 の矩形 = 0.99）。ウィンドウを `color: "transparent"` にし、**各領域を一度だけ塗る**こと（ヘッダ/タブ/ステータス/余白は設定値 α、入力欄は α+0.05）。設定は `Settings.windowOpacity`（0.4〜1.0、既定 0.9）で設定メニューの なし/弱/中/強
- 透過はアプリ側の α のみ。niri 側は `window-rule { match app-id="^yohaku$"; opacity 1.0 }` で全体の 0.9 透過を打ち消している（すりガラス的な blur は niri 非対応）
- 検証: 起動時 `YOHAKU_SCREENSHOT` の PNG α を測る + `niri msg action screenshot-screen` で合成結果を見る（`screenshot-window` は半透明を中立背景で合成するため見え方が実際と異なる。またフォーカス中ウィンドウしか撮れない）

## 性能

- 行番号ガターは可視行のみ描画（`visStart`/`visCount`）。全行再計測は 60ms デバウンス

## プレビュー描画の罠

- QQuickTextEdit では QTextDocument の**ブロック背景が描画されない**。コードブロックの背景は QML 側（positionToRectangle + 項目自身の x/y オフセット）で描く
- **言語付きコードフェンスは Qt が frame で包み**背景が効かなくなるので、パース前にフェンス記号だけへ正規化する
- positionToRectangle は関数なので、バインドでは contentHeight/width を読んで依存を作る（でないとレイアウト変更に追従しない）

## 配布

- Linux は Nix (`nix run` / `home.packages`)。macOS/Windows は GitHub Actions (`.github/workflows/build.yml`) が Qt 同梱の zip を生成
- GitHub Release を tag `v*` で作成（macOS .app zip / Windows DLL同梱 zip / Linux tarball）。ワンライナーは `scripts/install.sh`（macOS/Linux）と `scripts/install.ps1`（Windows）
- CI の Qt は macOS/Linux 6.11、Windows 6.8（macOS 6.8 は AGL 要求でリンク不可、Windows に 6.11 が無い）

## コミット

- このリポジトリのコミットに `Co-authored-by: CommandCodeBot` を付けない（ユーザー指示）。過去履歴からも除去済み（`git filter-branch --msg-filter` + force push）
