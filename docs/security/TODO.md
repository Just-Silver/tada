# Tada 安全与隐私修复 TODO

> 来源：2026-09-27 安全与隐私审查
> 基线分支：`baseline`（`fc3cb80`，已推送 `origin/baseline`）
> 用法：逐项推进，完成后更新状态。修复按主题拆分支，通用改进可向上游 `LoadShine/tada` 提 PR。

**状态图例**：`[ ]` 待办 · `[~]` 进行中 · `[x]` 已完成 · `⤴` 适合贡献上游

---

## 🔴 高优先级（凭据泄露）

- [ ] **1. 导出数据包含明文 API Key / 代理密码** ⤴
  - 位置：`packages/web/src/services/localStorageService.ts:681-702`、`packages/desktop/src/services/sqliteStorageService.ts`（`exportData`）、`packages/core/src/components/features/settings/DataSettings.tsx:301-333`
  - 问题：`exportData()` 原样导出 `settings.ai.apiKey` 与 `settings.proxy.password`；导入时 `includeSettings` 默认 `true`（`DataSettings.tsx:392`）。
  - 修复方向：**整包口令加密**——导出设密码、导入输密码解密；保留旧明文备份兼容。详见 [`specs/2026-09-27-encrypted-backup-design.md`](./specs/2026-09-27-encrypted-backup-design.md)。
  - 建议分支：`fix/export-secret-leak`

- [ ] **2. 代理凭据被写入控制台日志** ⤴
  - 位置：`packages/core/src/utils/networkUtils.ts:57-63`、`packages/core/src/components/features/settings/SettingsModal.tsx:1516`
  - 问题：打印含 `user:password@` 的 `proxyUrl`，以及整个含 `password` 的代理设置对象。
  - 修复：删除或脱敏这两处日志。
  - 建议分支：`fix/proxy-credential-log`

## 🟠 中优先级

- [ ] **3. 敏感数据明文存储，且文案与实际不符** ⤴
  - 位置：`localStorageService.ts`、`sqliteStorageService.ts:314-332`、`packages/core/src/locales/*/translation.json:200`、`packages/core/public/content/privacy-policy*.md:80`
  - 问题：API Key / 代理密码 / 任务全文以明文落盘（localStorage / SQLite `settings` 表），但文案称"安全地存储""静态加密"。
  - 修复：接入系统钥匙串（Tauri stronghold / keyring）单独加密密钥；否则如实修改文案。
  - 建议分支：`harden/secure-credential-storage`

- [ ] **4. Tauri CSP 为 null，且 HTTP 白名单全开** ⤴
  - 位置：`packages/desktop/src-tauri/tauri.conf.json:14`（`"csp": null`）、`:34-45`（`http://**`、`https://**`）
  - 修复：设置严格 CSP（`default-src 'self'` 起步），HTTP 白名单收窄到实际使用的 provider 域名。
  - 建议分支：`harden/tauri-csp`

- [ ] **5. ICS 自动同步会持续上传全部任务（含正文）**
  - 位置：`packages/core/src/services/icsAutoSync.ts`、`packages/core/src/App.tsx:53`、`packages/core/src/services/icsService.ts:26-42`
  - 问题：只要配置了服务器地址，任何任务变更都会把全部任务（含 `content` 正文）POST 出去；允许 `http://` 明文传输。
  - 修复：默认关闭 + 显式授权；仅允许 `https://`；明确提示上传范围。
  - 建议分支：`harden/ics-privacy`

- [ ] **6. 依赖漏洞（16 high / 21 moderate / 7 low）** ⤴
  - 重点：`react-router-dom`（`@remix-run/router` —— XSS via Open Redirect，运行时可达）；其余为构建链传递依赖（`glob`、`minimatch`、`picomatch`、`brace-expansion`、`postcss`、`browserslist`、`nanoid`）。
  - 修复：升级 `react-router-dom` 至修复版本，并统一 upgrade 传递依赖。
  - 建议分支：`chore/deps-security`

- [ ] **7. 隐私政策与实现自相矛盾（合规风险）** ⤴
  - 位置：`packages/core/public/content/privacy-policy.en.md` / `privacy-policy.zh-CN.md`（对比 `README.md` 的"零数据收集"声明）
  - 问题：政策声称使用 Google Analytics、错误跟踪、CDN 收集 IP、静态加密；代码中均不存在。
  - 修复：统一为"纯本地、零收集、无分析"，删除未使用的 Analytics / 加密条款。
  - 建议分支：`docs/privacy-policy-consistency`

## 🟡 低优先级

- [ ] **8. SQL 表名字符串拼接**
  - 位置：`packages/desktop/src/services/sqliteStorageService.ts:230`（`DELETE FROM ${op.table}`）
  - 说明：表名当前受内部枚举控制，非用户输入，风险低。修复：改为白名单校验。

- [ ] **9. `.gitignore` 未忽略 `.env` / `.env.*`**
  - 位置：`.gitignore`
  - 说明：当前仓库无 env 文件、git 历史干净，仅作预防。

- [ ] **10. 错误信息回显上游响应体**
  - 位置：`packages/core/src/services/aiService.ts:177,239,373,444,545`
  - 说明：把 provider 返回体拼进错误消息，可能把服务端内容带入 UI。修复：错误信息收敛。

- [ ] **11. docker-compose 开发配置**
  - 位置：`docker-compose.yml`
  - 说明：整仓挂载 + 容器内 root 安装依赖，仅限本地开发使用，勿用于生产。

- [ ] **12. 自定义 `baseUrl` + HTTP 全放开**
  - 与第 4 项联动修复。

---

## 建议推进顺序

1. **1 → 2**：凭据泄露（确定性最高，改动小）
2. **4 → 3**：桌面端加固
3. **5、6、7**：隐私 / 合规 / 依赖
4. **8 - 12**：低风险收尾
