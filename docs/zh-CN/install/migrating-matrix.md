---
summary: "OpenClaw 如何就地升级以前的 Matrix 插件,包括加密状态恢复限制和手动恢复步骤。"
read_when:
  - 升级现有 Matrix 安装
  - 迁移加密的 Matrix 历史记录和设备状态
title: "Matrix 迁移"
x-i18n:
  generated_at: "2025-01-09T00:00:00Z"
  model: "claude-sonnet-4-6"
  provider: "pi"
  source_hash: "69b7530846b977632b699dc3f880841c959c157919fb8115b783eca674a497fb"
  source_path: "docs/install/migrating-matrix.md"
  workflow: 15
---

# Matrix 迁移

本页面介绍从以前的公共 `matrix` 插件升级到当前实现的内容。

对于大多数用户来说,升级是就地进行的:

- 插件保持为 `@openclaw/matrix`
- 渠道保持为 `matrix`
- 您的配置保留在 `channels.matrix` 下
- 缓存的凭证保留在 `~/.openclaw/credentials/matrix/` 下
- 运行时状态保留在 `~/.openclaw/matrix/` 下

您不需要重命名配置键或以新名称重新安装插件。

## 迁移自动执行的操作

当 Gateway 网关启动时,以及当您运行 [`openclaw doctor --fix`](/gateway/doctor) 时,OpenClaw 会尝试自动修复旧的 Matrix 状态。
在任何可操作的 Matrix 迁移步骤改变磁盘状态之前,OpenClaw 会创建或重用一个聚焦的恢复快照。

当您使用 `openclaw update` 时,确切的触发器取决于 OpenClaw 的安装方式:

- 源码安装在更新流程中运行 `openclaw doctor --fix`,然后默认重启 Gateway 网关
- 包管理器安装更新包,运行非交互式 doctor 检查,然后依赖默认的 Gateway 网关重启,以便启动可以完成 Matrix 迁移
- 如果您使用 `openclaw update --no-restart`,则启动支持的 Matrix 迁移会延迟到您稍后运行 `openclaw doctor --fix` 并重启 Gateway 网关时

自动迁移涵盖:

- 在 `~/Backups/openclaw-migrations/` 下创建或重用预迁移快照
- 重用您缓存的 Matrix 凭证
- 保持相同的账户选择和 `channels.matrix` 配置
- 将最旧的扁平 Matrix 同步存储移动到当前的账户范围位置
- 当目标账户可以安全解析时,将最旧的扁平 Matrix 加密存储移动到当前的账户范围位置
- 从旧的 rust 加密存储中提取先前保存的 Matrix 房间密钥备份解密密钥(如果该密钥本地存在)
- 当访问令牌稍后更改时,为同一 Matrix 账户、homeserver 和用户重用最完整的现有令牌哈希存储根
- 当 Matrix 访问令牌更改但账户/设备身份保持不变时,扫描兄弟令牌哈希存储根以查找待处理的加密状态恢复元数据
- 在下一次 Matrix 启动时将备份的房间密钥恢复到新的加密存储中

快照详情:

- OpenClaw 在成功快照后在 `~/.openclaw/matrix/migration-snapshot.json` 写入一个标记文件,以便稍后的启动和修复过程可以重用同一归档。
- 这些自动 Matrix 迁移快照仅备份配置 + 状态(`includeWorkspace: false`)。
- 如果 Matrix 只有警告性迁移状态,例如因为 `userId` 或 `accessToken` 仍然缺失,OpenClaw 还不会创建快照,因为没有 Matrix 变更是可操作的。
- 如果快照步骤失败,OpenClaw 会跳过该运行的 Matrix 迁移,而不是在没有恢复点的情况下改变状态。

关于多账户升级:

- 最旧的扁平 Matrix 存储(`~/.openclaw/matrix/bot-storage.json` 和 `~/.openclaw/matrix/crypto/`)来自单一存储布局,因此 OpenClaw 只能将其迁移到一个已解析的 Matrix 账户目标
- 已经是账户范围的旧版 Matrix 存储会被检测并按配置的 Matrix 账户准备

## 迁移无法自动执行的操作

以前的公共 Matrix 插件**没有**自动创建 Matrix 房间密钥备份。它持久化本地加密状态并请求设备验证,但它不能保证您的房间密钥已备份到 homeserver。

这意味着某些加密安装只能部分迁移。

OpenClaw 无法自动恢复:

