# Jupyter Notebookを使用したHTML自動更新レポートの技術解説

## 1. 目的と位置づけ

本書は、**Jupyter Notebook + Papermill + nbconvert + ブラウザの自動再読み込み**を組み合わせ、定期更新するHTMLレポートを作る方式を解説する。今後のレポートアプリで共通利用するため、仕組みと実装上の注意点を整理する。

- **Notebook**：レポート内容（データ取得、計算、グラフ、文章）を定義する。
- **Papermill**：Notebookを別プロセス相当の実行環境で先頭から実行し、結果付きNotebookを出力する。
- **nbconvert**：結果付きNotebookをブラウザ閲覧用HTMLに変換する。
- **制御用Python**：定期実行、ファイル保存、ブラウザ起動、エラー処理を担当する。
- **ブラウザ**：生成済みHTMLを表示し、指定間隔で再読み込みする。

> **基本方針：レポートの中身はNotebook、実行と更新の管理は通常のPythonに分離する。**

## 2. 全体構成

```text
                 制御用Python（常駐）
                  create_realtime_report.py
                           │
             ① パラメータを渡して実行
                           ▼
                  Papermill
                           │
               入力Notebookを実行
                           ▼
             body_insp_measurement_trend.ipynb
             ├─ SQLiteから測定データ取得
             ├─ pandasでデータ処理
             └─ Matplotlibでグラフ表示
                           │
             ② 実行結果付きNotebookを保存
                           ▼
               output_measurement_trend.ipynb
                           │
                  ③ nbconvertで変換
                           ▼
              body_insp_measurement_trend.html
                  （デスクトップに保存）
                           │
               ④ HTMLにmeta refreshを追加
                           ▼
                 ブラウザで表示
                           │
                ⑤ ブラウザが定期再読込
                           │
        制御用Pythonは一定時間待って①に戻る
```

**重要：** Pythonによる「HTMLファイルの再生成」と、ブラウザによる「HTMLファイルの再読み込み」は**別々の仕組み**である。

## 3. 各技術の動作原理

### 3.1 Jupyter Notebook（`.ipynb`）

Notebookは、Markdownセルとコードセル、実行結果などをJSON形式で保存するファイルである。

- Markdownセル：タイトル、説明、見出し、ページ内リンクなど
- コードセル：SQLite検索、pandas集計、Matplotlib描画など
- 出力：文字列、表、画像、グラフなど

Notebook単体でも手動実行できる。レポートのレイアウトとデータ処理をNotebook側で確認してから、自動実行に組み込める点が利点。

**添付アプリの実例：** `measurement_result` テーブルから、検査機・検査項目・基準日時に対応する最新ロットを探し、ロット内の測定データを取得する。検査項目ごとに5台分のグラフを3行×2列のFigureに配置する。これは個別アプリ固有の処理であり、共通基盤には含めない。

### 3.2 Papermill：Notebookの自動実行

PapermillはNotebookをプログラムから実行し、結果を**別のNotebook**として保存できるツールである。

```python
import papermill as pm

pm.execute_notebook(
    NOTEBOOK_PATH,
    OUTPUT_NOTEBOOK_PATH,
    cwd=str(BASE_DIR),
    parameters={"STD_DATETIME": std_datetime},
)
```

| 引数 | 意味 |
|---|---|
| `NOTEBOOK_PATH` | 実行する元Notebook |
| `OUTPUT_NOTEBOOK_PATH` | 実行結果付きNotebookの保存先 |
| `cwd` | Notebook実行時の作業ディレクトリ |
| `parameters` | Notebookに渡すパラメータ |

**元Notebookは書き換えない。** 実行結果は出力Notebookに保存されるため、元のNotebookをGit管理しやすい。

#### パラメータ注入の仕組み

Papermillはパラメータ指定によりNotebookの実行時変数を設定できる。一般的にはNotebookに `parameters` タグ付きのパラメータセルを用意する。

```python
# Notebook側のパラメータセル（parametersタグを付与）
STD_DATETIME = "2026-11-11 18:06:00"
```

Papermillは指定値を注入するセルを追加して実行する。タグを付けない場合でも動作するケースはあるが、意図しない上書きや警告を避けるため、**標準テンプレートでは `parameters` タグを明示する**ことを推奨する。

`cwd` は相対パス解決に影響する。Notebookの `../data/...` のようなパスは、この作業ディレクトリを基準に解釈される。

### 3.3 nbconvert：NotebookからHTMLへの変換

Papermillの出力Notebookには実行結果が含まれる。nbconvertはこれを静的HTMLに変換する。

