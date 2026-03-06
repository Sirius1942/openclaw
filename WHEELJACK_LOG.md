# 千斤顶工作日志

## 🔧 千斤顶 Issue 分析 2026-03-06 23:05

### 📊 本次扫描概况
- 扫描 Issues: 20 个
- 新增严重 Issues: 10+ 个
- 可贡献 Issues: 6 个（3 个高优先级）

---

### 🚨 高优先级 Issues

#### #37955: session-memory hook: idle/daily session rollovers don't fire command:new/reset event ⭐⭐⭐⭐
- **类型**: Bug（功能缺失）
- **难度**: 中等
- **问题描述**:
  - `session-memory` hook 监听 `command:new` 和 `command:reset` 事件
  - 用户显式输入 `/new` 或 `/reset` 时正常工作
  - 但 idle timeout 或 daily reset 触发的会话重置**不会触发 hook 事件**
  - 导致自动重置的会话无法保存上下文到 `memory/YYYY-MM-DD.md`
- **根本原因**:
  - `dist/reply-DhtejUNZ.js:80584`: 只在 `resetRequested && params.command.isAuthorizedSender` 时触发 hook
  - idle/daily freshness rollover 路径（line ~85857）不调用 `emitResetCommandHooks`
- **技术方案**:
  在 `initSessionState` 中，当 idle/daily 触发 rollover 时，emit `command:reset` 事件，添加 `commandSource: "auto:idle-reset"` 或 `"auto:daily-reset"`
- **涉及文件**: `src/session/init.ts`, `src/hooks/lifecycle.ts`, `src/session/freshness.ts`
- **预计工作量**: 4-6 小时

#### #37943: Node Service fails to parse valid config on macOS ⭐⭐⭐⭐
- **类型**: Bug（严重）
- **难度**: 困难
- **问题描述**:
  - `ai.openclaw.node` launchd 服务持续报错 "Config invalid"
  - 即使 `openclaw.json` 已被 `jq` 验证为有效 JSON
  - Gateway 成功加载同一配置文件
  - 尝试过的修复均无效：重启、删除 state、launchd reload、文件重写
- **技术方案**:
  1. 对比 Gateway 和 Node 服务的 config 加载代码
  2. 添加详细的 debug 日志，定位具体哪个字段导致 "Config invalid"
  3. 修复 Node 服务的 config schema validation
- **涉及文件**: `src/node/config.ts`, `src/node/doctor.ts`, `src/config/loader.ts`
- **预计工作量**: 6-10 小时

#### #37939: Telegram channel: stale-socket false positive every 30 minutes ⭐⭐⭐⭐
- **类型**: Bug（误报）
- **难度**: 中等
- **问题描述**:
  - Health monitor 每 30 分钟检测到 "stale socket" 并重启 Telegram 连接
  - Telegram 使用 long polling，不是 WebSocket，30 分钟无消息是正常的
  - 导致消息丢失、cron job 投递失败
- **技术方案**:
  为 polling-based channels 禁用 stale-socket 检测。在 channel metadata 中添加 `transportType: "polling" | "websocket"`，Health monitor 只对 WebSocket channels 应用检测
- **涉及文件**: `src/gateway/health-monitor.ts`, `src/channels/telegram/client.ts`, `src/channels/base.ts`
- **预计工作量**: 4-6 小时

---

### 🎯 中优先级 Issues

#### #37915: nextcloud-talk plugin fails to load: Cannot find module abort-signal.js ⭐⭐⭐
- **类型**: Bug（打包问题）
- **难度**: 简单
- **问题**: `abort-signal.js` 未被正确打包到 dist
- **方案**: 修复 build 配置或修改 import 路径
- **预计工作量**: 2-4 小时

#### #37913: Chrome extension relay: tab not found when using remote gateway + macOS node proxy ⭐⭐⭐
- **类型**: Bug（复杂）
- **难度**: 困难
- **问题**: VPS gateway 通过 `browser.proxy` 无法看到 Mac node 的 Chrome tabs
- **方案**: `browser.proxy` 应使用 node 的本地 relay，添加 debug mode
- **预计工作量**: 8-12 小时