- 从未备份的仅本地房间密钥
- 当目标 Matrix 账户尚无法解析时的加密状态,因为 `homeserver`、`userId` 或 `accessToken` 仍然不可用
- 当配置了多个 Matrix 账户但未设置 `channels.matrix.defaultAccount` 时,一个共享扁平 Matrix 存储的自动迁移
- 固定到 repo 路径而不是标准 Matrix 包的自定义插件路径安装
- 当旧存储有备份密钥但未在本地保留解密密钥时缺少恢复密钥

当前警告范围:

- Gateway 网关启动和 `openclaw doctor` 都会显示自定义 Matrix 插件路径安装

如果您的旧安装具有从未备份的仅本地加密历史记录,则升级后一些较旧的加密消息可能仍然无法读取。

## 推荐的升级流程

1. 正常更新 OpenClaw 和 Matrix 插件。
   优先使用不带 `--no-restart` 的普通 `openclaw update`,以便启动可以立即完成 Matrix 迁移。
2. 运行:

   ```bash
   openclaw doctor --fix
   ```

   如果 Matrix 有可操作的迁移工作,doctor 将首先创建或重用预迁移快照并打印归档路径。

3. 启动或重启 Gateway 网关。
4. 检查当前验证和备份状态:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. 如果 OpenClaw 告诉您需要恢复密钥,请运行:

   ```bash
   openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"
   ```

6. 如果此设备仍未验证,请运行:

   ```bash
   openclaw matrix verify device "<your-recovery-key>"
   ```

7. 如果您有意放弃不可恢复的旧历史记录并希望为将来的消息建立新的备份基线,请运行:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

8. 如果还没有服务器端密钥备份,请为将来的恢复创建一个:

   ```bash
   openclaw matrix verify bootstrap
   ```

## 加密迁移的工作原理

加密迁移是一个两阶段过程:

1. 如果加密迁移是可操作的,启动或 `openclaw doctor --fix` 会创建或重用预迁移快照。
2. 启动或 `openclaw doctor --fix` 通过活动的 Matrix 插件安装检查旧的 Matrix 加密存储。
3. 如果找到备份解密密钥,OpenClaw 会将其写入新的恢复密钥流程并将房间密钥恢复标记为待处理。
4. 在下一次 Matrix 启动时,OpenClaw 会自动将备份的房间密钥恢复到新的加密存储中。

如果旧存储报告从未备份的房间密钥,OpenClaw 会发出警告而不是假装恢复成功。

## 常见消息及其含义

### 升级和检测消息

`Matrix plugin upgraded in place.`

- 含义:旧的磁盘 Matrix 状态已被检测并迁移到当前布局。
- 要做什么:除非相同的输出也包含警告,否则无需执行任何操作。

`Matrix migration snapshot created before applying Matrix upgrades.`

- 含义:OpenClaw 在改变 Matrix 状态之前创建了一个恢复归档。
- 要做什么:保留打印的归档路径,直到您确认迁移成功。

`Matrix migration snapshot reused before applying Matrix upgrades.`

- 含义:OpenClaw 找到了现有的 Matrix 迁移快照标记并重用了该归档,而不是创建重复的备份。
- 要做什么:保留打印的归档路径,直到您确认迁移成功。

`Legacy Matrix state detected at ... but channels.matrix is not configured yet.`

- 含义:存在旧的 Matrix 状态,但 OpenClaw 无法将其映射到当前的 Matrix 账户,因为尚未配置 Matrix。
- 要做什么:配置 `channels.matrix`,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Legacy Matrix state detected at ... but the new account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- 含义:OpenClaw 找到了旧状态,但它仍然无法确定确切的当前账户/设备根。
- 要做什么:使用有效的 Matrix 登录启动一次 Gateway 网关,或在缓存凭证存在后重新运行 `openclaw doctor --fix`。

`Legacy Matrix state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- 含义:OpenClaw 找到了一个共享的扁平 Matrix 存储,但它拒绝猜测哪个命名的 Matrix 账户应该接收它。
- 要做什么:将 `channels.matrix.defaultAccount` 设置为预期的账户,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Matrix legacy sync store not migrated because the target already exists (...)`

- 含义:新的账户范围位置已经有一个同步或加密存储,因此 OpenClaw 没有自动覆盖它。
- 要做什么:在手动删除或移动冲突目标之前,验证当前账户是否正确。

`Failed migrating Matrix legacy sync store (...)` 或 `Failed migrating Matrix legacy crypto store (...)`

- 含义:OpenClaw 尝试移动旧的 Matrix 状态,但文件系统操作失败。
- 要做什么:检查文件系统权限和磁盘状态,然后重新运行 `openclaw doctor --fix`。