```python
subprocess.run(
    [
        sys.executable, "-m", "jupyter", "nbconvert",
        "--to", "html",
        "--no-input",
        "--no-prompt",
        "--output", str(OUTPUT_HTML_PATH),
        OUTPUT_NOTEBOOK_PATH,
    ],
    check=True,
)
```

| 指定 | 意味 |
|---|---|
| `sys.executable` | 制御用Pythonを動かしているPython環境を使用 |
| `-m jupyter nbconvert` | Jupyterの変換機能を実行 |
| `--to html` | HTML形式に変換 |
| `--no-input` | コードセルの入力ソースをHTMLに表示しない |
| `--no-prompt` | `In [1]` 等の入力・出力プロンプトを表示しない |
| `--output` | 出力先HTMLファイルを指定 |
| `check=True` | コマンド失敗時に例外を発生させる |

`--no-input` は**HTML表示からコードを隠す指定**であり、Notebookのコード実行を省略する指定ではない。実行はすでにPapermillが行っている。

**HTMLは原則として静的な成果物**であり、表示するだけではSQLiteへの再接続やNotebookの再実行は起こらない。

### 3.4 HTMLの自動再読み込み（meta refresh）

生成したHTMLの `<head>` 内に次のタグを追加する。

```html
<meta http-equiv="refresh" content="60">
```

- `http-equiv="refresh"`：ブラウザにページ再読み込みを指示する。
- `content="60"`：60秒後に再読み込みする。
- ブラウザは同じHTMLファイルを読み直す。**NotebookやPythonを実行するわけではない。**

添付アプリは、生成されたHTMLをUTF-8で読み込み、最初の `<head>` の直後にタグを挿入して書き戻す。

```python
html = html_path.read_text(encoding="utf-8")
refresh_tag = '<meta http-equiv="refresh" content="60">'
html = html.replace("<head>", f"<head>\n{refresh_tag}", 1)
html_path.write_text(html, encoding="utf-8")
```

この例は添付コードの要点を抜粋したもの。実コードでは `<head>` が見つからなければ例外を発生させる。

## 4. 2種類の「60秒」の違い

添付の制御用Pythonには次の2つの設定がある。

```python
UPDATE_INTERVAL_SECONDS = 1 * 60
BROWSER_REFRESH_SECONDS = 60
```

| 設定 | 担当 | 起きること |
|---|---|---|
| `UPDATE_INTERVAL_SECONDS` | Python | 次のNotebook実行まで待つ |
| `BROWSER_REFRESH_SECONDS` | ブラウザ | 同じHTMLファイルを読み直す |

### 実際のタイミング

**Pythonの60秒待機は、レポート生成が完了してから始まる。** したがって生成に20秒かかれば、次の生成開始は約80秒後となる。

```text
時刻（例）  Python側                         ブラウザ側
00秒       Notebook実行開始
20秒       HTML生成・refreshタグ追加      初回ページを開く
20～80秒   60秒待機
80秒       次のNotebook実行開始           ページを再読み込み
100秒      次のHTML生成完了
140秒      次のNotebook実行開始           ページを再読み込み
```

※ 上記は生成時間20秒、ブラウザ初回表示を20秒時点と仮定した説明例。実際の処理時間やブラウザのタイミングは変動する。

**注意：** 両方を60秒にしても、更新と再読み込みが厳密に同期するわけではない。ブラウザが古いHTMLを再表示する場合もあり、次の再読み込みで新しい内容が表示される。生成中のファイルを読み込むタイミングが重なる可能性もある。

## 5. 添付アプリの処理順序

1. 制御用Pythonが起動し、実行環境を表示する。
2. ユーザーに基準日時を入力させる。Enterのみなら、その更新回の現在日時を使用する。
3. `create_html_report()` を呼ぶ。
4. Papermillが入力Notebookを実行し、一時Notebookを保存する。
5. nbconvertが一時NotebookをHTMLに変換する。
6. HTMLの `<head>` に自動再読み込みタグを追加する。
7. `finally` 節で一時Notebookを削除する。
8. 最初の成功時のみ `webbrowser.open()` でHTMLを開く。
9. 指定秒数待機して3に戻る。
10. `Ctrl+C` で終了する。

### 基準日時の扱い

- 日時入力あり：指定日時を以後の更新でも使用する（固定時点のレポート）。
- 入力なし：毎回 `datetime.now()` を取得する（現在時点のレポート）。
- `normalize_datetime()` は複数の日付・時刻表記を `YYYY-MM-DD HH:MM:SS` に統一する。

### エラー発生時

