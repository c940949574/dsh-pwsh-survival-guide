---
name: dsh-pwsh-survival-guide
description: 在 Windows 上用 PowerShell 跑命令、写临时脚本、调用 git/node/npm/HTTP 时的踩坑清单与正确写法。涵盖引号转义、JSON 校验、编码与 BOM、管道截断、代理、退出码误判等本机反复踩过的坑。
whenToUse: 准备执行 pwsh 命令、写临时脚本（.mjs/.ps1）、下载或解析 JSON、调 GitHub API、用 git 推送、处理中文文本或文件编码时，先看这份清单再动手。
---

# PowerShell 避坑指南（Windows）

这份清单来自实际踩坑记录。**动手前扫一眼，能省掉大量试错。**

> **环境前提**：结论来自一台具体的 Windows 开发机。「代理端口」「已装工具」「Node 版本」
> 这几项是那台机器的现状，换环境请先用第 9、10 条的方法自查自己这边是什么情况；
> 而**引号、BOM、管道截断、退出码**这几类问题与具体机器无关，放哪都适用。

## 三条铁律

1. **复杂脚本写文件，不要内联**
2. **校验 JSON / 解析文本用 node，不要用 PowerShell**
3. **写文件用显式无 BOM 的 UTF-8**

## 什么时候不用翻这份清单

清单本身也有成本 —— 每条命令都查一遍，反而拖慢工作。以下情况直接动手：

- 刚刚已经跑成功、只是换了参数的同类命令
- 纯只读、无引号嵌套、无管道的简单命令（`Get-ChildItem` / `Test-Path` / `node -v`）
- 本文档里出现过的**正确**写法 → 照抄，不用重新推演

---

## 1. 引号：`node -e "..."` 是陷阱

**症状**：`SyntaxError: Invalid or unexpected token` / `Expected ',', got '<eof>'` /
`Expected property name or '}'` —— 脚本本身没错，是 PowerShell 吃掉了转义引号。

```powershell
# ✗ 只要脚本里有引号嵌套、反斜杠、正则，几乎必炸
node -e "const t='x'; console.log(t.split('|'))"

# ✓ 写进文件再执行
# 用 write 工具写 E:\tmp\job.mjs，然后：
node E:\tmp\job.mjs
```

规则：**超过一行、含引号、含正则或中文的脚本，一律落盘再跑。**

## 2. JSON 校验：`ConvertFrom-Json` 会误报

**症状**：`ConvertFrom-Json : Invalid object passed in, ':' or '}' expected. (16641)`
—— 后面的数字是字符偏移，**文件其实是合法的**。这是老版 PowerShell 在大 JSON 上的已知毛病。

```powershell
# ✓ 用 node 校验
node -e "JSON.parse(require('fs').readFileSync('E:/path/file.json','utf8')); console.log('ok')"
```

判断文件是否真坏，**永远以 node 的 `JSON.parse` 为准**。

## 3. HTTP：`Invoke-WebRequest` 必须加 `-UseBasicParsing`

**症状**：`Windows PowerShell is in NonInteractive mode. Read and Prompt functionality is not available.`
（它试图调 IE 引擎渲染，在无交互环境直接失败）

```powershell
# ✓ 抓文本 / 原始字节
$r = Invoke-WebRequest -Uri $url -Headers $h -Proxy $proxy -UseBasicParsing -TimeoutSec 60
$text  = [System.Text.Encoding]::UTF8.GetString($r.RawContentStream.ToArray())
$bytes = $r.RawContentStream.ToArray()

# ✓ 抓 JSON（自动解析成对象）
$obj = Invoke-RestMethod -Uri $url -Headers $h -Proxy $proxy -TimeoutSec 60
```

⚠️ `Invoke-RestMethod` 会把 JSON 响应**解析成对象**，`ConvertTo-Json` 回写会改变格式与顺序。
要保存原始文件用 `Invoke-WebRequest -UseBasicParsing` + `RawContentStream`。

