# 009 多项目：方案

本文回答 issue #7：当前原型只配置一个 GitHub 仓库，那么应当支持配置多个项目，还是支持智能体 GitHub 账户下的所有项目？本文建立在 [001 领域模型](001-domain.md)、[002 授权](002-authorization.md)、[003 执行](003-execution.md) 和 [007 架构](007-architecture.md) 之上。

这是一份设计分析，不是当前构建的说明：[006 部署](006-deployment.md) 描述的仍是现在发布的单仓库行为，本文没有任何部分已经实现。

结论先说：支持**一份显式的仓库清单**，用 `github` 来源类型的多个实例来实现。"账户下的所有项目"不是一个独立方案；它唯一安全的形式就是同一份显式清单，只是可以由建议列表填充。代价不在 GitHub API，而在身份：今天 Issue、Thread、读取位置、回复和 pull request 全部只用一个编号命名，而两个仓库都有 issue #12。

## 今天"单项目"意味着什么

`Deployment` 里只有一个 `repository` 和一个 GitHub 账号（`src/deployment.ts`），下游全部读这个标量：

- `Commands` 给每个 API 路径加上 `repos/<repository>/` 前缀，并拒绝其他路径（`src/infra/commands.ts`）。这条限制是防御提示注入的护栏，不是操作系统边界，但它是唯一把智能体的 GitHub 写入限制在项目内的东西。
- GitHub 来源拿 `notification.repository.full_name` 和它比较，用 `githubAddress(url, repository)` 过滤 subject URL，并且只轮询这个仓库的仓库级数据流（`src/sources/github.ts`）。
- 执行环境把它暴露为 `DSH_SUPERAGENT_GITHUB_REPOSITORY`，文档中的 prepare 命令据此克隆（`docs/006-deployment.md`）；pull request 的 head 是那个仓库的 `refs/pull/<number>/head`。
- 回复地址只带 `{ number, url }`，不带仓库；`sendGithub` 从部署配置重新推导仓库（`src/sources/github-channel.ts`）。
- Setup 只校验这一个仓库、它的推送权限，并为账号写一个 `gh` shim 和一个 git credential helper（`src/deployment.ts`、`src/infra/environment.ts`）。
- 面板只显示一行仓库，`SETUP_FIELDS` 按来源类型列出影响启动检查的字段（`src/app/views/settings.ts`、`src/deployment.ts`）。

来源*类型*（kind）在文档中本就是可复用的实现，来源*实例*（instance）是它的某一份配置（[001 领域模型](001-domain.md)）。实现把两者合并了：`SourceRegistry` 用类型 id 作为存储实例的键，`resolve` 返回 `{ id: kind.kind, kind: kind.kind, … }`（`src/sources/registry.ts`）。于是一个类型就是一个实例、一张卡片、一份 Policy 片段、一个健康项和一条轮询循环。

## issue #7 的两个问题

**配置多个项目。** 由运维者指定智能体在哪些项目里工作。今天这只能靠多个部署实现：每个项目一个 profile、状态目录、面板和服务进程（`docs/006-deployment.md`）。它完全隔离、不需要改代码，代价是每个项目一个进程、一个数据库和一份定时检查。

**账户下的所有项目。** GitHub 在这个名字下提供的是两件不同的东西，而两者不加额外规则都不是"所有项目"：

- 账户通知流 `GET /notifications`，报告账号参与或关注（watch）的线程：提到、自己创建、评论、请求评审，以及关注仓库里的全部通知（`reason: subscribed`）。它**不会**报告一个账号有推送权限、但既未关注也未被提到的仓库里的新 Issue 或评论。GitHub 文档标注这些接口仅支持 classic personal access token；2026-11-23 抽查时智能体的 `gh` OAuth token 能拿到结果，但在依赖这条流之前必须用实际凭据再验证一次。
- 仓库级观察 `GET /repos/{owner}/{repo}/…`，报告一个仓库内的全部事件，但只限于有人枚举出来的仓库。`GET /user/repos` 可以枚举，但枚举不等于准入：新建的、被转移的、或误授权的仓库都会无声地进入智能体的范围。