Notebook実行またはHTML生成に失敗すると例外を表示し、次の更新時に再試行する。`finally` によって一時Notebookは削除対象となる。ただし、**失敗した回のHTMLが必ず最新・正常になるわけではない**。以前のHTMLが残る場合がある。

## 6. ファイルの役割とGit管理

```text
report_app/
├── create_realtime_report.py        # 実行・変換・更新の制御
├── body_insp_measurement_trend.ipynb # レポート定義（元Notebook）
├── output_measurement_trend.ipynb   # 一時ファイル（実行後に削除）
└── ...                              # DB、設定、依存パッケージ等

デスクトップ/
└── body_insp_measurement_trend.html # ブラウザで閲覧する成果物
```

**推奨Git運用：** 元Notebookと制御用Pythonを管理対象とし、一時Notebookや生成HTMLは通常コミットしない。Notebookを手動で実行した場合は、コミット前に出力をクリアして保存し、差分を確認する。

## 7. 標準レポートアプリにするための設計方針

### 共通化する部分

- Notebookパス、HTML出力先、更新間隔などの設定
- Papermill実行
- nbconvert変換
- meta refreshタグ追加
- ブラウザ初回起動
- ループ・終了・エラー処理

### Notebookごとに変える部分

- データソース（SQLite、CSVなど）
- 集計ロジック
- 図表・文章・見出し・目次
- 実行パラメータ

**KISS方針：** 最初から大規模な共通ライブラリを作らず、制御用Pythonをテンプレートとしてコピーし、設定値とNotebookだけ差し替える方法が分かりやすい。

### 標準化前に確認したい改善候補（未実装）

1. **HTMLの安全な置き換え**：一時HTMLを完成させてから最終ファイルへ置き換えることで、書き込み途中の読み込みを避ける。
2. **更新失敗の可視化**：HTML内に最終生成日時を表示し、古いレポートを最新と誤認しないようにする。
3. **更新周期の定義**：「前回完了から60秒」か「毎分00秒」かを明確にする。
4. **ブラウザの閲覧位置**：meta refreshによってスクロール位置が変わり得るため、ページ内リンクを使う長いレポートでは更新方法を検討する。
5. **Papermillのparametersセル**：パラメータセルのタグ付けをテンプレートの標準にする。
6. **リソースとログ**：長時間運用時のエラー履歴、Notebookの実行時間、メモリ消費を確認する。

これらは添付アプリに実装済みという意味ではない。

## 8. 動作確認チェックリスト

- [ ] Notebook単体で最後まで実行できる。
- [ ] Papermillがパラメータを正しく渡せる。
- [ ] 元Notebookの内容・出力がPapermill実行で変更されない。
- [ ] 実行結果付きNotebookからHTMLが生成される。
- [ ] HTMLにPythonコードが表示されず、図表は表示される。
- [ ] HTMLにmeta refreshタグが1個入る。
- [ ] ブラウザが生成後のHTMLを再読み込みする。
- [ ] データ更新後にレポート内容が更新される。
- [ ] DB接続失敗時に次回更新で復帰できる。
- [ ] `Ctrl+C` で停止できる。
- [ ] Gitの差分に一時的な実行結果が混入しない。

## 9. よくある誤解

| 誤解 | 正しい理解 |
|---|---|
| HTMLを再読み込みするとNotebookも再実行される | 再実行するのはPython + Papermill。HTMLは結果を読み直すだけ |
| nbconvertがNotebookを実行している | この構成ではPapermillが実行し、nbconvertは結果を変換する |
| `--no-input` はコードを実行しない | HTMLからコード表示を省く指定 |
| 更新間隔60秒なら毎分ちょうど更新される | 添付コードは処理完了後に60秒待機する |
| Papermillが元Notebookを上書きする | 入力と出力を別パスにしているので元Notebookはそのまま |
| HTMLを開くためにFlaskサーバが必須 | 今回はローカルHTMLをブラウザで開くため不要 |

## 10. まとめ

この方式は、**Notebookを「レポート定義」、Papermillを「実行装置」、nbconvertを「HTML変換装置」、制御用Pythonを「運転管理」、ブラウザを「表示装置」**として分離している。

そのため、データ取得とグラフだけNotebookごとに作り替えれば、同じ実行・表示の仕組みを別のレポートに流用しやすい。まずは既存コードをシンプルな標準テンプレートとして使い、実際の運用で必要になった部分だけ共通化するのがよい。

---

### 参照した添付ファイル

- `create_realtime_report(20261120-164106).py`
- `body_insp_measurement_trend(20261120-164106).ipynb`

※ 本書の「添付アプリの処理」は上記ソースに基づく。「標準化前の改善候補」および一般的な技術原理の説明は、今後の設計に向けた補足である。
