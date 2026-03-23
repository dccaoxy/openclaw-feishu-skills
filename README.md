# OpenClaw Feishu Skills

飞书 (Feishu/Lark) 技能集合，用于 OpenClaw 智能助手。

## 技能列表

| 技能名 | 说明 | 来源 |
|--------|------|------|
| feishu-bitable | 飞书多维表格操作，自动创建和填充 Bitable 表格 | LobeHub |
| feishu-doc-manager | 飞书文档 Markdown 发布 | LobeHub |
| feishu-sheets-skill | 飞书表格操作 | LobeHub |
| feishu-send-file | 飞书文件发送（支持 HTML、ZIP、PDF 等普通文件） | LobeHub |

## 安装方法

### 使用 LobeHub CLI

```bash
# 注册（首次使用）
npx -y @lobehub/market-cli register \
  --name "YourName" \
  --description "Your description" \
  --source open-claw

# 安装技能
npx -y @lobehub/market-cli skills install openclaw-skills-feishu-bitable-creator --agent open-claw
npx -y @lobehub/market-cli skills install openclaw-skills-feishu-doc-manager --agent open-claw
npx -y @lobehub/market-cli skills install openclaw-skills-feishu-sheets-skill --agent open-claw
npx -y @lobehub/market-cli skills install openclaw-skills-feishu-send-file --agent open-claw
```

### 手动安装

将对应技能文件夹复制到 `~/.openclaw/skills/` 目录下。

## 前置要求

- OpenClaw 已配置飞书频道（在 `~/.openclaw/openclaw.json` 中配置 `appId` 和 `appSecret`）
- 飞书应用已授予相应权限

## 许可证

MIT
