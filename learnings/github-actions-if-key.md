# GitHub Actions の `if` キーの使い方

## `if` を配置できる場所（スコープ）

`if`キーは2つのレベルで使用できる：

### 1. ジョブレベル
```yaml
jobs:
  build:
    if: github.ref == 'refs/heads/main'  # ← runs-onと同じインデント
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."
```

**影響範囲**: そのジョブ全体がスキップされる（全ステップが実行されない）

### 2. ステップレベル
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        if: github.event_name == 'push'  # ← name, runと同じインデント
        run: echo "Testing..."
```

**影響範囲**: そのステップのみがスキップされる（他のステップは影響を受けない）

## インデントの書き方

### 重要なポイント
- `if`は他のプロパティと**同じインデントレベル**に書く
- 「一つ下の行」ではない
- プロパティの順序は自由（`if`を先に書いても後に書いてもOK）

```yaml
# どちらもOK
- name: Step 1
  if: success()
  run: echo "hello"

- if: success()
  name: Step 2
  run: echo "hello"
```

## `if` が影響を受けるコンテキスト（条件判定に使える情報源）

| コンテキスト | 内容 | 使用例 |
|------------|------|--------|
| `github` | イベント、ブランチ、リポジトリ情報 | `if: github.ref == 'refs/heads/main'` |
| `needs` | 依存ジョブの結果・出力 | `if: needs.build.result == 'success'` |
| `steps` | 前のステップの出力・状態 | `if: steps.test.outcome == 'failure'` |
| `env` | 環境変数 | `if: env.DEPLOY_ENV == 'production'` |
| `inputs` | workflow_dispatchの入力 | `if: inputs.skip_tests != true` |

## 実例

```yaml
name: If Example

on: push

jobs:
  build:
    if: github.ref == 'refs/heads/main'  # mainブランチでのみ実行
    runs-on: ubuntu-latest
    steps:
      - name: Build step
        run: echo "Building..."

  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        if: github.event_name == 'push'  # pushイベントでのみ実行
        run: echo "Testing..."

      - name: Always run
        run: echo "This always runs"

  deploy:
    needs: [build, test]
    if: needs.build.result == 'success'  # buildが成功した場合のみ実行
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

## まとめ

- **ジョブレベル**: ジョブ全体を制御（`runs-on`と同じインデント）
- **ステップレベル**: 個別ステップを制御（`name`, `run`と同じインデント）
- **条件式**: 様々なコンテキスト（`github`, `needs`, `steps`, `env`, `inputs`）を使用可能
- **インデント**: 他のプロパティと同じレベルに配置
