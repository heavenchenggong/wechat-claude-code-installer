---
name: wechat-claude-code-installer
description: 一键装好「微信 ↔ Claude Code 桥」并避开 6 个新手大坑。包装 Wechat-ggGitHub/wechat-claude-code 的安装流程，自动修复 cwd 僵尸字段、关闭 hook 元话术泄露、加 caffeinate 防闲置睡眠、检测代理环境变量合规风险、提供 launchd 自启关闭命令。适用于希望在微信里用本机 Claude Code 但被原项目体验问题挡住的用户。
metadata:
  type: skill
  scope: global
---

# WeChat ↔ Claude Code 一键安装 + 6 坑全过

把上游项目 [`Wechat-ggGitHub/wechat-claude-code`](https://github.com/Wechat-ggGitHub/wechat-claude-code) 的安装流程包装成一个可重复执行的 skill，**自动修掉**首次使用一定会遇到的几个体验问题。

> 灵感来自《在你微信里用爽 Claude Code，我的开源 Skill 又更新了》一文（公众号「逛逛GitHub」，2026-06）。
> 上游项目本身很优秀（走腾讯 iLink AI 官方通道，没有封号风险），但首次使用有几个非显然的坑会让你觉得"装好了但不对劲"。这个 skill 就是把那些坑一次性提前消掉。

## 为什么要这个 skill

裸跑上游项目首次使用会遇到：

| # | 坑 | 现象 | 修法 |
|---|---|---|---|
| 1 | cwd 僵尸字段 | setup 让你输工作目录，但下次发消息 cwd 还是 `~/Documents/ClaudeCode` | session.json 真值 + `/cwd` 命令 |
| 2 | Hook 元话术泄露 | Claude 回复每条都带"评估：xxx 无需提取技能" | `/prompt` 覆盖系统提示 |
| 3 | 闲置睡眠中断 | 离开电脑几分钟 daemon 假死 | launchd plist 加 `caffeinate -i` 包裹 |
| 4 | 代理环境变量继承 | `ANTHROPIC_BASE_URL` 指向某个本地代理时，daemon 强依赖该代理 | 启动前检测 + 提示 |
| 5 | launchd 不死鸟 | `kill -9` 也杀不掉 daemon | 给出正确的 unload 命令 |
| 6 | 二维码扫了不接续 | setup 卡在"工作目录"那步等输入 | 一行命令传入工作目录 |

这些坑文档里没写，需要自己读源码 + 看 launchd plist + 看 session.json 才能搞明白。

## 触发方式

用户说以下任意一种时启动：

- 「装一下微信桥」「微信里用 Claude Code」「wechat-claude-code 安装」
- 「按那篇逛逛GitHub 的文章配一下」
- `/wechat-claude-code-installer`
- 「我前两天看到那个微信 ↔ AI 桥，帮我装」

## 前置条件

- macOS（launchd 守护；Linux 上游也支持 systemd / nohup，但本 skill 只测试过 macOS）
- Node.js >= 18
- 微信号一个（**强烈建议用个人小号或测试号**，不要用工作号）
- 已经在用 Claude Code（本 skill 不负责装 Claude Code 本身）

## 安装流程（4 步）

### 第 0 步：装前体检

```bash
echo "=== Node 版本（要 ≥ 18）==="
node --version
echo "=== 是否已装过 ==="
ls ~/.claude/skills/wechat-claude-code 2>/dev/null && echo "已存在" || echo "未装"
echo "=== 是否绑过账号 ==="
ls ~/.wechat-claude-code/accounts/ 2>/dev/null | head -1 && echo "已绑" || echo "未绑"
echo "=== 是否有 ANTHROPIC_BASE_URL 代理（合规提醒）==="
echo "ANTHROPIC_BASE_URL=$ANTHROPIC_BASE_URL"
grep -h "ANTHROPIC_BASE_URL\|ANTHROPIC_AUTH_TOKEN" ~/.claude/settings.json 2>/dev/null | head -2
```

**重要提示**：如果 `ANTHROPIC_BASE_URL` 指向某个**非 `https://api.anthropic.com` 的地址**（比如 `http://localhost:某端口/...`），意味着你所有 Claude 调用都经过这个代理。微信桥 daemon 也会继承同一路径，**所有微信对话也会经过代理日志**。

如果这个代理是公司部署的内部 LLM 网关：
- ✅ 优点：用公司额度
- ⚠️ 缺点：所有微信对话内容都进公司日志（即使你以为是"私聊"）
- 建议：聊私事请改成走个人 Anthropic 账号或第三方代理（见"切换 API 端点"章节）

### 第 1 步：clone + 装依赖

```bash
GH_HOST=github.com git clone https://github.com/Wechat-ggGitHub/wechat-claude-code.git \
  ~/.claude/skills/wechat-claude-code
cd ~/.claude/skills/wechat-claude-code && npm install
```

占用约 29 MB（node_modules 28 MB + 源码 < 1 MB）。

### 第 2 步：扫码绑定 + 设工作目录（一气呵成）

⚠️ 上游 setup 命令是**交互式**的，扫码后还会卡在"请输入工作目录"等输入。直接灌答案进去：

```bash
WORK_DIR="$HOME/Claude Code"   # ← 改成你想要的工作目录
mkdir -p "$WORK_DIR"

cd ~/.claude/skills/wechat-claude-code && \
  echo "$WORK_DIR" | npm run setup
```

setup 会：
1. 弹出二维码图片（macOS 自动用预览打开）
2. 等你用微信「扫一扫」+ 确认
3. 微信里多出一个 AI bot 好友
4. 把 `$WORK_DIR` 写进 `~/.wechat-claude-code/config.json`（**注意：上游 1.0.0 这个字段实际上不会被 daemon 读，下面会修**）

### 第 3 步：把 session.json 的 cwd 顶上去（修第 1 个坑）

上游 1.0.0 的 daemon 用的是 `~/.wechat-claude-code/sessions/<accountId>.json` 里的 `workingDirectory`，**不读 config.json**。第一次发消息会触发创建这个文件，里面写死 `~/Documents/ClaudeCode`。

正确修法（在用户**首次发完一条消息后**做）：

```bash
WORK_DIR="$HOME/Claude Code"   # 与第 2 步一致

# 等首次消息触发 session 文件生成（如果还没发，手动建）
ACCOUNT=$(ls ~/.wechat-claude-code/accounts/*.json | head -1 | xargs basename | sed 's/.json//')
SESSION_FILE=~/.wechat-claude-code/sessions/$ACCOUNT.json
mkdir -p ~/.wechat-claude-code/sessions

# 直接把 cwd + 系统提示词都设上
python3 <<PY
import json, os
sf = "$SESSION_FILE"
data = {}
if os.path.exists(sf):
    with open(sf) as f: data = json.load(f)
data['workingDirectory'] = "$WORK_DIR"
# 关掉 hook 元话术泄露（修第 2 个坑）
data['systemPrompt'] = '直接简洁回答用户的问题。不要在末尾追加任何"评估""技能提取"或类似元话术。'
data.setdefault('state', 'idle')
data.setdefault('chatHistory', [])
data.setdefault('maxHistoryLength', 100)
with open(sf, 'w') as f: json.dump(data, f, ensure_ascii=False, indent=2)
print("✅ session 已配置：cwd =", data['workingDirectory'])
PY
```

> **替代方案**：如果你已经在微信里跟 bot 聊起来了，直接发：
> ```
> /cwd /Users/你的用户名/你想要的目录
> /prompt 直接简洁回答用户的问题。不要追加元话术。
> ```
> 上游内置斜杠命令也能改这两个，**不需要改文件**。

### 第 4 步：启 daemon + 加 caffeinate 防睡（修第 3 个坑）

```bash
cd ~/.claude/skills/wechat-claude-code && npm run daemon -- start
sleep 2
npm run daemon -- status
```

加 caffeinate 防闲置睡眠（**强烈建议**——不加的话离开电脑 5 分钟 daemon 假死）：

```bash
PLIST=~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
cp "$PLIST" "${PLIST}.bak.$(date +%Y%m%d-%H%M%S)"

python3 <<'PY'
import plistlib, os
p = os.path.expanduser("~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist")
with open(p, 'rb') as f: d = plistlib.load(f)
# 已加过就不重复
if d['ProgramArguments'][0] == '/usr/bin/caffeinate':
    print("已经包过 caffeinate，跳过")
else:
    d['ProgramArguments'] = ['/usr/bin/caffeinate', '-i', '--', *d['ProgramArguments']]
    with open(p, 'wb') as f: plistlib.dump(d, f)
    print("✅ 已加 caffeinate -i 包裹")
PY

# 重新加载
launchctl unload ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
launchctl load ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
```

`caffeinate -i` 含义：阻止系统**闲置睡眠**（不动键鼠也不睡）。
- ✅ 闲置不会睡 → daemon 一直活
- ⚠️ **手动合盖还是会睡**（`-i` 不管 clamshell）。要合盖也不睡换成 `-id`（注意发热）
- ⚠️ 拔电源（用电池）默认还是会睡（macOS 电池策略）

### 第 5 步：验证

在微信里给 AI bot 发：

```
当前工作目录是哪里
```

正确响应应该：
- 路径是你设的 `WORK_DIR`（不是 `~/Documents/ClaudeCode`）
- 末尾**不带**"评估：xxx 无需提取技能"那段元话术

通过 → 装好了。

## 日常使用

| 微信里发 | 作用 |
|---|---|
| `/help` | 内置帮助 |
| `/cwd <绝对路径>` | 切换工作目录 |
| `/clear` | 清当前会话开新对话 |
| `/status` | 查看会话状态 |
| `/model <名称>` | 切 Claude 模型 |
| `/prompt <内容>` | 设/改系统提示词 |
| `/skills` | 列已装的 skill |
| 任意自然语言 | 转给本机 Claude Code 处理 |

发图、发文件、发语音都可以；Claude 生成的文件也会**直接推回微信**。

## 关掉 / 重启 daemon

```bash
# 临时关（这次开机不再起）
cd ~/.claude/skills/wechat-claude-code && npm run daemon -- stop

# 永久关（删除 launchd 注册，下次开机也不起；修第 5 个坑）
launchctl unload -w ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist

# 重新启用
launchctl load -w ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
```

`-w` 标志是关键：写入 disabled 状态，**重启后也不会自启**。不加 `-w` 的话，`KeepAlive` + `RunAtLoad` 会让它"杀不死"。

## 切换 API 端点（避开公司代理 / 改用其他模型）

如果你前面体检发现 `ANTHROPIC_BASE_URL` 指向公司内部代理，且不希望微信对话经过公司日志：

```bash
PLIST=~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist

python3 <<'PY'
import plistlib, os
p = os.path.expanduser("~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist")
with open(p, 'rb') as f: d = plistlib.load(f)
env = d.setdefault('EnvironmentVariables', {})
# 改成你想用的（这里以官方为例，换成第三方代理同理）
env['ANTHROPIC_AUTH_TOKEN'] = 'sk-ant-api03-你的个人key'
env['ANTHROPIC_BASE_URL'] = 'https://api.anthropic.com/v1/'
with open(p, 'wb') as f: plistlib.dump(d, f)
print("✅ 已切到个人账号")
PY

launchctl unload ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
launchctl load ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
```

## 卸载

```bash
launchctl unload -w ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
rm ~/Library/LaunchAgents/com.wechat-claude-code.bridge.plist
rm -rf ~/.claude/skills/wechat-claude-code
rm -rf ~/.wechat-claude-code

# 本 skill 自身
rm -rf ~/.claude/skills/wechat-claude-code-installer
```

完全干净。

## 风险与免责

1. **微信账号风险**：上游走腾讯 iLink AI 官方通道（`ilinkai.weixin.qq.com`），不是 wechaty/itchat 类客户端模拟，封号概率极低；但建议首次用**个人小号或测试号**，不要用工作号 / 公众号绑定的微信。
2. **数据流向**：所有微信消息 → 本机 daemon → 你设置的 `ANTHROPIC_BASE_URL` → Claude。如果 BASE_URL 指向公司代理，对话内容会进入公司日志。本 skill 在第 0 步会提醒检查。
3. **本机权限**：daemon 用本机的 Claude Code SDK，等于把"微信里发消息的人"等同于"坐在你电脑前的人"——能跑命令、读文件、改文件。**不要把绑定的微信号借给别人**。
4. **本 skill 不修改上游代码**，所有改动都在 launchd plist + session.json 文件里，可随时回滚。
5. 上游项目仍在迭代中（看作者还在加 caffeinate / 接续会话等功能）；本 skill 的修法基于上游 1.0.0，未来版本可能不需要。

## 参考

- 上游项目：[Wechat-ggGitHub/wechat-claude-code](https://github.com/Wechat-ggGitHub/wechat-claude-code)
- 灵感原文：「在你微信里用爽 Claude Code，我的开源 Skill 又更新了」（公众号「逛逛GitHub」，2026-06）
- 腾讯 iLink AI（官方机器人入驻通道）：`ilinkai.weixin.qq.com`

## License

MIT。本 skill 是上游项目的**安装与配置助手**，不重新分发上游代码。
