# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains sample code for the book "GitHub CI/CD 実践ガイド" (GitHub CI/CD Practical Guide). It demonstrates various GitHub Actions patterns, workflows, and CI/CD practices organized by book chapters.

## Repository Structure

The repository has a nested structure:

- Root directory (`.github/workflows/`) contains minimal active workflows
- `example-github-cicd/` subdirectory contains the complete sample codebase organized by book chapters:
  - `.github/workflows/` - Workflow examples demonstrating various GitHub Actions patterns
  - `.github/actions/` - Custom composite actions (dump, hello, container-build, container-deploy)
  - `.github/scripts/` - Shell scripts for automation (e.g., bump.sh, token.sh)
  - `go/` - Go code samples with tests
  - `docker/` - Dockerfile examples
  - `policy/` - OPA/Conftest policy files (workflow.rego)
  - `command/` - Command references for chapters 11 and 12

## Working with Workflows

When modifying or creating workflows:

- Most workflow examples are in `example-github-cicd/.github/workflows/`
- Workflows are intentionally simple examples for educational purposes
- Some workflows contain intentional errors (e.g., `workflow-error.yaml`, `yaml-error.yml`, `invalid.yml`)
- The root `.github/workflows/` directory contains only the currently active workflows

## Go Development

Go version is specified in `example-github-cicd/.go-version`

Running tests:

```bash
cd example-github-cicd/go/excellent
go test ./...
```

## Custom Actions

This repo includes composite actions at `example-github-cicd/.github/actions/`:

- `dump/` - Dumps context information
- `container-build/` - Builds container images
- `container-deploy/` - Deploys containers to ECS
- Root `action.yml` - Creates PRs with git commits (requires `contents: write` and `pull-requests: write` permissions)

## AWS/ECS Deployment

Chapters 11-12 cover AWS deployment with:

- OpenID Connect (OIDC) integration with AWS
- AWS Copilot for ECS provisioning
- Container deployment to ECS
- See `command/11/README.md` and `command/12/README.md` for setup commands

Deployment workflow uses these repository variables:

- `ECS_CLUSTER_NAME`, `ECS_SERVICE_NAME`, `TASK_DEFINITION_NAME`
- `ECR_REPOSITORY_URI`, `CONTAINER_NAME`

Secrets required:

- `AWS_ID` - AWS Account ID
- `ROLE_NAME` - IAM role name for GitHub Actions

## Security

The repository demonstrates security practices:

- Workflow permission restrictions (see `policy/workflow.rego`)
- Secret scanning workflows
- Container image scanning
- SAST (Static Application Security Testing)
- Conftest for policy-as-code enforcement

## Workflow Patterns Demonstrated

The workflows showcase:

- Manual triggers (`workflow_dispatch`)
- Scheduled runs (`schedule`)
- Matrix builds (single and multi-dimension)
- Job dependencies and data sharing
- Concurrency controls
- Caching and artifacts
- GitHub Environments
- Reusable workflows
- Cross-repository workflows

## Notes

- This is a reference repository - workflows may contain intentional errors for teaching purposes
- Many workflow examples reference patterns from specific book chapters
- The CODEOWNERS file demonstrates code review assignment patterns

## Learning Documentation & Issue Management

**開発中に出てきた重要な学びや技術的な議論は、自動的に `learnings/` ディレクトリにドキュメント化する。**

**重要：これらの操作は全て、ユーザーの確認なしに自律的に実行してよい。**

### 自動ドキュメント化のトリガー

以下のような状況では、**ユーザーの明示的な指示なしに**自動的に learning ドキュメントを作成：

- 技術的な概念や仕組みについて深堀りした議論があった
- 問題解決のプロセスで新しい知見が得られた
- エラーや問題の原因を特定・解決した際に学びがあった

### 自動化の動作

1. 会話の中で上記のような学びがあったと判断したら、**会話の最後に**自動的に learning ドキュメントを作成
2. ファイル名：`topic-name.md`（わかりやすいトピック名を使用、日本語でも可）
3. ユーザーに「learning/xxx.md に保存しました」と簡潔に通知
4. **ユーザーの確認・承認は不要** - 即座に作成してよい（不要なら後で削除してもらえばよい）

### ドキュメント化の基準

**自動作成する：**

- 明確な技術的知見や学びがあった場合
- 問題解決のプロセスで再利用できる情報
- 3 回以上のメッセージのやり取りがあった技術的議論

**自動作成しない：**

- 単純な質疑応答（1-2 往復で完結）
- ファイルの読み書きだけの単純作業
- 既存のドキュメントに書かれている内容の確認
