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

## 重要な気づき：JSONはデータ構造ではない！

### ❌ よくある誤解
「プログラムのデータ構造としてJSONというものがある」

### ✅ 正しい理解
**JSONはテキスト形式（シリアライゼーション形式）であって、データ構造ではない**

プログラム上のオブジェクトをJSON/YAMLというテキスト形式に変換しているだけ！

## シリアライゼーション（直列化）とは？

### 直列化の本質
**メモリ上に散在しているオブジェクトを、一本の連続したデータに変換すること**

```
メモリ上のオブジェクト（複雑な構造、ポインタであちこち参照）
    ↓ シリアライゼーション
一本の連続したデータ（先頭から最後まで順番に読める）
```

### 2種類のシリアライゼーション

| 形式 | 例 | 特徴 |
|---|---|---|
| **テキストベース** | JSON, YAML, XML | 人間が読める、デバッグしやすい |
| **バイナリベース** | Protocol Buffers, pickle, MessagePack | コンパクト、高速 |

### 実例：同じオブジェクトの異なるシリアライゼーション

```python
user = {"name": "Alice", "age": 30}

# テキストベース (JSON)
json.dumps(user)
# → '{"name": "Alice", "age": 30}' (文字列、人間が読める)

# バイナリベース (pickle)
pickle.dumps(user)
# → b'\x80\x04\x95...' (バイナリ、人間には読めない)
```

### GitHub Actionsの場合（テキストベース）

```
オブジェクト（メモリ上）
    ↓ シリアライゼーション（toJson, JSON.stringify, json.dumps）
JSON/YAML文字列（テキスト）← バイナリではない！
    ↓ ファイル保存 / ネットワーク送信
    ↓ ファイル読み込み / ネットワーク受信
JSON/YAML文字列（テキスト）
    ↓ デシリアライゼーション（fromJson, JSON.parse, json.loads）
オブジェクト（メモリ上）
```

### 実例：このリポジトリから

```yaml
# example-github-cicd/.github/workflows/json-functions.yml
- run: echo "${CONTEXT}"
  env:
    CONTEXT: ${{ toJSON(github) }}  # オブジェクト → JSON文字列に変換
```

1. `github` = メモリ上の**オブジェクト**
2. `toJSON(github)` = **JSON文字列**に変換（シリアライゼーション）
3. `echo "${CONTEXT}"` = **文字列**をログに出力

### なぜ変換が必要なのか？

| | オブジェクト | JSON/YAML文字列 |
|---|---|---|
| 存在場所 | プログラムのメモリ内のみ | どこでも |
| ファイル保存 | ❌ 不可能 | ✅ 可能 |
| ネットワーク送信 | ❌ 不可能 | ✅ 可能 |
| 他プログラムに渡す | ❌ 不可能 | ✅ 可能 |
| プログラムでの操作 | ✅ プロパティアクセス可 | ❌ ただの文字列 |

### 各言語での例

各言語は独自のオブジェクト形式を持つが、JSON文字列にすれば共通化できる：

```javascript
// JavaScript
const user = { name: "Alice" };        // オブジェクト
const json = JSON.stringify(user);     // → '{"name":"Alice"}' (文字列)
const back = JSON.parse(json);         // → { name: "Alice" } (オブジェクト)
```

```python
# Python
import json
user = {"name": "Alice"}               # dict
json_str = json.dumps(user)            # → '{"name": "Alice"}' (str)
back = json.loads(json_str)            # → {"name": "Alice"} (dict)
```

```go
// Go
import "encoding/json"
user := User{Name: "Alice"}            // struct
jsonBytes, _ := json.Marshal(user)    // → []byte(`{"name":"Alice"}`)
json.Unmarshal(jsonBytes, &user)      // → User{Name: "Alice"} (struct)
```

**JSONは異なる言語間でデータをやり取りするための「共通語」！**

## まとめ

| 誤解 | 正しい理解 |
|---|---|
| JSONはデータ構造 | JSONはテキスト形式（シリアライゼーション形式） |
| JSON形式でデータを持つ | オブジェクトとしてデータを持ち、必要に応じてJSON/YAMLに変換 |
| JSONオブジェクト | メモリ上のオブジェクト（JSON形式に変換可能） |

### GitHub Actionsの場合

- **ワークフローファイル**：YAMLで書く（人間にやさしい）
- **実行時のコンテキスト**：オブジェクトとして扱われる（プログラムが処理しやすい）
- **toJson()の出力**：JSON文字列（外部ツールやファイルに渡しやすい）

YAMLで書いても、JSONで書いても、内部的には同じデータ構造（オブジェクト）になる！
