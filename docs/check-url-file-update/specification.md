# Check URL File Update - 需求规格说明

## 1) 需求概述

### 目标
实现批量监控多个远程 URL 文件更新的自动化流程，当检测到文件更新时，自动下载并通过 Telegram 发送通知（包含文件）。

### 触发方式
- **定时触发**: 可配置的 cron 表达式（例如：每小时或每天检查）
- **手动触发**: `workflow_dispatch` 支持手动执行，可传入以下参数：
  - `config_file`: 配置文件名（不含扩展名），默认 `check-url-file-update/config`
  - `clear_cache`: 是否清理缓存（布尔值），默认 `false`
    - `true`: 清理所有历史缓存，重新检测所有 URL
    - `false`: 使用历史缓存，只检测有变化的 URL

### 输入参数（非敏感）
- **工作流输入**:
  - `config_file`: 配置文件名（不含扩展名），默认为 `check-url-file-update/config`
  - 完整路径为：`configs/<config_file>.yaml`
  - 默认配置文件：`configs/check-url-file-update/config.yaml`

- **配置文件结构** (`configs/<config_file>.yaml`)：
  - `urls`: 数组 - 要监控的 URL 地址列表
    ```yaml
    urls:
      - "https://example.com/file1.pdf"
      - "https://example.com/file2.jpg"
    ```
  - `telegram`: 对象 - Telegram 消息模板配置
    ```yaml
    telegram:
      message: |
        📄 文件更新通知
        文件名: {{FILE_NAME}}
        ...
    ```
    **注意**: chatId 通过 GitHub Variable `CHECK_URL_FILE_UPDATE` JSON 配置（必需）
  - `conditions`: 数组 - 用于检测变化的 HTTP header 字段列表（所有 URL 共享）
    ```yaml
    conditions:
      - "last-modified"
      - "location"
    ```
    **支持的字段**（只能使用以下两个）：
    - `last-modified`: 文件最后修改时间
    - `location`: 重定向后的最终 URL（用于检测下载地址变化）

### 密钥与变量（敏感与非敏感信息）

#### GitHub Secrets (JSON 格式) - 敏感信息
通过 `secrets.CHECK_URL_FILE_UPDATE` 配置：

```json
{
  "telegram_bot_token": "your-bot-token"
}
```

**字段说明**：
- `telegram_bot_token`: Telegram Bot API Token（敏感）

#### GitHub Variables (JSON 格式) - 非敏感配置
通过 `vars.CHECK_URL_FILE_UPDATE` 配置：

```json
{
  "telegram_chat_id": "your-chat-id"
}
```

**字段说明**：
- `telegram_chat_id`: Telegram 聊天 ID（非敏感，可公开）

### 输出结果
- 发送到配置的 Telegram 聊天的文件消息
- 工作流日志，显示哪些 URL 被检查以及哪些有更新

### 失败处理模式
- **网络错误**: 下载失败时重试一次；仍失败则记录错误并跳过
- **无效 URL**: 记录错误并继续处理下一个 URL
- **Telegram 发送失败**: 记录错误但不中断工作流
- **缓存读写失败**: 视为"无历史状态"并继续执行

---

## 2) 实现方案（数据流）

1. **接收配置文件名**: 从工作流输入获取 `config_file` 参数（默认：`check-url-file-update/config`）
2. **解析配置文件**: 加载 `configs/<config_file>.yaml`（默认：`configs/check-url-file-update/config.yaml`），解析得到：
   - `urls`: URL 列表数组
   - `telegram`: Telegram 配置对象
   - `conditions`: 检测条件数组
3. **生成 Matrix**: 将每个 URL 与 telegram、conditions 组合，生成 matrix JSON
   ```json
   {
     "include": [
       {"url": "https://example.com/file1.pdf", "telegram": {...}, "conditions": [...]},
       {"url": "https://example.com/file2.jpg", "telegram": {...}, "conditions": [...]}
     ]
   }
   ```
4. **矩阵并行执行**: 使用 GitHub Actions `matrix` 策略，对每个 URL 执行：
   - 生成缓存 key（基于 URL 的 SHA256 哈希）
   - **如果启用清理缓存** (`clear_cache=true`):
     - 删除历史缓存文件
     - 跳过恢复缓存步骤
   - **如果不清理缓存** (`clear_cache=false`，默认):
     - 从 `actions/cache` 恢复上次的状态（key: URL 哈希值，value: 上次的 HTTP headers）
   - 发送 HEAD 请求到 URL（使用 `curl -I -L` 跟随重定向）获取当前 headers
   - 将 `conditions` 中指定的 header 字段与缓存值对比
   - 更新缓存为当前 headers（无论是否变化）
   - **如果有变化**:
     - 使用 `curl -L -o` 下载文件到临时目录（失败时重试一次）
     - 提取文件信息（文件名、文件大小、检查时间等）
     - 使用模板变量替换 telegram.message 中的占位符
     - 使用 `appleboy/telegram-action` 发送文件和消息到 Telegram
   - **如果无变化**: 跳过下载和通知步骤