#### #37909: message tool cannot resolve SecretRef for channels.telegram.botToken ⭐⭐⭐
- **类型**: Bug（Secrets 管理）
- **难度**: 中等
- **问题**: `message` tool 在运行时无法解析 SecretRef
- **方案**: 从 gateway runtime snapshot 读取已解析的 token，而非重新解析
- **预计工作量**: 4-6 小时

---

### 📋 下一步行动

#### 推荐优先级
1. **#37955** (session-memory hook 不触发) - 影响核心功能，方案清晰
2. **#37939** (Telegram stale-socket 误报) - 影响稳定性，方案明确
3. **#37943** (Node 服务 config 解析失败) - 严重 bug，但需要深入调试

#### 行动计划
选择 **#37955** 作为首个贡献目标：
- 影响范围明确
- 技术方案清晰
- 测试相对简单
- 对用户价值高

**开发步骤**:
1. Fork 检查: `cd ~/Projects/openclaw-dev && git fetch upstream`
2. 创建分支: `git checkout -b fix/session-memory-idle-reset-hook`
3. 实现: 在 idle/daily rollover 路径添加 hook emit
4. 测试: 配置短 idle timeout 验证
5. 提交 PR 到 `openclaw/openclaw`

---

### 💡 技术洞察

1. **Lifecycle Events 设计**: #37955 暴露了 lifecycle events 的不完整性，所有会话边界都应有统一的 event 机制
2. **Health Monitor 适配性**: #37939 说明需要根据 channel transport type 调整策略
3. **Config 解析一致性**: #37943 反映 Gateway 和 Node 服务应共享同一套 config loader
4. **Secrets 解析时机**: #37909 揭示应在启动时解析并缓存，而非每次使用时重新解析
5. **Browser Proxy 架构**: #37913 涉及复杂网络拓扑，需要清晰的 tab registry 同步机制

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*

## 🔧 千斤顶 Issue 分析 2026-03-07 00:04

### 📊 本次扫描概况
- 扫描时间: 2026-03-07 00:04 (周六凌晨)
- 扫描 Issues: 20 个
- 新增 Issues: 18 个（过去 8 小时内）
- 可贡献 Issues: 5 个

---

### 🎯 可贡献 Issues

#### #38083: Slack channel auth failures cause retry storm without backoff ⭐⭐⭐⭐
- **类型**: Bug（稳定性）
- **难度**: 简单
- **问题描述**: Slack channel 认证失败时无退避机制，导致重试风暴，影响其他 channels 性能
- **技术方案**: 添加指数退避（1s → 2s → 4s → 最大 60s）
- **涉及文件**: `src/channels/slack/client.ts`
- **预计工作量**: 2-3 小时

#### #38076: skill-creator: make init_skill --resources parsing case-insensitive ⭐⭐⭐
- **类型**: Enhancement（易用性）
- **难度**: 简单
- **问题描述**: `--resources` 参数大小写敏感
- **技术方案**: 解析时转小写
- **涉及文件**: `extensions/skill-creator/init_skill.js`
- **预计工作量**: 1-2 小时

#### #38067: Cron isolated sessions lose sandbox.docker.dangerouslyAllow* flags ⭐⭐⭐⭐
- **类型**: Bug（配置丢失）
- **难度**: 中等
- **状态**: 已有 PR #38081 提交修复
- **备注**: 可以 review PR 代码质量


#### #38052: `openclaw gateway install` fails on Linux when service not yet installed ⭐⭐⭐⭐
- **类型**: Bug（安装失败）
- **难度**: 简单
- **状态**: 已有 PR #37345 修复相同根因
- **问题描述**: 首次安装时 `systemctl is-enabled` 报错，因为服务尚不存在
- **技术方案**: 捕获 "not found" 错误，视为正常
- **涉及文件**: `src/gateway/service.ts`
- **预计工作量**: 1-2 小时

