# etzhayyim-project-projection-operator App migration

projection-manager を projection-operator 側へ統合した構成です。
このディレクトリのみを運用対象とし、MCP エンドポイントは `po.etzhayyim.com` を正とします。

## 統合済み components

- `projection-operator-mcp-component` (`projection-operator-po1x9k2m`)
  - project/thread/message/run のオペレーション
- `projection-manager-mcp-component` (`projection-manager-pm7k3x9n`)
  - quota/contract/activity/auction/blocker 管理

## mailbox actor integration

- project 単位で `[project-nanoid]@etzhayyim.com` を Resend 受信
- inbound mail は activity/message として取り込み
- project manager actor が返信要否を判定