---

## 2.1) 实现方式

### 优先级顺序
1. **GitHub Marketplace Actions**（优先使用）:
   - `actions/cache@v4` 用于存储历史 header 状态
   - `appleboy/telegram-action@master` 用于发送 Telegram 文件通知

2. **命令行工具**（核心逻辑）:
   - `yq` 用于解析 YAML 配置文件
   - `curl -I -L` 用于 HTTP HEAD 请求
   - `curl -L -o` 用于文件下载（带重试）
   - `jq` 用于 JSON 处理
   - `bash` 脚本用于：
     - header 对比和流程控制
     - 文件信息提取
     - 消息模板变量替换

3. **最小化自定义脚本**: 仅在命令行工具无法满足时才使用脚本

### 技术选择
- **语言**: Bash 命令行 + 标准 Linux 工具
- **状态存储**: `actions/cache` 基于 URL 的 key
- **Header 检查**: `curl -I -L` 获取 headers
- **文件下载**: `curl -L -o` 下载文件（带重试机制）
- **配置解析**: `yq` 解析 YAML
- **Telegram 发送**: `appleboy/telegram-action@master`
- **敏感信息**: GitHub Actions Secrets (`secrets.TELEGRAM_BOT_TOKEN` 等)
- **矩阵策略**: GitHub Actions 原生 `matrix` 从配置动态生成

---

## 3) 文件布局

### 需要创建/修改的文件
```
.github/workflows/
  check-url-file-update.yml          # 主工作流文件

configs/check-url-file-update/
  config.yaml                        # URL 监控配置文件（默认）
  README.md                          # 配置说明文档
```

**说明**: 本方案主要使用 workflow 内的 bash 命令和现有的 GitHub Actions，无需创建自定义脚本模块。

---

## 4) 配置文件结构

### 文件: `configs/check-url-file-update/config.yaml`

```yaml
# URL 列表 - 要监控的文件地址数组
urls:
  - "https://example.com/file1.pdf"
  - "https://example.com/file2.jpg"
  - "https://example.com/document.docx"

# Telegram 消息模板配置 - 所有 URL 共享
# 注意：chatId 通过 GitHub Variable CHECK_URL_FILE_UPDATE (JSON 格式) 配置（必需）
telegram:
  message: |
    📄 **文件更新通知**

    **文件信息:**
    - 文件名: `{{FILE_NAME}}`
    - 大小: {{FILE_SIZE_MB}} MB
    - 下载链接: {{FILE_URL}}

    检查时间: {{CHECK_TIME}}

# 检测条件 - 用于判断文件是否更新的 HTTP header 字段
# 支持的字段（只能使用以下两个）：
#   - last-modified: 文件最后修改时间
#   - location: 重定向后的最终 URL（用于检测下载地址变化）
conditions:
  - "last-modified"
  - "location"
```

**配置说明**:
- `urls`: 字符串数组，每项为一个要监控的 URL
- `telegram`: 对象，包含 Telegram 消息模板配置
  - `message`: 消息模板（支持 Markdown 格式和变量替换）
    - 支持的变量：
      - `{{FILE_NAME}}`: 文件名（从 URL 提取）
      - `{{FILE_SIZE_MB}}`: 文件大小（MB，保留 2 位小数）
      - `{{FILE_SIZE_BYTES}}`: 文件大小（字节）
      - `{{FILE_URL}}`: 文件下载地址
      - `{{CHECK_TIME}}`: 检查时间（UTC 时间，格式：YYYY-MM-DD HH:MM:SS）
- `conditions`: 字符串数组，要检查的 HTTP header 字段名（小写）
  - **只支持**: `last-modified` 和 `location`

**注意**: Telegram Chat ID 通过 GitHub Variable `CHECK_URL_FILE_UPDATE` (JSON 格式) 配置，不在配置文件中设置

### 密钥与变量配置（GitHub Actions）

#### Secrets (敏感信息)
- `CHECK_URL_FILE_UPDATE`: JSON 对象（必需）
  ```json
  {
    "telegram_bot_token": "your-bot-token"
  }
  ```

#### Variables (非敏感配置)
- `CHECK_URL_FILE_UPDATE`: JSON 对象（必需）
  ```json
  {
    "telegram_chat_id": "your-chat-id"
  }
  ```