#### #38022: Secret Provider Configuration Has "Chicken and Egg" Validation Problem ⭐⭐⭐
- **类型**: Bug（配置验证）
- **难度**: 中等
- **标签**: bug, bug:behavior
- **问题描述**: Secret provider 配置验证存在循环依赖问题
- **预计工作量**: 4-6 小时（需进一步调查）

---

### 🚀 推荐贡献目标

#### 首选: #38083 (Slack retry storm)
**理由**:
- 影响生产稳定性
- 技术方案清晰（指数退避是标准模式）
- 代码改动小，风险低
- 容易测试

**开发步骤**:
1. `git checkout main && git pull upstream main`
2. `git checkout -b fix/slack-auth-backoff`
3. 修改 `src/channels/slack/client.ts`，添加退避逻辑
4. 测试: 使用无效 token 触发认证失败，验证退避行为
5. 提交 PR 到 `openclaw/openclaw`

#### 次选: #38076 (skill-creator case-insensitive)
**理由**: 改进用户体验，极简单（1 行代码），零风险

---

### 📋 下次扫描重点

1. 关注 #38067 的 PR #38081 合并状态
2. 跟踪 #38052 与 PR #37345 的关系
3. 监控新增的安全相关 Issue (#38074)
4. 观察 Telegram/WhatsApp 消息丢失问题 (#38058, #38055)

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*  
*下次扫描: 2026-03-07 01:04*

## 🔧 千斤顶 Issue 分析 2026-03-07 01:04

### 📊 本次扫描概况
- 扫描 Issues: 20 个
- 新增严重 Issues: 多个高影响 bug
- 可贡献 Issues: 4 个高价值目标

---

### 🚨 高优先级可贡献 Issues

#### #38179: Discord channel exec fails with gateway token mismatch ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression
- **难度**: 中等
- **问题**: Discord exec 失败，TUI 正常。已有 Telegram 修复参考（PR #37233）
- **方案**: 参考 Telegram 修复，为 Discord 应用相同的 gateway token 传递逻辑
- **工作量**: 3-4 小时

#### #38168: Venice.AI 默认配置导致 400 错误 ⭐⭐⭐⭐⭐
- **类型**: Bug（严重 UX 问题）
- **难度**: 简单
- **问题**: onboard 设置 maxTokens: 8192，但多数 Venice 模型只支持 4096
- **方案**: 修改 onboard 为 Venice.ai 设置 maxTokens: 4096
- **工作量**: 2-3 小时
- **优先级**: 影响所有新用户，Venice 团队正收到投诉

#### #38145: maxConcurrent 对 Telegram 无效（代码分割问题）⭐⭐⭐⭐
- **类型**: Bug（架构）
- **难度**: 困难
- **问题**: Rolldown 将 queue 代码复制到两个 chunk，导致两个独立的 lanes Map
- **方案**: 将 queue 模块移到共享 chunk
- **工作量**: 6-8 小时

#### #38126: Matrix 无法连接私有服务器 ⭐⭐⭐
- **类型**: Bug + Regression
- **难度**: 简单
- **问题**: 安全策略阻止私有 IP
- **方案**: 添加 channel 级别的 allowPrivateIPs 配置
- **工作量**: 2-3 小时

---

### 🎯 推荐贡献目标

**首选: #38168 (Venice.AI 配置)**
- 用户影响最大（所有新用户开箱即损坏）
- 修复最简单（改数字 + 条件判断）
- Venice 团队正在等待修复

**次选: #38179 (Discord exec)**
- Regression（高优先级）
- 有参考实现（Telegram PR #37233）

---

### 💡 技术洞察
1. Provider onboarding 缺乏模型能力验证
2. Rolldown 配置可能导致状态分裂
3. 需要 channel 级别安全策略覆盖
4. 多个 regression bugs 需要更好的回归测试

---

**千斤顶签名** 🔧  
*下次扫描: 2026-03-07 02:04*

## 🔧 千斤顶 Issue 分析 2026-03-07 02:05

### 可贡献 Issues

**优先推荐：**

1. **#38217 - Feishu plugin config validation fails when appId/appSecret in wrong location** [简单] [Bug]
   - 配置验证逻辑问题，Feishu 插件配置位置错误时验证失败
   - 明确的问题域，修复点清晰
   - 涉及配置验证和错误提示改进

2. **#38236 - Browser processes not cleaned up after automation tasks** [中等] [Bug]
   - 浏览器进程清理问题，自动化任务后未正确清理
   - 资源泄漏类问题，影响长期运行稳定性
   - 需要检查进程生命周期管理

**次优选择：**

3. **#38233 - Compaction timeout still freezes sessions in 2026.3.2** [中等] [Bug]
   - Session compaction 超时导致冻结
   - 已有版本历史，可能是回归问题
   - 需要深入理解 session 管理机制

4. **#38223 - WebUI displays "No output" for all tool results** [简单] [Bug]
   - WebUI 显示问题，工具结果显示为空
   - 前端展示逻辑问题，修复相对独立

### 技术方案建议

**#38217 (Feishu 配置验证):**
- 检查 `extensions/feishu` 配置验证逻辑
- 改进错误提示，明确指出正确的配置位置
- 添加配置迁移提示或自动修正

**#38236 (浏览器进程清理):**
- 审查 browser tool 的进程管理代码
- 确保 cleanup 钩子在所有退出路径都被调用
- 考虑添加进程监控和强制清理机制

### 下一步行动

1. 优先处理 #38217 (Feishu 配置验证) - 问题域清晰，修复影响面可控
2. 在 fork 仓库 (Sirius1942/openclaw) 创建分支开发
3. 完成后提交 PR 到上游 openclaw/openclaw


## 🔧 千斤顶 Issue 分析 2026-03-07 03:04

### 📊 本次扫描概况
- 扫描 Issues: 20 个
- 新增 Issues: 18 个（过去 3 小时内）
- 可贡献 Issues: 6 个高价值目标

---

### 🚨 高优先级可贡献 Issues

#### #38275: per-topic model overrides in Telegram forums ⭐⭐⭐⭐⭐
- **类型**: Feature Request
- **难度**: 中等
- **问题**: Telegram forum 支持 per-topic systemPrompt，但不支持 per-topic model
- **方案**: 在 config schema 添加 `channels.telegram.groups.*.topics.*.model` 字段
- **工作量**: 4-6 小时

#### #38273: Gemini 3.1-pro-preview 404 regression ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression
- **难度**: 中等
- **问题**: 2026.2.26 正常，2026.3.2 报 404，model registry 问题
- **方案**: 检查 model registry 变更，恢复 Gemini 3.1 系列注册
- **工作量**: 3-5 小时

#### #38272: Discord thread session binding not working ⭐⭐⭐⭐
- **类型**: Bug
- **难度**: 中等
- **问题**: `sessions_spawn` 创建的 thread 绑定会话无法接收消息
- **方案**: 修复 Discord inbound handler 的 session routing 逻辑
- **工作量**: 5-7 小时

#### #38270: FTS5 tokenizer for CJK languages ⭐⭐⭐⭐⭐
- **类型**: Feature Request（国际化）
- **难度**: 中等
- **问题**: 默认 tokenizer 导致中文 memory search 命中率 55%
- **方案**: 添加可配置 tokenizer（unicode61/cjk/icu），支持字符级分词
- **工作量**: 6-8 小时
- **影响**: 所有 CJK 用户受益


#### #38262: session.resetByChannel not working for Discord ⭐⭐⭐⭐
- **类型**: Bug
- **难度**: 中等
- **问题**: `recordInboundSession` 在 freshness 检查前更新 `updatedAt`，导致 daily reset 失效
- **方案**: 移动 `recordInboundSession` 到 freshness 检查之后
- **工作量**: 3-4 小时

#### #38260: openclaw-gateway crashes with SIGILL in libvips ⭐⭐⭐⭐
- **类型**: Bug (Crash)
- **难度**: 困难
- **问题**: 图片处理时 native crash，涉及 libvips-cpp.so
- **方案**: 需要调查 libvips 版本兼容性，可能需要降级或重新编译
- **工作量**: 8-12 小时

---

### 🎯 推荐贡献目标

**首选: #38270 (CJK tokenizer)**
- 社区影响最大（所有中日韩用户）
- 技术方案清晰，已有社区 workaround 验证
- 提升国际化支持

**次选: #38275 (Telegram per-topic model)**
- 用户需求明确
- 实现相对简单（schema + routing）
- 增强 Telegram forum 功能

**备选: #38273 (Gemini 3.1 regression)**
- Regression 高优先级
- 需要快速修复避免影响更多用户

---

### 💡 技术洞察

1. **国际化支持**: FTS5 tokenizer 问题影响所有非英语用户，需要可配置方案
2. **配置层级**: Telegram forum 需要更细粒度的配置继承（topic > group > agent）
3. **会话路由**: Discord thread binding 暴露了 session routing 的复杂性
4. **Model registry**: Provider 模型更新需要更好的版本跟踪机制
5. **进程生命周期**: 多个 issue 涉及资源清理问题（browser、native libs）

---

### 📋 下一步行动

1. **立即开始**: #38270 (CJK tokenizer) - 创建分支 `feat/fts5-cjk-tokenizer`
2. **并行准备**: 研究 #38275 的 config schema 变更范围
3. **监控**: 跟踪 #38273 是否有官方快速修复

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*  
*下次扫描: 2026-03-07 04:04*

## 🔧 千斤顶 Issue 分析 2026-03-07 04:04

### 📊 本次扫描概况
- 扫描时间: 2026-03-07 04:04 (周六凌晨)
- 扫描 Issues: 20 个
- 新增 Issues: 大量新 bug 和 regression
- 可贡献 Issues: 4 个高价值目标

---

### 🚨 高优先级可贡献 Issues

#### #38305: Subagent Model Resolution Broken in v2026.3.1+ ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression
- **难度**: 中等
- **标签**: bug, regression
- **问题描述**: 
  - Subagent 模型解析在 v2026.3.1+ 版本损坏
  - 影响所有使用 subagent 的场景
  - 严重的回归问题
- **技术方案**: 
  - 对比 v2026.3.0 和 v2026.3.1 的 model resolution 代码变更
  - 定位引入 regression 的 commit
  - 恢复正确的 model resolution 逻辑
- **涉及文件**: `src/session/spawn.ts`, `src/model/resolver.ts`
- **预计工作量**: 4-6 小时
- **优先级理由**: Regression + 影响核心功能

#### #38298: Config hot-reload corrupts provider adapter state ⭐⭐⭐⭐⭐
- **类型**: Bug (严重)
- **难度**: 困难
- **问题描述**:
  - 配置热重载后 provider adapter 状态损坏
  - 持续 404 错误直到 gateway 重启
  - 影响生产环境稳定性
- **技术方案**:
  - 审查 config reload 时的 provider adapter 生命周期
  - 确保 adapter 正确重新初始化或保持状态一致性
  - 添加 adapter health check
- **涉及文件**: `src/gateway/config-watcher.ts`, `src/providers/adapter.ts`
- **预计工作量**: 8-10 小时

#### #38284: TUI color contrast for light-background users ⭐⭐⭐
- **类型**: Bug (UX)
- **难度**: 简单
- **标签**: bug, bug:behavior
- **问题描述**: TUI 颜色对比度对浅色背景用户不友好
- **技术方案**: 
  - 检测终端背景色或添加配置选项
  - 调整颜色方案以提高对比度
- **涉及文件**: `src/tui/theme.ts`
- **预计工作量**: 2-3 小时

#### #38273: Gemini 3.1-pro-preview returns 404 in 2026.3.2 ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression
- **难度**: 中等
- **标签**: bug, regression
- **问题描述**:
  - Gemini 3.1-pro-preview 在 2026.3.2 返回 404
  - 已有 PR #38276 修复但被 barnacle bot 关闭
  - 阻塞大量用户
- **技术方案**: 
  - Review PR #38276 的修复方案
  - 重新提交或请求合并
- **涉及文件**: `src/providers/gemini/models.ts`
- **预计工作量**: 2-3 小时
- **状态**: 已有修复 PR，需要推动合并

---

### 🎯 推荐贡献目标

**首选: #38305 (Subagent Model Resolution Regression)**
- **理由**:
  - 严重 regression，影响核心功能
  - 版本对比可快速定位问题
  - 高优先级修复需求
  
**开发步骤**:
1. `cd ~/Projects/openclaw-dev && git checkout main && git pull upstream main`
2. `git checkout -b fix/subagent-model-resolution-regression`
3. 对比 v2026.3.0 和 v2026.3.1 的相关代码
4. 定位并修复 model resolution 逻辑
5. 测试 subagent spawn 场景
6. 提交 PR 到 `openclaw/openclaw`

**次选: #38284 (TUI color contrast)**
- **理由**: 简单、低风险、改善用户体验

**备选: #38273 (Gemini 3.1 404)**
- **理由**: 已有 PR，可以 review 并推动合并

---

### 💡 技术洞察

1. **Regression 管理**: 多个严重 regression (#38305, #38273, #38298) 说明需要更好的回归测试覆盖
2. **Config 热重载**: #38298 暴露了状态管理的复杂性，需要更健壮的 reload 机制
3. **PR 管理**: #38273 的修复被 bot 关闭，说明 PR limit 策略可能需要调整
4. **用户体验**: #38284 提醒需要考虑不同终端环境的可访问性

---

### 📋 下一步行动

1. **立即开始**: #38305 (Subagent model resolution) - 创建分支并开始调查
2. **监控**: 跟踪 #38273 的 PR #38276 状态
3. **准备**: 研究 #38298 的 config reload 机制

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*  
*下次扫描: 2026-03-07 05:04*

## 🔧 千斤顶 Issue 分析 2026-03-07 05:04

### 📊 本次扫描概况
- 扫描时间: 2026-03-07 05:04 (周六凌晨)
- 扫描 Issues: 20 个
- 新增严重 Issues: 多个高影响 bug 和 regression
- 可贡献 Issues: 3 个高价值目标

---

### 🚨 高优先级可贡献 Issues

#### #38335: Auth profile failover: no retry/backoff for recovered profiles ⭐⭐⭐⭐⭐
- **类型**: Feature Request (稳定性改进)
- **难度**: 中等
- **问题描述**:
  - Auth profile 失败后降级到备用 profile，但永不重试高优先级 profile
  - OAuth token 恢复后系统仍停留在低优先级 fallback
  - 无法区分临时失败（rate limit）和永久失败（broken token）
- **技术方案**:
  - 添加周期性 profile promotion 机制（每 N 分钟探测失败的高优先级 profile）
  - 区分失败原因（rate_limit / auth_error / timeout）
  - 添加配置项 `auth.retryFailedProfilesEveryMs`
- **涉及文件**: `src/auth/failover.ts`, `src/auth/profile-manager.ts`
- **预计工作量**: 6-8 小时
- **优先级理由**: 影响生产环境成本和可用性

#### #38327: "Cannot convert undefined or null to object" in 2026.3.2 with Gemini ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression (Critical)
- **难度**: 中等
- **标签**: bug, regression
- **问题描述**:
  - 2026.3.2 版本使用 google-vertex/gemini-3.1-pro-preview 时完全损坏
  - 任何消息都触发 "Cannot convert undefined or null to object" 错误
  - 2026.2.26 版本正常工作
- **技术方案**:
  - 对比 2026.2.26 和 2026.3.2 的 Gemini provider 代码变更
  - 定位 null/undefined 访问点
  - 添加防御性检查
- **涉及文件**: `src/providers/google-vertex/adapter.ts`, `src/agent/embedded.ts`
- **预计工作量**: 4-6 小时
- **优先级理由**: Critical regression，完全阻塞 Gemini 用户

#### #38321: agents.list[].runtime rejected by schema validation ⭐⭐⭐⭐
- **类型**: Bug (文档与实现不一致)
- **难度**: 简单
- **问题描述**:
  - 文档中的 `agents.list[].runtime` 配置被 schema 验证拒绝
  - 导致 ACP 后端使用错误的 cwd，创建意外的 workspace 目录
- **技术方案**:
  - 更新 config schema 添加 `agents.list[].runtime` 字段
  - 确保 ACP backend 正确读取 per-agent runtime config
- **涉及文件**: `src/config/schema.ts`, `src/agents/acp.ts`
- **预计工作量**: 3-4 小时

---

### 🎯 推荐贡献目标

**首选: #38327 (Gemini 3.1 Critical Regression)**
- **理由**:
  - Critical severity - 完全阻塞功能
  - Regression - 需要紧急修复
  - 影响大量用户
  - 版本对比可快速定位

**开发步骤**:
1. `cd ~/Projects/openclaw-dev && git checkout main && git pull upstream main`
2. `git checkout -b fix/gemini-null-object-regression`
3. 对比 v2026.2.26 和 v2026.3.2 的 Gemini provider 代码
4. 定位 null/undefined 访问，添加防御性检查
5. 测试 google-vertex/gemini-3.1-pro-preview
6. 提交 PR 到 `openclaw/openclaw`

**次选: #38321 (agents.list[].runtime schema)**
- **理由**: 简单、明确、文档与实现一致性问题

**备选: #38335 (Auth profile retry)**
- **理由**: 高价值功能改进，但工作量较大

---

### 💡 技术洞察

1. **Regression 频发**: 连续多个版本出现严重 regression (#38327, #38305, #38273)，说明需要更完善的回归测试套件
2. **Config Schema 维护**: #38321 暴露了文档与 schema 不同步的问题
3. **Auth 弹性**: #38335 揭示了 failover 机制的不完整性，生产环境需要自动恢复机制
4. **Provider 稳定性**: 多个 provider 相关 bug 说明需要标准化测试框架

---

### 📋 下一步行动

1. **立即开始**: #38327 (Gemini null object) - 创建分支并开始调查
2. **并行准备**: 研究 #38321 的 schema 变更范围
3. **长期规划**: 设计 #38335 的 auth retry 机制

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*  
*下次扫描: 2026-03-07 06:04*

## 🔧 千斤顶 Issue 分析 2026-03-07 06:06

### 📊 本次扫描概况
- 扫描 Issues: 20 个
- 新增 Issues: 18 个（过去 1 小时内）
- 可贡献 Issues: 3 个高价值目标

---

### 🚨 高优先级可贡献 Issues

#### #38351: AI agent replies twice with same text after update ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression
- **难度**: 中等
- **标签**: bug, regression
- **问题**: 更新到最新版本后，AI 回复重复两次
- **技术方案**: 检查消息发送逻辑，可能是 reply handler 被调用两次
- **工作量**: 3-5 小时

#### #38336: OAuth renewal doesn't update provisioned.json ⭐⭐⭐⭐⭐
- **类型**: Bug (Critical)
- **难度**: 中等
- **问题**: OAuth token 更新后只写入 auth-profiles.json，不更新 provisioned.json，导致恢复循环
- **技术方案**: 在 OAuth renewal 时同步更新两个文件
- **工作量**: 4-6 小时

#### #38312: Model alias defaults need urgent update ⭐⭐⭐⭐⭐
- **类型**: Bug (Urgent - 3 days deadline)
- **难度**: 简单
- **问题**: Gemini 3 Pro 将于 3 月 9 日弃用，需更新默认别名
- **技术方案**: 
  - Line 27: `gpt: "openai/gpt-5.4"` (was gpt-5.2)
  - Line 31: `gemini: "google/gemini-3.1-pro-preview"`
- **工作量**: 1-2 小时
- **紧急**: 3 天后 Gemini 3 Pro 停用

---

### 🎯 推荐贡献目标

**首选: #38312 (Model alias update - URGENT)**
- **理由**: 
  - 3 天截止期限
  - 修复极简单（2 行代码）
  - 已有社区提供 diff
  - 影响所有使用默认别名的用户

**开发步骤**:
```bash
cd ~/Projects/openclaw-dev
git checkout main && git pull upstream main
git checkout -b fix/update-model-aliases-march-2026
# 编辑 src/config/defaults.ts
git commit -m "fix: bump gpt→5.4, gemini→3.1-pro-preview (March 9 deprecation)"
git push origin fix/update-model-aliases-march-2026
gh pr create --repo openclaw/openclaw --title "fix: update model aliases before March 9 deprecation" --body "Fixes #38312"
```

**次选: #38351 (Duplicate replies regression)**
- **理由**: Regression，影响用户体验

**备选: #38336 (OAuth provisioned.json sync)**
- **理由**: Critical bug，但需要更深入理解 auth 架构

---

### 💡 技术洞察

1. **紧急维护**: #38312 提醒需要跟踪上游 provider 弃用时间表
2. **Auth 双写**: #38336 暴露了双文件同步问题，需要统一写入接口
3. **消息去重**: #38351 可能与最近的 reply handler 重构有关

---

### 📋 下一步行动

1. **立即开始**: #38312 (Model aliases) - 最高优先级，简单快速
2. **并行调查**: #38351 (Duplicate replies) - 定位 regression commit
3. **深入研究**: #38336 (OAuth sync) - 理解 auth-profiles 架构

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*  
*下次扫描: 2026-03-07 07:06*

## 🔧 千斤顶 Issue 分析 2026-03-07 07:04

### 📊 本次扫描概况
- 扫描 Issues: 20 个
- 新增 Issues: 18 个（过去 1 小时内）
- 可贡献 Issues: 4 个高价值目标

---

### 🚨 高优先级可贡献 Issues

#### #38382: Recursively scan symlinked directories in workspace ⭐⭐⭐⭐
- **类型**: Feature Request
- **难度**: 中等
- **问题**: workspace context injection 不递归扫描符号链接目录
- **方案**: 修改 workspace scanner 支持 `followSymlinks` 选项，添加循环检测
- **工作量**: 4-6 小时

#### #38379: openclaw onboard --install-daemon broken on v2026.3.2 ⭐⭐⭐⭐⭐
- **类型**: Bug + Regression (Critical)
- **难度**: 中等
- **问题**: v2026.3.2 版本 onboard --install-daemon 损坏，v2026.3.1 正常
- **方案**: 对比版本差异，定位 regression commit
- **工作量**: 3-5 小时
- **优先级**: 阻塞新用户安装

#### #38372: pnpm version pinned at 10.23.0, should be 10.30.3 ⭐⭐⭐
- **类型**: Maintenance
- **难度**: 简单
- **方案**: 更新 package.json 和 CI 配置
- **工作量**: 1-2 小时

#### #38370: limitHistoryTurns breaks tool_use/tool_result pairs ⭐⭐⭐⭐⭐
- **类型**: Bug (Critical)
- **难度**: 中等
- **问题**: history sanitization 破坏 tool_use/tool_result 配对，导致 API 失败
- **方案**: 修改 sanitizer 保持 message pair 原子性
- **工作量**: 4-6 小时

---

### 🎯 推荐贡献目标

**首选: #38379 (onboard --install-daemon regression)**
- Critical severity - 阻塞新用户
- Regression - 版本对比可快速定位
- 高优先级修复需求

**次选: #38370 (limitHistoryTurns tool pairs)**
- Critical bug，破坏核心功能

**备选: #38372 (pnpm version update)**
- 简单、低风险、维护性改进

---

### 💡 技术洞察

1. **Regression 管理**: v2026.3.2 引入多个严重 regression
2. **History Sanitization**: 需要更智能的 message pair 原子性处理
3. **Symlink 处理**: 需要考虑复杂文件系统场景
4. **依赖更新**: 需要定期更新工具链版本

---

### 📋 下一步行动

1. **立即开始**: #38379 (onboard regression) - 创建分支调查
2. **并行准备**: 研究 #38370 的 history sanitizer 逻辑
3. **快速修复**: #38372 (pnpm update)

---

**千斤顶签名** 🔧  
*"精准设计，稳定实现"*  
*下次扫描: 2026-03-07 08:04*