当前来源已经在用仓库级通知接口，因为 `Commands` 会把 `notifications` 改写成 `repos/<repository>/notifications`。改用账户流会扩大"收进来什么"，却不改变"被授权做什么"，而且它需要第二个显式的账户级调用：如果放开 `Commands` 让无前缀路径通过，就会一次性移除所有调用方的仓库限制。

## API 成本

智能体账号每小时有 5,000 次 core 请求，与评估、回复、pull request 写入和每个执行器 session 的 `gh` 共用。返回 `304 Not Modified` 的条件请求不计入这个额度。

默认 60 秒轮询下，一个空闲仓库跑一轮当前轮询器的成本如下：

| 读取 | 请求数 | 现在是否条件请求 |
| --- | --- | --- |
| 通知 | 1 | 否 |
| 新 Issue 与 pull request | 1 | 是，游标不动时 |
| 会话评论 | 1 | 同上 |
| 行内 review 评论 | 1 | 同上 |
| 默认分支 CI | 4 | 否 |
| 一个开启的 pull request 及其 CI | 3 | 否 |
| 两个订阅的 Thread | 2–8 | 否 |

其中三条仓库级数据流在无变化时能返回 `304`；通知调用和 CI 读取永远计数。因此一个完全观察的空闲项目每轮约 5 次请求，有 pull request 打开时再多 3 次，即每小时 300–480 次。把轮询控制在 5,000 额度三分之一以内，只够三到五个完全观察的项目；只轮询通知则约 25 个。

最划算的改进与是否做多项目无关：通知调用目前不发 `If-None-Match` 或 `If-Modified-Since`，而 GitHub 文档明确这个接口适合条件轮询。2026-11-23 对线上 API 实测：带已存 ETag 的请求返回 `304 Not Modified` 且 `x-ratelimit-remaining` 不变，而不带条件的调用会减一；仓库级和账户级两个接口都是如此。代码本来就在读响应头拿 `X-Poll-Interval`，所以存下 ETag、把 `304` 当作"没有变化"只是单个调用点的小改动。

## 成本所在：身份

Issue 和 pull request 编号在每个仓库都从 1 开始。原型持久化的每一个名字都只由编号推导：

| 只用编号命名 | 两个仓库时的结果 |
| --- | --- |
| `ThreadRef` 与 `threadKey` = `github:<number>`（`src/core/model.ts`、`src/sources/github-trigger.ts`） | 两个仓库的 issue #12 是同一行 Thread，于是 B 仓库的评论（其 Trigger id 带有仓库）落在 A 仓库的 Thread 上，扇出到 A 仓库的工单 |
| Issue Trigger id `key('github-issue', String(number))`，pull request 相同 | 第二个仓库的 Issue 被 `Intake.receive` 判为 `duplicate` 丢弃 |
| 默认分支 CI Thread `ci:<branch>` 与 `key('github-ci', 'branch:main', …)` | 两个 `main` 共用一个 Thread 和 Trigger id |
| pull request CI `key('github-ci', number, revision)` 与 `KV.github.ci(number)` | 一个仓库的 CI 覆盖另一个的指纹，Trigger id 相撞 |
| `KV.github.since`、`retries`（按编号）、`cursor(number)`、`stream(id)`、`branchCi`（`src/kv.ts`） | 读取位置按最后轮询的仓库前进 |
| 渠道地址 `{ number, url }` 与 `sendGithub` 从部署取仓库 | 给 B 仓库 #12 的回复被发到 A 仓库 #12 |
| `pullsOf`/`pullOf`、`claimPull`（比较 `meta.number`）、`observePull`（分支最后一段） | B 仓库的 pull request 可能绑定到 A 仓库的工单，"每个工单只跟一个开启的 PR"的检查跨越两个仓库 |
| `find` 把任意 `#12` 或任意仓库的 URL 映射为 `github:12` | 搜索解析到错误的 Thread |
| `KV.sources.labels(source)` 与 `source:<id>` 设置块 | 两个仓库共用一份标签映射和一张卡片 |

