# Check URL File Update - 配置说明

本目录包含 `check-url-file-update` 工作流的配置文件。

## 配置文件

- **`config.yaml`**: 默认配置文件

## 配置文件结构

```yaml
# URL 列表 - 要监控的文件地址数组
urls:
  - "https://example.com/file1.pdf"
  - "https://example.com/file2.jpg"

# Telegram 消息模板配置 - 所有 URL 共享
# 注意：chatId 通过 GitHub Variable CHECK_URL_FILE_UPDATE (JSON 格式) 配置（必需）
telegram:
  message: |  # 消息模板（支持 Markdown 和变量）
    📄 **文件更新通知**

    **文件信息:**
    - 文件名: `{{FILE_NAME}}`
    - 大小: {{FILE_SIZE_MB}} MB ({{FILE_SIZE_BYTES}} bytes)
    - 下载链接: {{FILE_URL}}

    🕐 检查时间: {{CHECK_TIME}}

# 检测条件 - 用于判断文件是否更新的 HTTP header 字段
# 支持的字段（只能使用以下两个）：
#   - last-modified: 文件最后修改时间
#   - location: 重定向后的最终 URL（用于检测下载地址变化）
conditions:
  - "last-modified"
  - "location"
```

## 支持的消息模板变量

在 `telegram.message` 中可以使用以下变量：

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{FILE_NAME}}` | 文件名（从 URL 提取） | `document.pdf` |
| `{{FILE_SIZE_MB}}` | 文件大小（MB，保留 2 位小数） | `2.45` |
| `{{FILE_SIZE_BYTES}}` | 文件大小（字节） | `2568192` |
| `{{FILE_URL}}` | 文件下载地址 | `https://example.com/file.pdf` |
| `{{CHECK_TIME}}` | 检查时间（UTC） | `2025-12-31 17:30:00` |

## 使用方法

### 1. 编辑配置文件

修改 `config.yaml`，添加要监控的 URL 和 Telegram 配置。

### 2. 配置 GitHub Secrets 和 Variables

在仓库设置中添加（**必需**）：

#### Secrets（敏感信息）
**Settings → Secrets and variables → Actions → Secrets**

**Secret 名称**: `CHECK_URL_FILE_UPDATE`
**Secret 值** (JSON 格式):
```json
{
  "telegram_bot_token": "your-bot-token"
}
```

#### Variables（非敏感配置）
**Settings → Secrets and variables → Actions → Variables**

**Variable 名称**: `CHECK_URL_FILE_UPDATE`
**Variable 值** (JSON 格式):
```json
{
  "telegram_chat_id": "your-chat-id"
}
```

**字段说明**：
- `telegram_bot_token`: Telegram Bot Token（敏感，存放在 Secrets）
- `telegram_chat_id`: Telegram 聊天 ID（非敏感，存放在 Variables）

### 3. 触发工作流

- **定时触发**: 每天 UTC 00:00 自动执行
- **手动触发**: Actions → Check URL File Update → Run workflow
  - 可选参数：
    - **config_file**: 配置文件名（默认：`check-url-file-update/config`）
    - **clear_cache**: 是否清理缓存（默认：`false`）
      - ✅ 勾选：清理所有历史缓存，重新检测所有 URL（即使文件未变化也会下载并发送）
      - ⬜ 不勾选：使用历史缓存，只检测有变化的 URL

### 4. 使用自定义配置文件

如果需要多个配置文件，可以创建：
- `configs/check-url-file-update/production.yaml`
- `configs/check-url-file-update/development.yaml`

手动触发时传入配置文件名（不含扩展名）：
- `check-url-file-update/production`
- `check-url-file-update/development`

## 常用检测条件

| Header 字段 | 说明 | 使用场景 |
|------------|------|---------|
| `last-modified` | 文件最后修改时间 | 检测文件内容是否更新（推荐） |
| `location` | 重定向后的最终 URL | 检测下载地址是否变化 |

**注意**: 只支持以上两个 header 字段。

## 相关文档

- 完整规格说明: `docs/check-url-file-update/specification.md`
- 原始需求: `docs/check-url-file-update/原始需求.md`
- 工作流文件: `.github/workflows/check-url-file-update.yml`