## 4. 编码与 BOM

```powershell
# ✗ 老版本会写 BOM：JSON.parse 报错、git 把 BOM 当正文、diff 出现鬼影首行
Set-Content $p $text -Encoding UTF8
$text | Out-File $p -Encoding utf8

# ✓ 明确无 BOM
[System.IO.File]::WriteAllText($p, $text, (New-Object System.Text.UTF8Encoding($false)))
# ✓ 需要保留原有 BOM 时
[System.IO.File]::WriteAllText($p, $text, (New-Object System.Text.UTF8Encoding($true)))
```

**控制台里的中文乱码是显示问题，不是文件问题。** 判断文件内容用 `read` 工具，别靠 `Get-Content` 的输出。

## 5. 管道截断会造成"假失败"

```powershell
git log --oneline | Select-Object -First 8      # 后面会跟一句 [exit code: 1]
```

`Select-Object -First N` 提前关闭管道 → 上游进程收到断管 → 退出码变成 1，
还常伴 `NativeCommandError`。**这不是命令失败**，是截断的副作用。

- 想保留真实退出码：不要截断，或把退出码在截断**之前**取出来
- 只想看尾部就用 `Select-Object -Last N`
- git / npm 的进度输出走 stderr，成功也会显示成红字错误 —— 同样忽略

## 6. 统计字符串出现次数

```powershell
# ✗ String.Split(string) 是按“字符”拆的，结果会是几万
$t.Split("keyword").Length - 1

# ✓
([regex]::Matches($t, [regex]::Escape("keyword"))).Count
```

## 7. 环境是「一次性」的

每次 `pwsh` 调用都是**新进程**：变量、`$env:*`、当前目录都不保留。

- 切目录用工具参数 `workdir`，不要靠 `cd`
- 需要重复使用的值（token、路径）每次命令里重新赋值

## 8. 代理

**系统代理开着，git 却不读它** —— 在 Windows 上非常常见，症状是直连被 reset。

先查出系统代理（本机示例：clash 监听 `127.0.0.1:6789`）：

```powershell
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' |
    Select-Object ProxyEnable, ProxyServer
```

查到端口后：

```powershell
# ✗ 直连会被 reset：fatal: unable to access ... Recv failure: Connection was reset
git ls-remote https://github.com/owner/repo.git

# ✓ 单次指定代理（推荐，不污染用户全局配置）
git -c http.proxy=http://127.0.0.1:6789 -c https.proxy=http://127.0.0.1:6789 ls-remote https://github.com/owner/repo.git

# ✓ 或只给这个仓库配（repo 级，不影响 ~/.gitconfig）
git -C <repo> config http.proxy http://127.0.0.1:6789
```

`Invoke-RestMethod` / `Invoke-WebRequest` 用 `-Proxy http://127.0.0.1:6789`。

## 9. 命令是否存在，先探测

有些顺手就用的 CLI 其实没装（本机就没有 `gh`、`rg`）。别直接调用：

```powershell
if (Get-Command gh -ErrorAction SilentlyContinue) { ... } else { "gh not installed" }
```

搜索优先用宿主提供的 grep 工具（比 `Select-String` 快，且能处理大文件），
没有工具时再退回 `Select-String`。

## 10. 版本行为差异

同一台机器上 `Invoke-WebRequest` / `ConvertFrom-Json` 的表现像 Windows PowerShell 5.1
（需要 `-UseBasicParsing`、大 JSON 误报），但 `Get-ChildItem -Depth` 又能用（7.0+ 特性）。
**不要赌具体版本**：

- 只用两边都有的参数
- `-Depth` 之类新版专属参数，先小范围试一条
- 拿不准就换写法（`Get-ChildItem -Recurse | Where-Object`）

## 11. 其他本机要点

