# renovate-config

所有项目共用的 Renovate 配置：统一周更、单一常规升级组、通过检查后自动合并。

## 接入项目

确保 Renovate GitHub App 已获得目标仓库的访问权限，在目标仓库添加 `renovate.json`：

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>mrgeneralgoo/renovate-config"]
}
```

未指定 Git ref 时，Renovate 从本仓库默认分支读取 `default.json`。
共享配置合并到默认分支后，各项目会在 Renovate 后续运行时读取新规则；不是立即触发所有项目的 CI。
本仓库首次发布前，上面的远端引用不可用。

迁移已有项目时，清理旧的调度、分组和宽泛自动合并规则。
本地配置可以覆盖共享设置，`packageRules` 会合并，因此仅追加 `extends` 不会自动移除旧规则。
保留必要的兼容性例外，例如停更依赖、必须人工升级的镜像。
新增项目仍需主动接入；创建共享仓库不会自动接管整个账号。

## 统一策略

| 项目 | 行为 |
| --- | --- |
| 维护窗口 | 每周一全天，时区为 `Asia/Shanghai` |
| 普通更新 | 使用内置 `group:all`，所有生态共用一个升级组，不拆分 major/minor 组 |
| 窗口之外 | 不自动创建或刷新普通升级分支 |
| 冷却期 | 有发布时间的普通版本等待 7 天；严格过滤尚未成熟的版本 |
| 常规 PR 限制 | 每仓库最多 1 个开放 PR，每小时最多创建 1 个 |
| 自动合并 | minor、patch、pin、pinDigest、digest 更新可以自动合并；任何当前版本为 0.x 的依赖除外 |
| 人工合并 | major 和未明确允许自动合并的更新类型；同批任一更新不允许自动合并，整批转人工 |
| 合并方式 | Renovate 等待检查通过后 squash 合并 PR，不依赖 GitHub 原生 auto-merge |
| 锁文件 | 普通升级照常同步锁文件，不额外创建周期性全量刷新 PR |
| 安全修复 | 不受普通维护窗口、冷却期和普通 PR 限额约束，单独创建修复 PR，人工确认 |

维护窗口是允许更新的时间段，不是定时启动 Renovate 的任务。
托管 App 在窗口内可能多次处理仓库，因此不是严格保证每周只扫描一次或只创建一个 PR。
合并时间没有单独限制。窗口外如果需要 rebase，普通 PR 可能等到下个维护窗口。

没有发布时间的数据源不会永久阻塞，因此冷却期不是所有包及间接依赖的绝对保证。
Docker 浮动标签的 digest 也可能包含破坏性变化；没有可靠功能检查的镜像应在项目中禁止自动合并。
同批出现失败检查或需人工处理的更新时，接受整批等待，不再自动拆分成多个维护组。

## 合并安全与 CI

- 接入自动合并前，目标仓库必须有真实的 PR 验证，并允许 squash 合并。
- 不设置 `ignoreTests: true`，也不把 Renovate 内部检查当作唯一成功信号。
- 默认保留自动 rebase；不通过禁用分支同步来节约 CI。
- GitHub 套餐支持时，应启用分支保护并配置必需检查。Renovate 自己检查后合并不等同于平台强制门禁。
- 给 PR 检查设置合理超时和取消旧运行，避免重复执行同一阶段的检查；不要盲目取消生产发布。
- 安全修复依赖 GitHub Dependency graph、Dependabot alerts 和 Renovate 的告警读取权限，不等于完整容器漏洞扫描。

共享配置只管理 Renovate，不修改各项目的工作流、密钥、仓库权限或部署方式。

## 验证配置

使用 Node.js 26：

当前锁定的 Renovate 仅声明支持 Node.js 24，尚未声明支持 Node.js 26；安装时会出现 engines 兼容性警告。这里不修改上游声明或关闭兼容性检查。

```sh
npm ci --no-audit --no-fund
npm test
```

验证器版本和依赖树由 `package.json`、`package-lock.json` 固定，由本仓库自己的 Renovate 配置继续管理。
CI 使用相同命令校验配置，固定第三方 Action 的完整提交 SHA，且只有仓库内容读取权限。
CI 仅在 PR 和主分支 push 时运行，不添加定时工作流。

验证器检查配置语法和选项，不证明每个消费项目的测试覆盖、权限和实际升级结果。
首次接入仍需检查 Renovate 的配置解析结果和真实 PR 检查。

## 参考

借鉴 [Hatena renovate-config](https://github.com/hatena/renovate-config) 的共享 preset、仓库自身引用和配置校验方式。
不照搬其多生态分组、业务时间窗口或旧版配置字段。

- [Renovate 共享 preset](https://docs.renovatebot.com/config-presets/)
- [Renovate 配置选项](https://docs.renovatebot.com/configuration-options/)
- [Renovate 自动合并](https://docs.renovatebot.com/key-concepts/automerge/)