`Legacy Matrix encrypted state detected at ... but channels.matrix is not configured yet.`

- 含义:OpenClaw 找到了旧的加密 Matrix 存储,但没有当前的 Matrix 配置可以附加到它。
- 要做什么:配置 `channels.matrix`,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Legacy Matrix encrypted state detected at ... but the account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- 含义:加密存储存在,但 OpenClaw 无法安全地决定它属于哪个当前账户/设备。
- 要做什么:使用有效的 Matrix 登录启动一次 Gateway 网关,或在缓存凭证可用后重新运行 `openclaw doctor --fix`。

`Legacy Matrix encrypted state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- 含义:OpenClaw 找到了一个共享的扁平旧版加密存储,但它拒绝猜测哪个命名的 Matrix 账户应该接收它。
- 要做什么:将 `channels.matrix.defaultAccount` 设置为预期的账户,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Matrix migration warnings are present, but no on-disk Matrix mutation is actionable yet. No pre-migration snapshot was needed.`

- 含义:OpenClaw 检测到旧的 Matrix 状态,但迁移仍然因缺少身份或凭证数据而被阻止。
- 要做什么:完成 Matrix 登录或配置设置,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Legacy Matrix encrypted state was detected, but the Matrix plugin helper is unavailable. Install or repair @openclaw/matrix so OpenClaw can inspect the old rust crypto store before upgrading.`

- 含义:OpenClaw 找到了旧的加密 Matrix 状态,但无法从通常检查该存储的 Matrix 插件加载帮助程序入口点。
- 要做什么:重新安装或修复 Matrix 插件(`openclaw plugins install @openclaw/matrix`,或对于 repo checkout 使用 `openclaw plugins install ./path/to/local/matrix-plugin`),然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Matrix plugin helper path is unsafe: ... Reinstall @openclaw/matrix and try again.`

- 含义:OpenClaw 找到了一个帮助程序文件路径,该路径逃逸了插件根或未通过插件边界检查,因此它拒绝导入它。
- 要做什么:从可信路径重新安装 Matrix 插件,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`- Failed creating a Matrix migration snapshot before repair: ...`

`- Skipping Matrix migration changes for now. Resolve the snapshot failure, then rerun "openclaw doctor --fix".`

- 含义:OpenClaw 拒绝改变 Matrix 状态,因为它无法首先创建恢复快照。
- 要做什么:解决备份错误,然后重新运行 `openclaw doctor --fix` 或重启 Gateway 网关。

`Failed migrating legacy Matrix client storage: ...`

- 含义:Matrix 客户端侧后备找到了旧的扁平存储,但移动失败。OpenClaw 现在中止该后备,而不是静默地使用新存储开始。
- 要做什么:检查文件系统权限或冲突,保持旧状态完整,并在修复错误后重试。

`Matrix is installed from a custom path: ...`

- 含义:Matrix 固定到路径安装,因此主线更新不会自动将其替换为 repo 的标准 Matrix 包。
- 要做什么:当您想返回默认 Matrix 插件时,使用 `openclaw plugins install @openclaw/matrix` 重新安装。

### 加密状态恢复消息

`matrix: restored X/Y room key(s) from legacy encrypted-state backup`

- 含义:备份的房间密钥已成功恢复到新的加密存储中。
- 要做什么:通常无需执行任何操作。

`matrix: N legacy local-only room key(s) were never backed up and could not be restored automatically`

- 含义:一些旧的房间密钥仅存在于旧的本地存储中,从未上传到 Matrix 备份。
- 要做什么:期望一些旧的加密历史记录保持不可用,除非您可以从另一个已验证的客户端手动恢复这些密钥。

`Legacy Matrix encrypted state for account "..." has backed-up room keys, but no local backup decryption key was found. Ask the operator to run "openclaw matrix verify backup restore --recovery-key <key>" after upgrade if they have the recovery key.`

- 含义:备份存在,但 OpenClaw 无法自动恢复恢复密钥。
- 要做什么:运行 `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"`。

`Failed inspecting legacy Matrix encrypted state for account "..." (...): ...`

- 含义:OpenClaw 找到了旧的加密存储,但无法足够安全地检查它以准备恢复。
- 要做什么:重新运行 `openclaw doctor --fix`。如果它重复出现,保持旧状态目录完整并使用另一个已验证的 Matrix 客户端加上 `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"` 进行恢复。

`Legacy Matrix backup key was found for account "...", but .../recovery-key.json already contains a different recovery key. Leaving the existing file unchanged.`