表里还能推出两点。仓库必须进入 Trigger 的 `meta`，因为 pull request 相关路由、Thread 上下文读取器和执行环境在接收之后都需要它。仓库还必须进入身份，而不只是 `meta`，因为 `Thread` 和 `Trigger` 行按它作键。

## 方案 A：配置多个项目

**A1，多个部署。** 今天就能用：每个项目一份状态目录、profile 和面板。完全隔离、无需改代码；运维者付出每项目一个进程、一份定时检查和一块面板。把它当作诚实的基线，也当作需要独立账号或独立系统用户的项目的答案。

**A2，同一来源类型的多个实例。** 一个进程、一块面板、一份策略；每个仓库一个 `github` 实例。这就是领域模型已经描述的形态，也是本文其余部分假定的方案。它需要：

1. **实例身份。** 存储的实例带上自己的类型；某类型的第一个实例保持 id 为 `github`，于是所有已持久化的 `source` 字符串、Thread 键、Policy 片段、健康项名和 KV 键继续有效。后续实例取 `github-widgets` 这类 id，`Trigger.source` 变成实例 id。
2. **带仓库的身份。** Issue 与 Trigger id 带上仓库（或实例 id）；Thread 引用同理；`githubAddress` 与 `sendGithub` 从 Trigger 的渠道地址取仓库；`find` 通过 URL 所属仓库的实例解析。
3. **按实例的部署。** 每个实例一份解析后的部署（仓库、账号、推送权限），一个绑定它的 `Commands`，以及取值来自工单所属实例的执行变量。变量名保持不变，文档中的 prepare 命令对每个仓库继续可用。
4. **按实例的轮询与健康。** 每个实例一条轮询循环，各自拥有游标、连续失败计数、`X-Poll-Interval` 和健康项；一个仓库的失败不能拖住另一个。
5. **按实例的策略。** 允许名单、机器人名单和准入表挂在实例下，而不是类型下。在某个仓库里列出来的人不会因此获得另一个仓库的授权，这正是 [002 授权](002-authorization.md) 已经隐含的规则：Mandate 记录它的来源，来源之间从不共享身份。
6. **设置与面板。** 每个实例一张来源卡片，提供一个指明类型的"添加"操作，并按实例报告启动检查问题。

**A3，一个实例持有仓库列表。** 这省掉了实例管道，却省不掉身份工作：列表里每一项仍然需要自己的 Thread 命名空间、读取位置、健康状态和准入行，否则就是上面那些冲突。它还让一条轮询循环服务多个仓库，一个慢的或坏的仓库会拖住其余仓库。同样的代价，更差的形态。

## 方案 B：账户下的所有项目

不应按字面实现 B。它唯一的安全属性只能来自隐式范围：一份凭据、一份按来源记录的允许名单，以及以运维者系统用户身份、带 `danger-full-access` 运行的工作进程（`docs/006-deployment.md`）。[000 理念](000-philosophy.md) 要求从小处开始、有意地扩大；隐式的"这个账号能推送的任意仓库"与之相反。具体的风险正是范围错误会带来的：在运维者没打算的仓库里干活、把回复发到错误的仓库、或对着 fork 开 pull request。

B 里有用的部分是发现，它可以叠在 A2 上：

- `GET /user/repos` 提示账号能到达的仓库，面板把它们列出来供选择。
- 只有显式启用的实例才轮询。被改名、归档、删除或已无推送权限的仓库不再被准入，并写进该实例的健康项，而不是永远重试。
- 准入与否的判断使用仓库的数字 id，这样改名不会被当成新项目。
- 如果使用账户通知流，它只对已准入的仓库产生 Trigger；其余记录为已过滤，不进入评估。

