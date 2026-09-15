# dsh-pwsh-survival-guide

在 Windows 上写 PowerShell 命令时最常踩的坑，以及正确写法。

内容不是从文档抄的，而是从**真实翻车记录**里总结的：一个 AI agent 在 Windows 上连续几轮执行命令，反复因为引号转义、JSON 校验、编码 BOM、管道语义而失败，最后把教训固化成这份清单。

## 它解决什么问题

如果你（或你的 AI agent）在 Windows 上跑 PowerShell，下面这些场景大概率遇到过：

| 症状 | 真实原因 |
| --- | --- |
| `Expected ',', got '<eof>'` / `Invalid or unexpected token` | `node -e "..."` 里的引号被 PowerShell 吃掉 |
| `ConvertFrom-Json : Invalid object passed in, ':' or '}' expected. (16641)` | 老版 PowerShell 对**合法** JSON 误报 |
| `Windows PowerShell is in NonInteractive mode` | `Invoke-WebRequest` 缺 `-UseBasicParsing`，它在试图调 IE 引擎 |
| `JSON.parse` 报错 / git diff 出现鬼影首行 | `Set-Content -Encoding UTF8` 写入了 BOM |
| 命令明明成功，却跟着 `[exit code: 1]` 和一片红字 | `Select-Object -First N` 提前关管道，或 git 把进度写到 stderr |
| `fatal: ... Recv failure: Connection was reset` | git 不读系统代理 |
| 统计字符串出现次数得到几万 | `String.Split("abc")` 是按**字符**拆分的 |

## 适用环境

清单里有些结论是**环境特定**的，换机器时先自查一下：

| 清单里提到的 | 怎么确认你自己的 |
| --- | --- |
| 代理在 `127.0.0.1:6789`（clash） | `Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' \| Select-Object ProxyEnable, ProxyServer` |
| 没有 `gh` / `rg` | `Get-Command gh, rg -ErrorAction SilentlyContinue` |
| PowerShell 版本行为不一致 | 看 `$PSVersionTable.PSVersion`，再按第 10 条的方式试 |
| Node v24、自带 zstd | `node -v` |

而**引号、BOM、管道截断、退出码**这几类问题与具体机器无关，放哪都适用。

## 目录

- **[SKILL.md](SKILL.md)** —— 清单本体，17 个主题，每条都给出「错误写法 → 正确写法」

主题列表：

1. 引号：`node -e "..."` 是陷阱
2. JSON 校验：`ConvertFrom-Json` 会误报
3. HTTP：`Invoke-WebRequest` 必须加 `-UseBasicParsing`
4. 编码与 BOM
5. 管道截断造成的"假失败"
6. 统计字符串出现次数
7. 环境是「一次性」的
8. 代理
9. 命令是否存在，先探测
10. 版本行为差异
11. 其他本机要点（目录联接、zstd 多帧、含中文的 HTTP body 等）
12. 路径：`-Path` 会把它当通配符
13. 退出码有两套系统，别混用
14. 紧凑参数传给原生程序可能被拆开
15. Unix 工具不存在，或用的是「同名不同物」
16. `&&` / `||` 在 PowerShell 5.1 里是语法错误
17. 执行策略只对当次生效

SKILL.md 开头还有一节 **「什么时候不用翻这份清单」** —— 明确列出不该触发本技能的场景，避免给日常命令增加"行为税"。

## 当作 DSH skill 使用

DSH（Deepseek Harness）的技能格式是 `<skills-dir>/<kebab-name>/SKILL.md`。直接克隆进去即可：

```powershell
git clone https://github.com/c940949574/dsh-pwsh-survival-guide "$env:USERPROFILE\.dsh\skills\dsh-pwsh-survival-guide"
```

如果 DSH 用了自定义 home（例如 `DSH_HOME` 指向别处），放到对应的 `skills\` 目录下：

```powershell
git clone https://github.com/c940949574/dsh-pwsh-survival-guide "$env:DSH_HOME\skills\dsh-pwsh-survival-guide"
```

装好后，涉及 PowerShell 的任务会自动加载这份清单；也可以手动让 agent「加载 dsh-pwsh-survival-guide 技能」。

## 同类项目

这个方向已经有不少积累，各有侧重，值得一并参考：

- [GuanKr/pwsh-pitfalls](https://github.com/GuanKr/pwsh-pitfalls) —— 7 个 trap + bash→pwsh 翻译表，定位是「命令失败之后的修复」
- [chaggle/dsh-powershell-check](https://github.com/chaggle/dsh-powershell-check) —— DSH 原生插件，挂在 `tools/pre-execute` 拦截点上**执行前就拦下**有问题的 pwsh 调用，比 skill 更强硬
- [devops-collective-inc/big-book-of-powershell-gotchas](https://github.com/devops-collective-inc/big-book-of-powershell-gotchas) —— 面向人类的经典陷阱合集

本仓库的差异：结论直接来自真实的 DSH agent 会话，侧重**执行环境层面**的坑 —— `ConvertFrom-Json` 误报、`Select-Object -First` 造成的假失败、每次调用都是独立进程、宿主代理配置 —— 而不只是命令语法本身。

## 给非 DSH 用户

`SKILL.md` 除了开头的 YAML frontmatter（`name` / `description` / `whenToUse`）之外就是普通 Markdown，直接当速查表读也没问题。

## 贡献

如果你也踩过别的坑，欢迎开 issue 或 PR —— 请附上**症状原文**（报错信息尽量完整）和**验证过的正确写法**，这样比抽象描述有用得多。

## 许可

[MIT](LICENSE)
