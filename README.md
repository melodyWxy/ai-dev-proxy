
你是一名资深平台工程 Agent。请为我的某个业务项目,从零搭建一套「需求迭代自动化工作台」(下称"工作台")。这套工作台的设计参考一套已验证的成熟范式(见下文"参考范式"),你要做的是**按同样的架构,为我的新项目生成一套等价的、配置驱动的工作台**,而不是照抄某个具体业务。

==================================================
第 0 步:先访谈我,收集搭建所需信息(不要假设,缺什么就问我)
==================================================
在动手前,用一组问题向我确认以下输入,并把答案落进配置:
1. 工作台仓库名与远端(工具仓,独立于业务代码)。
2. 项目元信息:name / domain(一句话业务域) / language(输出语言,默认中文) / timezone。
3. 业务代码形态:是「多仓」还是「monorepo 单仓多端」?各业务模块清单,每个模块给出:
   - key(短标识,如 admin/server/web/api)、显示名、远端 git 地址、(monorepo 时)子目录 subPath、角色描述;
   - 技术栈与运行时类型 type(node / go / python / static / miniprogram / 其它);
   - 本地启动命令 devCommand、依赖安装命令、包管理器与 lockfile、必需脚本(如 dev/build);
   - 本地 dev 访问方式:端口、是否需要自定义 hosts 域名、访问 URL 模板(用 {port} 占位)。
4. 运行时前置:需要的 node 版本/nvm、go 版本、python 等;哪些只能由我手动满足(VPN、私有源、账号、密钥、AppID 等外部条件)。
5. 发布(publish)策略:哪些模块支持发布、目标环境(test/prod 的 URL/健康检查)、是否需要人工确认。
6. 特殊端:如有小程序端,记录微信开发者工具 CLI 路径、AppID;如有其它特殊预览/验证方式一并说明
把没问清的项标记为 TODO,不要瞎填。

