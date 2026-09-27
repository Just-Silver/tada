# 加密备份设计（Encrypted Backup）

| 项 | 内容 |
|---|---|
| 日期 | 2026-09-27 |
| 状态 | **Approved（方向与参数已定，待实现）** |
| 关联 | `docs/security/TODO.md` 第 1 项 |
| 基线 | `baseline`（fork 自 `LoadShine/tada`） |
| 目标分支 | `fix/export-secret-leak` |

---

## 1. Goal

让"导出全部数据"生成的备份文件在**离开用户设备后仍不可被读取**，同时**不牺牲备份/迁移的完整性**——备份里依然能带上 AI 配置与代理配置，换机迁移后可直接使用。

一句话：**对导出的整体数据做口令加密；导出设密码，导入输密码解密；始终加密，不提供不加密入口。**

---

## 2. Current State（现状）

- 导出入口：`DataSettings.tsx:301 handleExportData` → `storage.exportData()`；另有未被调用的死代码 `store/data.ts:29 exportDataAtom`。
- 两个存储实现：`packages/web/src/services/localStorageService.ts:681`、`packages/desktop/src/services/sqliteStorageService.ts:863`。
- `exportData()` 返回**全量明文**对象：`settings(appearance/preferences/ai/proxy)` + `userProfile` + `lists` + `tasks` + `summaries` + `echoReports`。
- 其中 `settings.ai.apiKey`、`settings.proxy.password` 为**明文敏感凭据**。
- 导出为纯 JSON，文件名 `tada-backup-YYYY-MM-DD.json`。
- 产品定位：UI 文案明确为**本地备份**（EN: "Create a local backup of all your data in a JSON file."；ZH: "将您的所有数据创建一个本地 JSON 文件备份。"），配套导入有冲突解决与 `replaceAllData`，面向**备份/恢复 + 换机迁移**，**不是分享**。
- 导入：`DataSettings.tsx:379 handleImportFile` 读取 JSON → `analyzeImport` → `importData`，默认 `includeSettings: true`。

---

## 3. Problem

- 备份文件是**明文**，一旦被上传网盘 / 邮件 / U 盘 / 转发给他人，`apiKey` 与 `proxy.password` 直接泄露。
- 备份天然会被"搬来搬去"，正是最容易被外流的产物；而现有实现对其中的凭据毫无保护。

---

## 4. Decision（已定方向）

**整包加密**：在导出对象序列化后，对整体做口令加密，产出信封格式的密文文件；导入时校验信封、要求输入口令、解密后走原有 `importData`。

选择理由：
- **不碰数据结构**：`exportData()` / `importData()` 的字段逻辑、冲突解决全部不变，只在最外层包一层。
- **保留完整性**：密钥与代理配置随包迁移，换机后 AI 即可用。
- **默认且始终安全**：不提供"不加密导出"入口，消除明文外流的可能。

---

## 5. Design

### 5.1 信封格式（仅"基本信息"明文，其余全部加密）

```json
{
  "format": "tada-encrypted-backup",
  "version": 1,
  "exportedAt": 1758931200000,
  "platform": "web",
  "kdf": "PBKDF2-SHA256",
  "iterations": 600000,
  "salt": "<base64>",
  "iv": "<base64>",
  "data": "<base64 密文>"
}
```

**明文暴露范围（仅"基本信息"）**：`format`、`version`、`exportedAt`、`platform`，以及解密所必需的 `kdf` / `iterations` / `salt` / `iv`。这些均不含任何用户数据。

**加密范围**：`data` 之内的一切——`settings`（含 `ai.apiKey`、`proxy.password`）、`userProfile`、`lists`、`tasks`、`summaries`、`echoReports`，即现有 `JSON.stringify(exportedData)` 的完整内容。

- `format` 作为加密备份的识别标记（导入时据此判定是否需要密码）。
- `version` 预留格式演进。
- `exportedAt` / `platform` 仅用于辨识文件。

### 5.2 加密算法

- **AES-GCM**（加密 + 认证；篡改/密码错误会在解密时失败）。
- **PBKDF2-SHA256** 派生密钥，随机 `salt`、随机 `iv`、迭代次数 ≥ 600000。
- 使用浏览器原生 **Web Crypto（`crypto.subtle`）**；Web 与 Tauri WebView 均支持。
- **禁止**自造/弱加密（base64、异或、无认证的 AES-CBC 等）。

### 5.3 导出流程

