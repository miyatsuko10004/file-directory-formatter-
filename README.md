# Directory Flattener (ファイル階層フラット化ツール)

## 概要
指定したディレクトリ以下の深い階層にあるExcel・PowerPointファイルを再帰的に検索し、**ディレクトリ構造をファイル名に付与した状態で**1つのディレクトリにまとめてコピー（または移動）するPythonスクリプト。

散らばった資料を整理したり、一括でファイル名を管理したい場合に使用。

## 特徴
* **階層構造の維持**: `dir1/dir2/file.xlsx` → `dir1_dir2_file.xlsx` のようにリネームするため、元の場所がわかる。
* **日本語対応**: フォルダ名やファイル名に日本語が含まれていても問題なく動作。
* **進捗表示**: `tqdm` を使用し、処理の進捗状況をプログレスバーで表示。
* **依存管理**: `uv` のスクリプト実行機能に対応しており、手動でのライブラリインストール不要。
* **ログ出力**: 拡張子が対象外でスキップされたファイルがある場合、処理完了後に一覧を表示。

## 動作環境
* Python 3.10 以上
* [uv](https://github.com/astral-sh/uv) (推奨)
* `uv` がない場合は標準の `pip` で `tqdm` をインストールしても動作。

## 使い方

### 1. 準備
このディレクトリに `flatten.py`（Pythonスクリプト）を配置。

### 2. 設定
`SOURCE_DIR` と `DEST_DIR` は、プロジェクトルートにある `.env` ファイルで設定します。
`.env.example` を参考に、プロジェクトルートに `.env` ファイルを作成し、以下の形式で記述してください。

```
# 1. 元のファイルが入っているディレクトリ (絶対パス推奨)
SOURCE_DIR="C:\Users\YourName\Documents\TargetData"

# 2. まとめたファイルを保存するディレクトリ
DEST_DIR="C:\Users\YourName\Documents\FlattenedOutput"
```

注意: `.env` ファイルはGitの管理対象外(`.gitignore` に追加済み)です。各自の環境に合わせて設定してください。

### 3. 実行
ターミナル（PowerShell / Command Prompt / Bash）で以下のコマンドを実行。
`uv` を使用する場合（推奨）:
必要なライブラリ（`tqdm`, `python-dotenv`）が自動でセットアップされ、`.env` ファイルの設定を読み込んで実行されます。

```bash
uv run flatten.py
```

標準の Python を使用する場合:
事前にライブラリを入れる必要がある。

```bash
pip install tqdm
python flatten.py
```

## カスタマイズ

### 対象ファイルを増やしたい
スクリプト内の target_extensions を編集。
```python
# 例: PDFとWordも追加する場合
target_extensions = {'.xlsx', '.xls', '.pptx', '.ppt', '.pdf', '.docx'}
```

### コピーではなく「移動」したい
デフォルトでは安全のため「コピー」になっている。元ファイルを削除して移動させたい場合は、スクリプト内の shutil.copy2 を shutil.move に書き換える。

```python
# 変更前
shutil.copy2(file_path, dest_file_path)

# 変更後
shutil.move(file_path, dest_file_path)
```

### 【高速化】事前スキャンをスキップしたい
ファイル数が数十万件あり、最初のスキャン（数え上げ）待ち時間をなくして即座にコピーを開始したい場合は、スクリプトの 3. 【事前スキャン】 から 4. 【本処理】 のループ部分を以下のコードに置き換える。
※ この場合、進捗バーは「％」や「残り時間」が出ない、「処理件数」のカウントアップのみになる。
```python
    print("即時処理モードで開始します...")

    # ジェネレータ式を作成（メモリ展開せず、見つけ次第処理する）
    files_iterator = (
        p for p in source_dir.rglob('*') 
        if p.is_file() and p.suffix.lower() in target_extensions
    )

    # totalを指定せずにループ
    for file_path in tqdm(files_iterator, desc="Copying", unit="file"):
        try:
            relative_path = file_path.relative_to(source_dir)
            new_filename = "_".join(relative_path.parts)
            dest_file_path = dest_dir / new_filename
            shutil.copy2(file_path, dest_file_path)
        except Exception as e:
            tqdm.write(f"[エラー] {file_path.name}: {e}")
```

## 注意事項
• パスの長さ制限 (Windows):
階層が非常に深く、かつ日本語の長いフォルダ名などが連結されることで、Windowsのパス長制限（約260文字）を超える可能性がある。その場合、出力先フォルダをドライブ直下（例: C:\Data）にするなどして調整する。
• 同名ファイルの扱い:
変換後のファイル名が重複した場合（例: A/B.xlsx と A_B.xlsx が元々存在していた等）、ファイル名の末尾に連番（`_1`, `_2`...）を付与して保存する（上書きはしない）。