==================================================
参考范式:工作台的核心架构(请等价复刻)
==================================================
【双仓分离】
- 工具仓(本工作台):只放流程、检查脚本、配置、迭代记录、文档、模板、Cursor 接入。
- 业务仓:克隆/拆分到 `target-repos/<key>` 子目录副本,各自独立管理 Git(有自己的 origin、分支
- 工具仓 `.gitignore` 必须忽略 `target-repos/`;**严禁把业务代码提交进工具仓**;业务改动只在对应 `target-repos/<key>` 子仓内、按"单功能单 feature 分支单 commit"提交。

【配置驱动】一切行为由根目录 `automation.config.json` 描述,脚本读配置而非硬编码业务。给出 `docs/automation-config.schema.json`(JSON Schema)并让配置满足它。配置至少包含:
- `project`: { name, domain, language, timezone, primaryUsers, targetRepositories[] }
  - targetRepositories[i]: { key, name, url, subPath?, role }
- `targetRepos`: { localDir:"target-repos", businessDocsDir, overviewDoc, syncBeforeIteratiion[] }
- `records`: { iterationsDir:"iterations", activeIterationFileName:".active.md", iterationFileName:"iteration.md", activityFileName:"activity.md", statusValues[], requireActivityBeforeStage,
requireActivityAfterStage }
- `stages`:
  - init.autoRepair { agentFirst, installRuntimes, installProjectDependencies, maxAgentRepaor[] }
  - dev.branchPolicy { baseBranch, mustPullBaseBeforeFeatureBranch, instruction } + dev.localServicePolicy { ifBackendCheckWarning, defaultStartBackend, defaultStartFrontends, openBrowserAfterDev,
promptLoginForE2EAfterDev, e2eProofCommand }
  - test / commit / publish(publish: requiresUserConfirmation, required, supportedRepositories[], environments{})
- `checks`: { defaultScope, runtime{requireNvm,nvm,go,…}, git, cursor, business:{ repositor}
- 业务运行时配置(每个 key 一份),字段随 type 而定:
  - 通用:type、blockOnFailure(false 则该仓检查失败只 warning 不阻塞)
  - node:requiredNodeVersion、packageManager、lockFiles[]、requiredScripts[]、installCommand、devCommand、devHost{hostname,target,port,urlTemplate,openPath}
  - go:goMod、goSum、dependencyCommand、dependencyCheckCommand、blockOnFailure
  - miniprogram/static/python:requiredFiles[]、专用子配置(如小程序 preview{cliPath(按平台 map),qrImageName,qrSize,compileCondition})

【CLI:`commands/automation.mjs`(纯 Node ESM,无三方依赖)】
通过 `package.json` 的 scripts 暴露,命令分发用一个 `commands` 映射表。实现这些命令(行为对齐
- `check`(full/workspace/business 三个 scope):
  - 工作台检查:node/nvm/go 等运行时存在与版本、git、Cursor 配置、关键文档存在;
  - 业务检查:逐个 target 仓——目录/.git 存在、远端匹配、当前分支与 HEAD、运行时依赖(node_modules/go deps)、必需文件与脚本;
  - 结果汇总写入 `health-check` 报告;`blockOnFailure:false` 的失败记为 warning 不阻塞。
- `init`:按 init.autoRepair 自动修可处理项(装 nvm/node、go、各仓依赖),**只对外部条件(VPN/私有源/账号/密钥)才请求用户介入**,其余 Agent 自行排障。
- `sync-target-repos`:按 targetRepositories 克隆/校验各业务仓到 `target-repos/<key>`(monore
- `new-iteration -- "需求标题"`:在 `iterations/<时间戳-slug>/` 用模板生成迭代文档,并写 `.active.md` 设为当前迭代。
- `status` / `set-active` / `set-status -- --status <值> --note <原因>` / `complete` / `aba records.statusValues)。
- `activity`:向当前迭代 `activity.md` 追加阶段记录(每阶段开始/结束都记:Phase/Status/Note/时间)。
- `dev -- [--repos a,b | --<key>]`:读当前迭代→跑业务 check(有阻塞则中止)→按 localServicePol必要时写 hosts、探测端口、open 浏览器),后端有 warning时默认不启,小程序等特殊端走各自预览;产出 `dev-plan.md` 并记 activity;支持 `--plan`(只出计划不启动)、`--no-open`。
- `test` / `e2e-proof`:产出测试报告 / 端到端验证证据(截图等)。
- `publish`:按 stages.publish 与 supportedRepositories/environments 执行或仅产出发布计划(未就绪的端给出明确说明)。
- 复用的辅助层放 `commands/lib/`:`exec.mjs`(统一进程执行,永不抛异常,返回{ok,stdout,stderr}) 解析各 key 的磁盘路径/monorepo 映射)、`dev-server.mjs`(解析 dev 日志端口、拼访问URL)等;新增运行时类型时以"适配器"方式扩展,保持 type 可插拔。

【迭代生命周期与产物】目录 `iterations/<id>/`,核心产物用 `templates/*.md`
模板生成:iteration.md、activity.md、health-check.md、product-design.md、ux-design.md、techns.md、e2e-test-cases.md、test-report.md、publish-report.md、external-blockers.md等。生命周期:`check → init → new-iteration → 读业务文档 → dev(业务仓切 feature 分支)→ test → e2e-proof → publish → complete`。

【业务知识文档】`docs/`
下维护:business-domain-overview.md(业务域总览)、business-change-map.md(改动索引)、business- )、external-coordination.md(外部协作/阻塞)、getting-started.md。提供一个脚本(如`bootstrap-business`)用于扫描各业务仓刷新这些介绍文档。

【Agent/Cursor 接入】
- 根目录 `AGENTS.md` 与 `CLAUDE.md`:必读顺序、开工检查(先 `npm run check`)、业务路径与运行  业务码进工具仓、不读密钥/.env、中文输出、单功能单 commit、仅外部条件求助用户)。
- `.cursor/rules/<name>.mdc`(front-matter `alwaysApply: true`):列出"每次工作前必读"清单与开发规则、输出规范。
- `.cursor/commands/` 下提供 `check.md`/`init.md`/`start.md`/`dev.md`/`publish.md` 等斜杠命

【护栏(务必内建到脚本与文档)】
- 不把业务代码提交进工具仓;`target-repos/` 被忽略。
- 不读取/输出任何密钥与 `*/.env`;忽略本地数据目录(如 `data/`)。
- 业务仓在各自 `target-repos/<key>` 内提交;默认从 baseBranch 切 feature 分支。
- 仅 VPN/私有源/账号/密钥/AppID 等外部条件才请求用户;其余 Agent 自动排障。
- 对用户输出用配置里的 language(默认中文),并始终记录路径、命令与验证结果。

==================================================
验收标准
==================================================
1. 全新克隆后 `npm install`(若有依赖)+ `npm run check` 能跑通并产出健康检查报告。
2. `automation.config.json` 满足 `docs/automation-config.schema.json`,且只含我的新项目信息(无任何参考范式里的残留业务名)。
3. 提供 `test:workspace`(node:test)自检:校验 package.json 中指向 automation.mjs 的脚本都已  出等。
4. `new-iteration` 能创建迭代并被 `status` 正确列出;`dev --plan` 能对各业务端产出计划而不启动真实服务。
5. README 写清架构、命令表、各端路径/端口/远端、文档索引。

请先进入第 0 步向我提问;待我确认输入后再开始生成文件,并在每个阶段告诉我:
产出了哪些文件；
如何验证（具体命令与预期结果）；
是否有 TODO 或需我手动满足的外部条件。