1. 用户点导出 → 弹出**设置密码**对话框：输入 + 二次确认。
2. **始终加密**：不提供"不加密导出"入口；密码为空、长度不足或两次不一致时不允许导出。
3. `storage.exportData()` 取明文对象 → `JSON.stringify`。
4. 加密 → 生成信封 JSON。
5. 下载 `tada-backup-YYYY-MM-DD.json`。
6. 显著提示：**"密码丢失将无法恢复此备份，请自行妥善保管。"**

### 5.4 导入流程

1. 读取文件文本。
2. 若含 `format: "tada-encrypted-backup"` → 弹出**输入密码**对话框 → 解密。
3. 解密成功 → 走原有 `analyzeImport` / `importData`。
4. 解密失败（GCM 认证失败）→ 提示 **"密码错误或文件已损坏"**，不崩溃。

### 5.5 旧格式兼容（必须）

- 已存在的明文 `tada-backup-*.json`（无 `format` 标记）→ 视为旧格式，**直接按现状导入**，不要求密码。
- 保证向后兼容，不让老备份作废。

### 5.6 密码策略

- **最低长度**：≥ 8 个字符（导出与导入均校验）。
- **强度提示**：按长度与字符类别给出弱 / 中 / 强提示。
- **二次确认**：导出时必须两次输入一致。
- 密码**不做任何存储**，仅用于本次加解密，丢失无法找回。

### 5.7 代码落点（不改 storage 接口）

- 新增 `packages/core/src/utils/backupCrypto.ts`：
  - `encryptBackup(obj, password): Promise<string>`
  - `isEncryptedBackup(text): boolean`
  - `decryptBackup(text, password): Promise<unknown>`
- 改 `packages/core/src/components/features/settings/DataSettings.tsx`：导出/导入各加密码对话框，`handleImportFile` 支持加密与明文两种。
- i18n（`en` + `zh-CN`）：密码输入 / 二次确认 / 强度提示 / 错误 / 丢失警告等文案。
- `store/data.ts` 的 `exportDataAtom` 为死代码，可选顺手清理或标记。

> 加解密是异步（`SubtleCrypto` 返回 Promise），而 `exportData()` 是同步方法，因此加解密放在 **UI/工具层**，不改动 `IStorageService` 签名。

---

## 6. Alternatives Considered

| 方案 | 说明 | 结论 |
|---|---|---|
| A. 字段级剥离 | 导出时剔除/置空 `apiKey`、`proxy.password`，导入时空值不覆盖 | 改动散布在数据层，且换机迁移需重填密钥；否决 |
| B. 可选含凭据 | 默认剥离，提供"包含凭据"开关 + 警告 | 仍留明文窗口，用户可能误勾选后外流；否决 |
| C. 明文 + 警告 | 保留明文仅加提示 | 不解决外流；否决 |
| **D. 整包加密（选定）** | 导出设密码、导入输密码，始终加密 | **采用** |

---

## 7. Security Considerations

- **忘记密码 = 备份永久不可恢复**：无法找回，必须二次确认 + 显著警告。
- **密码策略**：最低 8 位 + 强度提示。
- **仅保护"文件外流"**：拿到密码者可得全部内容；这不是字段级脱敏。
- **本机明文存储不在本规格范围**：`localStorage` / SQLite 中的凭据仍是明文，见 `docs/security/TODO.md` 第 3 项，独立处理。
- GCM 认证保证密文完整性；不使用无认证的加密模式。

---

## 8. Out of Scope

- 本机存储层加密（TODO 第 3 项）。
- 字段级脱敏 / "仅导出任务不含设置"。
- 云同步、团队共享等在线能力。
- 密码找回 / 密钥托管。

---

## 9. Verification

- [ ] 导出加密备份 → 用同一密码导入，数据（任务/列表/摘要/设置）完全一致。
- [ ] 错误密码导入 → 明确报"密码错误或文件损坏"，无崩溃、无部分写入。
- [ ] 旧明文备份导入 → 仍可直接导入（向后兼容）。
- [ ] 篡改密文（改一个字节）→ 解密失败被正确拒绝。
- [ ] 密码为空 / 长度 < 8 / 两次不一致 → 导出被阻止。
- [ ] 导出的信封仅含规定的"基本信息"明文，无任何用户数据外泄。
- [ ] Web（localStorage）与 Desktop（SQLite）两条路径均验证。
- [ ] 导出/导入的密码对话框支持取消，且取消后不产生空文件 / 不误清数据。

---

## 10. Decisions（已拍板）

1. **始终加密**：不提供"不加密导出"入口。
2. **密码策略**：最低长度 ≥ 8 + 强度提示；导出需二次确认。
3. **明文暴露范围**：仅"基本信息"（`format` / `version` / `exportedAt` / `platform` 及解密必需的 `kdf` / `iterations` / `salt` / `iv`）；其余**全部加密**。