**在 workflow 中使用**:
```yaml
env:
  SECRETS_JSON: ${{ secrets.CHECK_URL_FILE_UPDATE }}
  VARS_JSON: ${{ vars.CHECK_URL_FILE_UPDATE }}

# 使用时
token: ${{ fromJson(env.SECRETS_JSON).telegram_bot_token }}
to: ${{ fromJson(env.VARS_JSON).telegram_chat_id }}
```

---

## 5) 详细工作流逻辑

### 工作流步骤

1. **准备作业 (Prepare Job)**:
   - 检出代码仓库
   - 安装 `yq` 和 `jq` 工具
   - 从输入参数获取配置文件名（默认：`check-url-file-update`）
   - 读取 `configs/<config_file>.yaml` 配置文件
   - 解析配置文件，获取 `urls`、`telegram`、`conditions`
   - 生成 matrix JSON：将每个 URL 与 telegram、conditions 组合
   - 输出 matrix JSON 供后续作业使用

2. **检查作业 (Check Job)** - 使用 matrix 策略，依赖 Prepare Job:
   - 对 matrix 中的每个 URL：
     - **生成缓存 Key**:
       - 使用 URL 的 SHA256 哈希生成唯一 key
     - **清理缓存** (如果 `clear_cache=true`):
       - 删除历史缓存文件
       - 后续将视为首次检查
     - **恢复缓存** (如果 `clear_cache=false`):
       - Key: `url-headers-<sha256(url)>`
       - Path: `.cache/<sha256(url)>.headers`
     - **检查更新**:
       - 使用 `curl -I -L` 获取当前 headers
       - 提取配置中指定的 header 字段
       - 与缓存文件中的值对比
       - 输出: `has_update` (true/false)
     - **保存缓存**:
       - 总是保存当前 headers 到缓存（覆盖）
     - **下载文件** (如果 `has_update` 为 true):
       - 使用 `curl -L -o` 下载文件到临时目录
       - 失败时重试一次
       - 提取文件信息（文件名、大小等）
       - 输出: 文件路径
     - **准备消息**:
       - 读取 `matrix.telegram.message` 模板
       - 替换变量占位符（FILE_NAME、FILE_SIZE_MB、FILE_URL、CHECK_TIME 等）
       - 输出: 处理后的消息内容
     - **验证文件**:
       - 检查文件是否存在
       - 输出文件大小信息
     - **发送 Telegram** (如果下载成功):
       - 使用 `appleboy/telegram-action@master`
       - 传入参数：
         - `to`: `${{ fromJson(env.VARS_JSON).telegram_chat_id }}`
         - `token`: `${{ fromJson(env.SECRETS_JSON).telegram_bot_token }}`
         - `message`: 处理后的消息内容（作为文件的 caption）
         - `document`: 下载的文件路径（文件本体）
         - `format`: `markdown`（消息支持 Markdown 格式）
         - `disable_web_page_preview`: `true`

---

## 6) 模块化与复用

### 复用策略
- 使用标准 Linux 命令行工具（curl、jq、yq）
- 利用 GitHub Actions 生态的成熟 action
- Workflow 内的 bash 函数封装可复用逻辑

### 配置驱动
- 所有 URL、检查条件、Telegram 设置均在配置文件中
- 代码中无硬编码的 URL 或聊天 ID
- 增删 URL 无需修改代码

---

## 7) 验收标准

- [ ] 工作流成功解析配置文件
- [ ] Matrix 策略为配置中的每个 URL 创建一个作业
- [ ] 缓存正确存储和恢复历史 header 状态
- [ ] HEAD 请求正确提取 headers（跟随重定向）
- [ ] 对比逻辑根据配置的 conditions 正确识别变化
- [ ] 文件下载成功并在失败时重试一次（使用 `curl -L -o`）
- [ ] 正确提取文件信息（文件名、大小、检查时间）
- [ ] 消息模板变量正确替换（支持所有定义的变量）
- [ ] Telegram 通知使用 `appleboy/telegram-action` 发送文件和消息
- [ ] Telegram 消息支持 Markdown 格式
- [ ] 单个 URL 失败时工作流继续处理其他 URL
- [ ] 所有密钥通过 GitHub Secrets 配置且永不记录到日志
- [ ] 文档包含配置示例和变量使用说明

---

## 8) 实现要点

### 缓存策略
- 使用 URL 的 SHA256 哈希作为缓存 key，避免特殊字符
- Headers 存储为文本格式，便于对比
- 每个 URL 独立缓存，允许独立跟踪

### 错误处理
- 非阻塞式: 一个 URL 失败不影响其他 URL
- 网络瞬态错误的重试逻辑
- 详细日志便于调试

### 性能考虑
- Matrix 并行化实现 URL 并发检查
- HEAD 请求最小化带宽消耗
- 缓存减少不必要的下载

### 命令行实现示例

