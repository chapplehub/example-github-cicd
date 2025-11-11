# VS Code が YAML ファイルのエラーを検出する仕組み

## 疑問

GitHub Actions の YAML ファイルで `run-on` というタイポを書いたら、VS Code がエラーを表示した。

```yaml
jobs:
  run:
    run-on: ubuntu-latest  # 本当は runs-on
```

エラー内容：
```
Unexpected value 'run-on' [Ln 5, Col 5]
Required property is missing: runs-on [Ln 5, Col 5]
```

TypeScript ならコンパイラがあるから型エラーを検出できるのは分かるが、なぜ GitHub Actions のような外部サービスの内部エラーまで検出できるのか？

## 答え：JSON Schema による検証

VS Code は **JSON Schema** を使って YAML ファイルを検証している。

### 仕組みの全体像

```
TypeScript の場合:
  .ts ファイル → TypeScript コンパイラ → 型チェック → エラー検出

YAML の場合:
  .github/workflows/*.yml → JSON Schema → バリデーション → エラー検出
```

### 詳細な流れ

1. **ファイルパスで自動判定**
   - VS Code の YAML 拡張が `.github/workflows/*.yml` というパスを検出
   - 自動的に「これは GitHub Actions Workflow だ」と判断

2. **JSON Schema の適用**
   - [SchemaStore.org](https://www.schemastore.org/) から GitHub Actions Workflow Schema を取得
   - このスキーマには構造のルールが厳密に定義されている：
     - `runs-on` は必須フィールド
     - `run-on` というフィールドは存在しない
     - `runs-on` は文字列または配列
     - など

3. **リアルタイムバリデーション**
   - YAML 拡張が YAML をパースして Schema と照合
   - ルール違反を検出してエラー表示

### JSON Schema の例（簡略版）

```json
{
  "type": "object",
  "properties": {
    "jobs": {
      "type": "object",
      "patternProperties": {
        "^[_a-zA-Z][a-zA-Z0-9_-]*$": {
          "required": ["runs-on"],
          "properties": {
            "runs-on": {
              "description": "The type of machine to run the job on",
              "oneOf": [
                { "type": "string" },
                { "type": "array" }
              ]
            }
          }
        }
      }
    }
  }
}
```

## なぜ GitHub 内部エラーまで分かるのか？

GitHub が公式に JSON Schema を公開しているから：

1. GitHub 自身がワークフローの構造を JSON Schema で定義
2. それを SchemaStore.org で公開
3. VS Code の YAML 拡張が自動的に利用

→ GitHub が実際に使っている構造定義と同じものを使ってローカルで検証できる

## 使われている拡張機能

以下のどちらか（または両方）が JSON Schema を利用している：

- **Red Hat の YAML 拡張** (`redhat.vscode-yaml`)
- **GitHub Actions 拡張** (`github.vscode-github-actions`)

これらは SchemaStore から自動的に Schema を取得してバリデーションする。

## 他の設定ファイルでも同じ仕組み

SchemaStore.org には 600 以上の設定ファイルの Schema がある：

| ファイル | スキーマ |
|---------|---------|
| `package.json` | npm package schema |
| `tsconfig.json` | TypeScript config schema |
| `docker-compose.yml` | Docker Compose schema |
| `.eslintrc.json` | ESLint config schema |
| `azure-pipelines.yml` | Azure Pipelines schema |

すべて同じ仕組みでエディタがエラーチェックしている。

## 実際の Schema を確認する方法

```bash
# GitHub Actions Workflow Schema
curl https://www.schemastore.org/github-workflow.json | jq .
```

または SchemaStore.org で検索：
https://www.schemastore.org/json/

## まとめ

- TypeScript にコンパイラがあるように、YAML にも「構造の定義書（JSON Schema）」がある
- VS Code の YAML 拡張は、ファイルパスを見て適切な Schema を自動適用
- GitHub などのサービスが公式に Schema を公開しているため、ローカルで正確なエラー検出ができる
- この仕組みは GitHub Actions だけでなく、多くの設定ファイルで使われている標準的な方法

## 参考リンク

- [SchemaStore.org](https://www.schemastore.org/)
- [GitHub Actions Workflow Schema](https://www.schemastore.org/github-workflow.json)
- [JSON Schema 仕様](https://json-schema.org/)