- 含义:OpenClaw 检测到备份密钥冲突并拒绝自动覆盖当前的恢复密钥文件。
- 要做什么:在重试任何恢复命令之前验证哪个恢复密钥是正确的。

`Legacy Matrix encrypted state for account "..." cannot be fully converted automatically because the old rust crypto store does not expose all local room keys for export.`

- 含义:这是旧存储格式的硬限制。
- 要做什么:仍然可以恢复备份的密钥,但仅本地的加密历史记录可能仍然不可用。

`matrix: failed restoring room keys from legacy encrypted-state backup: ...`

- 含义:新插件尝试恢复但 Matrix 返回了一个错误。
- 要做什么:运行 `openclaw matrix verify backup status`,然后在需要时使用 `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"` 重试。

### 手动恢复消息

`Backup key is not loaded on this device. Run 'openclaw matrix verify backup restore' to load it and restore old room keys.`

- 含义:OpenClaw 知道您应该有一个备份密钥,但它在此设备上不活动。
- 要做什么:运行 `openclaw matrix verify backup restore`,或在需要时传递 `--recovery-key`。

`Store a recovery key with 'openclaw matrix verify device <key>', then run 'openclaw matrix verify backup restore'.`

- 含义:此设备当前没有存储恢复密钥。
- 要做什么:首先使用您的恢复密钥验证设备,然后恢复备份。

`Backup key mismatch on this device. Re-run 'openclaw matrix verify device <key>' with the matching recovery key.`

- 含义:存储的密钥与活动的 Matrix 备份不匹配。
- 要做什么:使用正确的密钥重新运行 `openclaw matrix verify device "<your-recovery-key>"`。

如果您接受丢失不可恢复的旧加密历史记录,您可以改为使用 `openclaw matrix verify backup reset --yes` 重置当前备份基线。

`Backup trust chain is not verified on this device. Re-run 'openclaw matrix verify device <key>'.`

- 含义:备份存在,但此设备还不足够强烈地信任交叉签名链。
- 要做什么:重新运行 `openclaw matrix verify device "<your-recovery-key>"`。

`Matrix recovery key is required`

- 含义:您在需要恢复密钥时尝试了恢复步骤而没有提供恢复密钥。
- 要做什么:使用您的恢复密钥重新运行命令。

`Invalid Matrix recovery key: ...`

- 含义:提供的密钥无法解析或与预期格式不匹配。
- 要做什么:使用 Matrix 客户端或恢复密钥文件中的确切恢复密钥重试。

`Matrix device is still unverified after applying recovery key. Verify your recovery key and ensure cross-signing is available.`

- 含义:密钥已应用,但设备仍无法完成验证。
- 要做什么:确认您使用了正确的密钥并且账户上有交叉签名可用,然后重试。

`Matrix key backup is not active on this device after loading from secret storage.`

- 含义:Secret storage 没有在此设备上生成活动的备份会话。
- 要做什么:首先验证设备,然后使用 `openclaw matrix verify backup status` 重新检查。

`Matrix crypto backend cannot load backup keys from secret storage. Verify this device with 'openclaw matrix verify device <key>' first.`

- 含义:在设备验证完成之前,此设备无法从 secret storage 恢复。
- 要做什么:首先运行 `openclaw matrix verify device "<your-recovery-key>"`。

### 自定义插件安装消息

`Matrix is installed from a custom path that no longer exists: ...`

- 含义:您的插件安装记录指向一个不再存在的本地路径。
- 要做什么:使用 `openclaw plugins install @openclaw/matrix` 重新安装,或者如果您从 repo checkout 运行,使用 `openclaw plugins install ./path/to/local/matrix-plugin`。

## 如果加密历史记录仍然没有回来

按顺序运行这些检查:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
openclaw matrix verify backup restore --recovery-key "<your-recovery-key>" --verbose
```

如果备份成功恢复但一些旧房间仍然缺少历史记录,那些缺失的密钥可能从未被以前的插件备份过。

## 如果您想为将来的消息重新开始

如果您接受丢失不可恢复的旧加密历史记录并且只想要一个干净的备份基线,请按顺序运行这些命令:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

如果此后设备仍未验证,请通过比较 SAS emoji 或十进制代码并确认它们匹配,从您的 Matrix 客户端完成验证。

## 相关页面

- [Matrix](/channels/matrix)
- [Doctor](/gateway/doctor)
- [迁移](/install/migrating)
- [插件](/tools/plugin)
