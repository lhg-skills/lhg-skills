# Skill 分发方法论（lhg-skills 统一规范）

> 适用范围：lhg-skills 组织下所有 Skill 与需要「一键安装」的小项目。
> 目标：让**不懂技术的用户只做一件事——复制一段话发给自己的 Agent**，剩下的下载、安装、验证、教学都由 Agent 完成。

---

## 一、核心原则

1. **按最差环境设计**：假设用户打不开 GitHub、Agent 会拦截 `npx`、会在微信里乱点链接、分不清「打开网页」和「装好了」。
2. **Agent 代办优先于人工教程**：分发的是一段「发给 Agent 的指令」，不是「教用户操作的步骤」。
3. **一份话术适配所有 Agent**：用「按当前 Agent 的规范安装」把平台差异交给 Agent 自己处理，不为每个平台单独写教程。
4. **安装 → 验证 → 教学 → 报错，闭环写全**：装完必须自检、告诉用户怎么用；失败必须说清具体原因。
5. **双源分发：Gitee 优先，GitHub 兜底**：GitHub 是唯一源头（source of truth），Gitee 是自动同步的镜像。
6. **人机分工写死**：密码、付费、删除等操作必须停下来等用户亲手完成，写进 SKILL.md。

---

## 二、双源分发架构

| 角色 | 平台 | 说明 |
|---|---|---|
| 源头仓库 | GitHub `lhg-skills/<slug>` | 所有修改只在这里提交 |
| 国内镜像 | Gitee `lhg-skills/<slug>`（组织名、仓库名与 GitHub 保持一致） | 由 GitHub Actions 自动推送，**禁止直接在 Gitee 改代码** |

**为什么需要 Gitee**：Gitee 不是必需品，它只服务「打不开 GitHub」的用户。如果你的用户全都能访问 GitHub，可以只用 GitHub。

**注意事项**
- Gitee 公开仓库可能需要实名认证与内容审核，首次建仓预留时间。
- 两边路径必须完全一致（同组织名 + 同仓库名），话术才能用一句「同名仓库」覆盖两个来源。
- 若 Gitee 上 `lhg-skills` 组织名被占用，选定一个替代名后在本文件「附录：来源登记」里记录，并同步修改话术模板。

### 自动同步（每个仓库放一份）

路径：`.github/workflows/sync-to-gitee.yml`

```yaml
name: Sync to Gitee
on:
  push:
    branches: [main]
    tags: ['*']
  workflow_dispatch:
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Push to Gitee
        env:
          GITEE_USER: ${{ secrets.GITEE_USER }}
          GITEE_TOKEN: ${{ secrets.GITEE_TOKEN }}
        run: |
          REPO="${GITHUB_REPOSITORY#*/}"
          REMOTE="https://${GITEE_USER}:${GITEE_TOKEN}@gitee.com/lhg-skills/${REPO}.git"
          git push --force "$REMOTE" HEAD:main
          git push --force --tags "$REMOTE"
```

- `GITEE_USER` / `GITEE_TOKEN` 建议设为 **GitHub 组织级 Secrets**，所有仓库共用。
- 令牌只给 Gitee 仓库写权限，不要放进任何文件。

---

## 三、标准安装话术（对外分发唯一版本）

### 模板

> 请帮我安装 Skill「{slug}」：先从 Gitee 上 lhg-skills 组织下的 {slug} 仓库下载；如果 Gitee 无法访问或下载失败，改从 GitHub 上 lhg-skills 组织下的同名仓库下载。按当前 Agent 的规范安装其中的 Skill，优先全局安装，保留完整目录和依赖。请直接执行，完成后验证是否安装成功，告诉我实际使用的来源、安装的版本号以及如何调用；遇到访问或安装限制请说明具体原因。

### 每个短语的作用（改模板前先读）

| 短语 | 防的是什么坑 |
|---|---|
| 用文字描述仓库位置，不写网址 | 微信会把网址变成可点击链接，用户点开就以为装好了 |
| 不写 `npx` 命令 | 部分 Agent（如 WorkBuddy、豆包）可能因安全策略拦截 `npx` |
| 先 Gitee，失败再 GitHub | 国内网络优先可达；Gitee 未同步或下架时自动兜底 |
| 按当前 Agent 的规范安装 | 各 Agent 的 Skill 目录与安装方式不同，交给 Agent 自己判断 |
| 优先全局安装 | 避免只装在当前项目，换个会话就找不到 |
| 保留完整目录和依赖 | 防止只复制 SKILL.md，漏掉脚本、模板、references |
| 请直接执行 | 防止 Agent 把命令丢回给用户手动跑 |
| 验证是否安装成功 | 区分「网页能打开」和「Skill 已安装」 |
| 告诉我来源和版本号 | 方便排查是不是 Gitee 镜像落后 |
| 如何调用 | 省掉「装好了然后呢」的追问 |
| 说明具体原因 | 用户转述的是可定位的报错，不是「装不上」 |

### 话术使用规则

