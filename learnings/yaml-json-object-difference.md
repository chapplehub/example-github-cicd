# YAML vs JSON vs オブジェクト - GitHub Actionsでのデータ表現

## 疑問

- GitHub ActionsのワークフローはYAMLで書くのに、なぜコンテキスト（`github`, `matrix`, `env`など）は「JSONオブジェクト」と呼ばれるのか？
- `toJson()`関数は何を何に変換しているのか？

## 答え：YAMLもJSONも同じデータ構造を表現している

### 処理フロー

```
1. 開発者がYAMLで記述（人間にやさしい）
   ↓
2. GitHub ActionsがYAMLをパース
   ↓
3. メモリ上にデータ構造（オブジェクト）として保存
   ↓
4. ワークフロー実行時にオブジェクトとしてアクセス
   ↓
5. toJson()で必要に応じて文字列に変換
```

### 実例（このリポジトリから）

```yaml
# example-github-cicd/.github/workflows/matrix.yml
strategy:
  matrix:                                             # YAMLで定義
    os: [ubuntu-latest, windows-latest, macos-latest]
runs-on: ${{ matrix.os }}                             # オブジェクトとしてアクセス
```

この `matrix` は：
- **記述時**：YAML形式
- **実行時**：メモリ上のオブジェクト `{os: ["ubuntu-latest", ...]}`
- **文字列化時**：`toJson(matrix)` → `'{"os":["ubuntu-latest",...]}'`

### YAMLとJSONの関係

実は**YAMLはJSONのスーパーセット**に近い形式：

```yaml
# このYAML
matrix:
  os: [ubuntu-latest, windows-latest]

# このJSONと完全に同じ構造
{"matrix": {"os": ["ubuntu-latest", "windows-latest"]}}
```

重要：**有効なJSONは全て有効なYAML**です！

## 3つの形式の違い

| 形式 | 実体 | 用途 | 例 |
|---|---|---|---|
| **YAML** | テキスト | ワークフロー記述 | `os: [ubuntu-latest]` |
| **オブジェクト** | メモリ上のデータ構造 | プログラム内部での処理 | `matrix.os` でアクセス可能 |
| **JSON文字列** | テキスト | ファイル保存、外部ツールへの受け渡し | `'{"os":["ubuntu-latest"]}'` |

## toJson()関数の役割

```yaml
steps:
  # オブジェクトのプロパティにアクセス
  - run: echo "${{ matrix.os }}"

  # オブジェクト全体を文字列に変換してファイルに保存
  - run: echo '${{ toJson(matrix) }}' > matrix.json

  # オブジェクト全体を文字列に変換してAPIに送信
  - run: curl -X POST https://api.example.com -d '${{ toJson(github.event) }}'
```

**toJson()は：**
- 入力：メモリ上のオブジェクト（データ構造）
- 出力：JSON形式の文字列（テキスト）

## なぜ「JSONオブジェクト」と呼ばれるのか？

- YAMLとJSONは同じデータ構造を表現する2つの記述形式
- GitHub Actionsの内部ではオブジェクト（データ構造）として扱われる
- 文字列化する際にJSON形式が使われる（YAMLよりコンパクトで機械的に処理しやすい）
- そのため「JSONオブジェクト」という呼び方が一般的

## まとめ

- **ワークフローファイル**：YAMLで書く（人間にやさしい）
- **実行時のコンテキスト**：オブジェクトとして扱われる（プログラムが処理しやすい）
- **toJson()の出力**：JSON文字列（外部ツールやファイルに渡しやすい）

YAMLで書いても、JSONで書いても、内部的には同じデータ構造（オブジェクト）になる！