在这条规则下，B 就变成"A2 加一个建议列表"，可以在 A2 之后追加，而不用再改一次身份。

## 建议

按下面的顺序实现 A2。每一步都让原型保持可用，前三步不改变行为。

1. **不改变行为的实例身份。** 给存储实例加 `kind`，并在 registry、设置视图、历史记录和 Policy 查找中按 `实例 → 类型` 解析。迁移写回该字段；`github` 实例保留今天的全部 id、Thread 键、健康项名和 KV 键。
2. **按实例的部署与执行环境。** 每个实例解析一份部署，把 `Commands`、启动检查问题和执行变量取值绑定到工单所属实例。只配置一个仓库时行为与现在完全一致。
3. **带仓库的身份。** 把仓库放进 Trigger 的 `meta`、Issue/PR/CI 的 Trigger id、Thread 引用、渠道地址和 PR 绑定路由（`pullsOf`、`claimPull`、`observePull`、`pullEndNotices`），并为已有的 `github:<number>` 行写迁移。首个实例的名字保持逐字节一致，历史不会被重新接收。
4. **按实例的轮询与健康。** 每个配置实例一条循环，按实例的 KV 键、失败计数和健康项。
5. **设置、API 与面板。** 为已安装类型新增一个来源实例、删除一个实例，并显示每个实例的标题、启动检查问题、计数和 Policy 片段。
6. **可选的发现。** 把 `GET /user/repos` 作为建议提供，由运维者启用；绝不轮询未准入的仓库。

在第 2 步和第 3 步之间，把通知调用改成条件请求。它只改一个调用点，实测无变化时不消耗额度，而且无论是否真的启用多项目都值得做。

两条边界要写明。多 GitHub 账号不在范围内：一个 `gh` shim 和一个 credential helper 服务智能体账号，按账号分实例还需要按实例的包装脚本。需要独立账号、系统用户或调度的项目使用 A1，即独立部署，本方案不替代它。

## 每个阶段必须守住

[001 领域模型](001-domain.md) 已经写明的规则，在多做项目时最容易破坏，重述如下：

- 一个 Thread、一个 Trigger 和一个 Channel 属于同一个来源实例；丢掉实例或仓库的 id 即使今天恰好唯一也是错的。
- 准入与允许名单按实例生效。在某个仓库里接单，不会给另一个仓库任何授权。
- 删除一个实例会停止它的轮询，但存储里仍保留它的 Thread、工单、排队中的 Trigger 和 Mandate。要显式决定删除如何处理它们；registry 的 `scope` 对既没有已安装类型、也没有存储实例的来源字符串回答 `listened: true`，所以删除不能依赖准入把该来源已存的 Trigger 挡掉。
- 工单的执行环境描述的是该工单的仓库，而不是部署的仓库。
- 某个仓库出错只报在它自己的实例上，不拖住其他仓库。

## 验证

测试里已有的 `gh` 桩按仓库路由（`tests/sources.spec.ts`），所以这些阶段不需要连线上 GitHub 就能验证：

- 两个仓库使用相同 issue 编号时产生不同的 Trigger id 和 Thread 键，且都被接收。
- 对其中一个仓库 Thread 的回复，走的是该仓库下的路径；每个已配置仓库之外的路径被拒绝。
- 条件通知请求返回 `304` 并推进游标，不计一次请求。
- 各仓库游标独立前进；某个实例返回 404 时另一个仍为就绪。
- 同一登录在一个实例里被允许，在另一个实例里是 `not_a_member`；在那里被接受的 Mandate 不会被另一个实例的名单重新验证。
- 第二个实例的工单拿到该仓库的 `DSH_SUPERAGENT_GITHUB_REPOSITORY`，且只绑定本仓库中同编号的 pull request。
- 从 schema 11 迁移来的数据库保留首个实例的全部 Thread、Trigger、工单和读取位置不变。