- 建目录联接用 `cmd /c mklink /J <link> <target>`（比复制 `node_modules` 快得多）
- Node v24 自带 zstd，但 `zstdDecompressSync` **不处理多帧流**（会报
  `Unknown frame descriptor`）→ 按 `28 b5 2f fd` 魔数切帧后逐帧解压
- 发含中文的 HTTP body：`[System.Text.Encoding]::UTF8.GetBytes($json)` 传 `-Body`，
  `ContentType` 写 `application/json; charset=utf-8`
- 后台任务用工具自带的 `run_in_background`，不要用 `Start-Job` / `Start-Process`

---

## 12. 路径：`-Path` 会把它当通配符

```powershell
# ✗ 路径里的 [ ] 被当成通配符 → 报“找不到路径”
Get-Content "C:\logs\app[1].log"
Get-ChildItem "C:\logs\v[2]"

# ✓ 按字面路径处理
Get-Content -LiteralPath "C:\logs\app[1].log"
```

规则：路径来自变量或用户输入、或含 `[ ] * ?` 时，一律用 `-LiteralPath`。

## 13. 退出码有两套系统，别混用

```powershell
git push
$?                # 布尔：上一条命令成功没有
$LASTEXITCODE     # 数字：只有原生程序（.exe / .cmd）才会更新它
```

坑在于 **PowerShell 自己的 cmdlet 不更新 `$LASTEXITCODE`**。
`Test-Path x; if ($LASTEXITCODE)` 判断的是**更早那一条 exe** 的退出码，跟 `Test-Path` 毫无关系。

- 判断 cmdlet 成败 → `$?` 或 `-ErrorAction`
- 判断 exe（git / node / npm）→ `$LASTEXITCODE`
- 宿主显示的 `[exit code: N]` 取的就是 `$LASTEXITCODE`，所以会被管道截断污染（见第 5 条）

## 14. 紧凑参数传给原生程序可能被拆开

```powershell
# ✗ 传给 .exe / .cmd 的紧凑参数可能被重新解析
some.exe -Dkey=value
# ✓ 用引号把整个参数锁住
some.exe "-Dkey=value"
```

遇到"参数明明写对了却报错"时，先试加引号。
（这条来自 [GuanKr/pwsh-pitfalls](https://github.com/GuanKr/pwsh-pitfalls) 的实测记录，本机未复现过。）

## 15. Unix 工具不存在，或用的是「同名不同物」

```powershell
find . -name "*.log"     # ✗ 这是 Windows 自带的 find.exe（搜字符串用），不是 GNU find
grep -r foo .            # ✗ 默认没有
sed / awk / head / tail  # ✗ 默认没有
```

替代：`Get-ChildItem -Recurse`、`Select-String`，或直接用宿主提供的 glob / grep 工具（见第 9 条）。
**别把 bash 惯用法直接搬过来** —— 报错信息往往看起来很莫名其妙。

## 16. `&&` / `||` 在 PowerShell 5.1 里是语法错误

```powershell
cmd1 && cmd2             # ✗ 5.1 直接报错（7.0+ 才支持）
cmd1; if ($?) { cmd2 }   # ✓ 跨版本都能用
```

写脚本时一律用 `;` + `if ($?)`，别赌运行环境是 7。

## 17. 执行策略只对当次生效

```powershell
powershell -ExecutionPolicy Bypass -File script.ps1   # 只影响这一次调用
Set-ExecutionPolicy -Scope Process Bypass             # 只影响当前会话
```

别去改 `LocalMachine` 作用域 —— 那是对整台机器的改动，属于越权操作。

---

## 写在最后

踩坑时的固定动作：**先问自己这三句**

1. 这个脚本是不是该落盘？（引号 / 多行 / 正则 / 中文）
2. 这个校验是不是该交给 node？（JSON / 文本解析）
3. 这个写文件有没有处理 BOM？（会被 git 或 JSON.parse 读到）
