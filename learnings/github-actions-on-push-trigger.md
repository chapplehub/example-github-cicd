# GitHub Actions の `on: push` トリガーの動作

## 誤解しやすいポイント

`on: push` は**ワークフローファイル自体がpushされたときだけ**トリガーされるわけではない。

## 実際の動作

```yaml
on: push
```

この設定は、以下のすべての場合にワークフローをトリガーする：

- リポジトリの**任意のブランチ**に対するpush
- **任意のファイル**の変更（ワークフローファイル、ソースコード、README、設定ファイルなど）
- 例：`go/excellent/main.go` を修正してpushしても実行される

つまり、**リポジトリ内のどのファイルをpushしても、このワークフローは実行される**。

## トリガーを限定する方法

### 特定のファイル・ディレクトリに限定

ワークフローファイル自体や特定のファイルが変更されたときだけ実行したい場合：

```yaml
on:
  push:
    paths:
      - '.github/workflows/**'  # ワークフローファイルのみ
      - 'src/**'                # srcディレクトリ配下のみ
      - '*.go'                  # Goファイルのみ
```

### 特定のファイルを除外

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'               # docsディレクトリの変更は無視
      - '**.md'                 # Markdownファイルの変更は無視
```

### 特定のブランチに限定

```yaml
on:
  push:
    branches:
      - main                    # mainブランチのみ
      - 'release/**'            # release/で始まるブランチ
      - '!experimental'         # experimentalブランチは除外
```

### 組み合わせ

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'tests/**'
```

これは「mainブランチの src/ または tests/ ディレクトリに変更があったとき」にのみ実行される。

## このリポジトリでの例

`.github/workflows/hello.yml`、`yaml-error.yml`、`workflow-error.yaml` はすべて単純な `on: push` を使用している。

これは教育目的のシンプルな例として提供されており、実際のプロダクションでは適切なフィルターを使用することが推奨される。

## 参考

- [GitHub Actions: Workflow syntax for GitHub Actions - on.push](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore)