**生成 Matrix JSON** (Prepare Job):
```bash
CONFIG_FILE="${{ inputs.config_file || 'check-url-file-update/config' }}"
CONFIG_PATH="configs/${CONFIG_FILE}.yaml"

# 读取配置
URLS=$(yq eval '.urls[]' "$CONFIG_PATH" | jq -R . | jq -s .)
TELEGRAM=$(yq eval '.telegram' "$CONFIG_PATH" -o=json)
CONDITIONS=$(yq eval '.conditions' "$CONFIG_PATH" -o=json)

# 生成 matrix
MATRIX=$(jq -n \
  --argjson urls "$URLS" \
  --argjson telegram "$TELEGRAM" \
  --argjson conditions "$CONDITIONS" \
  '{include: [$urls[] | {url: ., telegram: $telegram, conditions: $conditions}]}')

echo "matrix=$MATRIX" >> $GITHUB_OUTPUT
```

**提取并对比 headers** (Check Job):
```bash
# 获取当前 headers（根据 conditions 提取）
for condition in $(echo '${{ toJson(matrix.conditions) }}' | jq -r '.[]'); do
  curl -I -L "${{ matrix.url }}" 2>/dev/null | grep -i "^${condition}:" >> current.headers
done

# 对比
if diff previous.headers current.headers >/dev/null 2>&1; then
  echo "has_update=false" >> $GITHUB_OUTPUT
else
  echo "has_update=true" >> $GITHUB_OUTPUT
fi
```

**下载文件（带重试）并提取信息**:
```bash
URL="${{ matrix.url }}"
FILE_NAME=$(basename "$URL" | cut -d'?' -f1)  # 提取文件名（去除查询参数）
FILE_PATH="/tmp/${FILE_NAME}"

# 下载文件（失败重试一次）
curl -L -o "$FILE_PATH" "$URL" || curl -L -o "$FILE_PATH" "$URL"

# 获取文件大小
FILE_SIZE_BYTES=$(stat -f%z "$FILE_PATH" 2>/dev/null || stat -c%s "$FILE_PATH")
FILE_SIZE_MB=$(echo "scale=2; $FILE_SIZE_BYTES / 1024 / 1024" | bc)

# 获取检查时间
CHECK_TIME=$(date -u "+%Y-%m-%d %H:%M:%S")

# 输出变量
echo "file_path=$FILE_PATH" >> $GITHUB_OUTPUT
echo "file_name=$FILE_NAME" >> $GITHUB_OUTPUT
echo "file_size_bytes=$FILE_SIZE_BYTES" >> $GITHUB_OUTPUT
echo "file_size_mb=$FILE_SIZE_MB" >> $GITHUB_OUTPUT
echo "check_time=$CHECK_TIME" >> $GITHUB_OUTPUT
```

**替换消息模板变量**:
```bash
MESSAGE_TEMPLATE='${{ matrix.telegram.message }}'

# 替换变量
MESSAGE="${MESSAGE_TEMPLATE//\{\{FILE_NAME\}\}/$FILE_NAME}"
MESSAGE="${MESSAGE//\{\{FILE_SIZE_MB\}\}/$FILE_SIZE_MB}"
MESSAGE="${MESSAGE//\{\{FILE_SIZE_BYTES\}\}/$FILE_SIZE_BYTES}"
MESSAGE="${MESSAGE//\{\{FILE_URL\}\}/$URL}"
MESSAGE="${MESSAGE//\{\{CHECK_TIME\}\}/$CHECK_TIME}"

echo "message<<EOF" >> $GITHUB_OUTPUT
echo "$MESSAGE" >> $GITHUB_OUTPUT
echo "EOF" >> $GITHUB_OUTPUT
```

**发送 Telegram 通知**:
```yaml
- name: Send to Telegram
  if: steps.download.outcome == 'success'
  uses: appleboy/telegram-action@master
  with:
    to: ${{ fromJson(env.VARS_JSON).telegram_chat_id }}
    token: ${{ fromJson(env.SECRETS_JSON).telegram_bot_token }}
    message: ${{ steps.prepare-message.outputs.message }}  # Caption（文件说明）
    document: ${{ steps.download.outputs.file_path }}      # 文件本体
    format: markdown                                       # 支持 Markdown 格式
    disable_web_page_preview: true
```

**说明**：
- `document`: 发送的文件路径（文件本体）
- `message`: 文件的 caption（说明文字），支持 Markdown 格式
- `format: markdown`: 使 caption 支持 Markdown 富文本格式（**粗体**、`代码`、链接等）

---

## 9) 后续增强（可选）

初始实现范围外的潜在改进:
- 支持认证（headers、basic auth）
- 自定义下载文件名模式
- 下载前的文件大小限制
- 多通知渠道（Discord、Slack）
- 基于校验和的验证（不只是 header 对比）
