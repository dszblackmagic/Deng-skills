---
name: prd-test-execution
description: Use when confirmed PRD test planning documents need browser automation verification, field comparison, failure evidence, Chrome DevTools MCP, or Playwright CLI recording.
---

# PRD Test Execution

## 目标

基于已确认的 `01-04` 规划资料执行模块自动化验证，按用例优先级目录记录结果，并对失败进行分流取证。此 skill 不负责重新设计测试用例；发现规划缺口时先阻塞并要求回到规划阶段补齐。

## 执行前门禁

开始页面验证前必须确认：

1. `01-模块拆解.md`、`02-测试用例.md`、`03-执行清单.md`、`04-测试数据与账号.md` 已存在并经用户确认。
2. 测试环境地址和测试路径已确认，测试路径可以是菜单路径或 URL。
3. 登录方式已确认：指定测试账号登录、复用当前已登录状态，或无需登录直接访问。
4. 执行模式已确认：单 agent 独立执行，或多 agent 并行执行。
5. 高风险动作授权边界已确认。

未满足门禁时，不进入页面执行；把阻塞原因写入对应优先级目录的 `05-执行记录.md`，无法归属优先级时写入根目录汇总 `05-执行记录.md`。

## 依赖检查

执行前检查当前环境是否具备 `chrome-devtools-mcp` 和 `playwright-cli`。

### chrome-devtools-mcp

按顺序判断：

1. 当前会话可用工具中存在 `mcp__chrome_devtools__` 时，视为可用。
2. 否则运行 `codex mcp list`，检查是否存在 `chrome-devtools` 配置。
3. 仍不可用时，先读取官方仓库或官方文档确认最新安装方式，再询问用户是否安装；不得自动安装。

当前已知的 Codex 推荐命令：

```bash
codex mcp add chrome-devtools -- npx chrome-devtools-mcp@latest
```

需要全局 CLI 时可使用：

```bash
npm i chrome-devtools-mcp@latest -g
```

官方地址：https://github.com/ChromeDevTools/chrome-devtools-mcp

### playwright-cli

按顺序判断：

1. 运行 `command -v playwright-cli`。
2. 不存在时运行 `npm list -g @playwright/cli --depth=0`。
3. 仍不可用时，先读取官方仓库或官方文档确认最新安装方式，再询问用户是否安装；不得自动安装。

当前已知安装命令：

```bash
npm install -g @playwright/cli@latest
playwright-cli install --skills
```

官方地址：https://github.com/microsoft/playwright-cli

## 执行目录

按用例优先级创建并写入：

- `执行结果/P0/`
- `执行结果/P1/`
- `执行结果/P2/`
- `执行结果/P3/`

每个优先级目录必须包含：

- `05-执行记录.md`
- `06-缺陷记录.md`
- `screenshots/`
- `videos/`
- `scripts/`

根目录 `05-执行记录.md` 和 `06-缺陷记录.md` 是最终汇总产物，只在各优先级执行完成或阶段性收口后生成或更新。

## 执行规则

1. 单 agent 独立执行时，默认按 `P0 -> P1 -> P2 -> P3` 执行，但仍写入各自优先级目录。
2. 多 agent 并行执行时，每个 agent 只能写自己负责的优先级目录，不直接写根目录汇总文件。
3. 已登录时复用当前会话，不重复登录；未登录且登录方式为指定账号时再执行登录。
4. `P0` 验证页面进入、核心入口和主链路；`P1` 验证关键场景和关键字段；`P2` 验证异常、权限和边界；`P3` 仅做低频或观察性补充。
5. 通过用例不截图，只在执行记录中写实际结果摘要。
6. 缺账号、缺权限、缺数据、环境不稳定或依赖受限时，标记为“阻塞”，默认不登记缺陷。
7. 高风险动作未授权时，只验证入口、展示、校验、二次确认或提示，不执行真实提交。

## 字段一致性比对

默认比对查询区、列表表头、详情区、关键操作按钮和关键弹窗字段。字段差异包括缺少、多出、命名不一致和语义疑似不一致。

字段差异先记录到所属优先级目录 `05-执行记录.md`。确认属于产品设计或实现问题后，再写入同目录 `06-缺陷记录.md`，最终汇总到根目录 `06-缺陷记录.md`。

字段类失败只截图，不录视频。

## 失败分流

字段类失败：

1. 截图。
2. 记录预期字段、实际字段和差异分类。
3. 关联 `F-xxx` 功能编号、字段区域或用例编号。

操作类失败：

1. 首轮截图。
2. 采集 Console 摘要。
3. 采集 Network 摘要。
4. 使用 `playwright-cli` 重跑并录制视频。

如果 `playwright-cli` 不可用，不阻塞整体执行；在执行记录中说明依赖不可用，并退化为截图、Console 和 Network 证据。

## 完成标准

至少满足：

1. 已执行或阻塞记录 `P0/P1` 用例。
2. 已执行的优先级目录均存在独立 `05-执行记录.md` 和 `06-缺陷记录.md`。
3. 字段差异、失败、阻塞或高风险提示证据已落盘并在记录中引用。
4. 根目录汇总 `05-执行记录.md` 已更新。
5. 若存在确认缺陷，根目录汇总 `06-缺陷记录.md` 已更新。

## 模板

优先复用本 skill 的模板：

- `templates/05-执行记录.md`
- `templates/06-缺陷记录.md`
- `templates/执行结果/优先级/05-执行记录.md`
- `templates/执行结果/优先级/06-缺陷记录.md`