- 分发渠道（微信、视频号、社群）**只发这段话术**，不附链接。
- 有特殊前置条件的 Skill（例如需要管理员密码、需要先退出某软件），在话术末尾追加一句：「需要输入密码或确认风险时停下来等我。」
- 话术中的 `{slug}` 必须与仓库名、`manifest.yaml` 的 `slug`、`SKILL.md` 的 `name` 完全一致。

---

## 四、仓库标准结构

```
<slug>/
├── SKILL.md            # 必需：Agent 执行入口（frontmatter 含 name、description）
├── manifest.yaml       # 必需：slug / display_name / description / version / author / license
├── README.md           # 必需：按下方固定章节顺序
├── LICENSE             # 必需
├── references/         # 可选：方法论、长文档
├── scripts/            # 可选：可执行脚本
├── smoke-test.*        # 推荐：安装后自检脚本
└── .github/workflows/sync-to-gitee.yml  # 必需：镜像同步
```

### README 固定章节顺序

1. **一句话说明**：这个 Skill 解决什么问题。
2. **发给 Agent 的安装话术**：直接粘贴第三节模板（已替换 slug）。
3. **开发者安装**（给能访问 GitHub、会用命令行的人）：`npx skills add lhg-skills/{slug}` 或 `git clone`。放在话术之后，不作为主路径。
4. **如何验证安装成功**：Agent 应能看到哪个 Skill 名、或运行哪个自检命令。
5. **如何调用**：2～3 个可以直接说给 Agent 的示例句子。
6. **常见失败与原因**：见第六节。
7. **卸载 / 还原**。
8. **反馈**：issue 需附带的信息（Agent 名称、来源、版本号、报错原文）。

### SKILL.md 必含内容

- 前置环境检查（系统、依赖、权限）。
- 人机分工：哪些步骤必须用户亲手做（密码、付费、删除、不可逆操作）。
- 完成判定：通过哪些检查才允许向用户报「完成」；失败时先还原，再如实汇报。

---

## 五、版本与发布流程

1. 在 GitHub 修改 → 更新 `manifest.yaml` 的 `version`（语义化版本）。
2. 打 tag `v{version}` 并推送 → Actions 自动同步到 Gitee。
3. 确认 Gitee 仓库的 `manifest.yaml` 版本与 GitHub 一致。
4. 在一台**无法访问 GitHub** 的环境，用至少一个国产 Agent（如豆包或 WorkBuddy）跑一遍标准话术。
5. 在一台能访问 GitHub 的环境，用 Claude Code / Codex / Cursor 之一跑一遍标准话术。
6. 两边都能装上、能验证、能调用，才对外分发。

---

## 六、常见失败与应对

| 现象 | 原因 | 应对 |
|---|---|---|
| 用户说「点开了，装好了吧？」 | 话术里出现了链接 | 删除链接，只保留文字描述 |
| Agent 报「命令被拦截」 | 使用了 `npx` 或远程脚本管道执行 | 话术不写命令，让 Agent 用自身的下载方式 |
| Gitee 和 GitHub 都失败 | 网络受限或 Agent 无联网能力 | 提供 zip 离线包，话术改为「安装我发给你的压缩包」 |
| 装上了但调用不到 | 装在项目级目录，或目录名与 Skill 名不一致 | 强调「优先全局安装」，统一 slug |
| 装上了但运行报错 | 只复制了 SKILL.md，缺脚本或依赖 | 强调「保留完整目录和依赖」，加冒烟测试 |
| 用户装到旧版本 | Gitee 镜像同步失败 | 检查 Actions 日志；话术要求 Agent 报告版本号 |

---

## 七、安全边界

「下载并直接执行安装」意味着用户信任这个来源。因此：

- 只分发自己维护的仓库；引用第三方代码必须在 `THIRD-PARTY-NOTICES.md` 声明来源与许可证。
- 不在仓库中放任何令牌、密钥；发布前运行一次密钥扫描。
- 不可逆或高风险操作必须在 SKILL.md 中写明「停下来等用户确认」。
- 鼓励用户在安装第三方 Skill 前先用 `lhg-finder` 做质检。

---

## 八、迁移检查清单（现有仓库逐个对照）

- [ ] Gitee 已建同名镜像，`sync-to-gitee.yml` 已添加并跑通
- [ ] README 第一段换成标准安装话术（Gitee 优先），`npx` 移到「开发者安装」
- [ ] README 中「发给 Agent」的句子不含网址
- [ ] 仓库描述（About）不再以 `npx` 作为唯一安装方式
- [ ] `manifest.yaml` 含 `version`，与 `SKILL.md` 的 `name`、仓库名一致
- [ ] 有「如何验证」「如何调用」「常见失败」三个章节
- [ ] 关键 Skill 有冒烟测试

---

## 附录：来源登记

| 平台 | 组织 | 备注 |
|---|---|---|
| GitHub | lhg-skills | 源头 |
| Gitee | lhg-skills | 已创建（2026-10-15），16 个仓库已导入；同步账号 `GITEE_USER` = kanshan28 |
