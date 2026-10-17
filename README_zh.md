[English](README.md) | **中文**

# ClaudeCode-genimg：让 Claude Code 通过本地 codex CLI 生图

![用 genimg 做的六张图：一次调用画出的灯笼、风筝和青蛙；透明背景的青蛙；同一只青蛙改图戴上派对帽；中文文字原样保留的招牌](media/examples.png)

*测试时的真实输出。上排：三个提示词，一次调用，一个 codex 会话。下排：`-Transparent`（棋盘格 = 真实透明通道）、对这只青蛙 `-Edit`（提示词只写要改的地方）、引号里的文字原样画出。*

## 一句话

在 Claude Code 里说要一张图，就由 [codex CLI](https://github.com/openai/codex) 用你的 **ChatGPT 登录来画，不要 API key，也不按张计费**。一次要多少张都走**一个 codex 会话**；每次运行都会检查 codex 更新有没有改掉脚本依赖的内部格式，坏了会报 `DRIFT` 并附修复流程，而不是悄悄交出错的文件。

## 三个部分

| 部分 | 是什么 |
| --- | --- |
| `skill/genimg/genimg.ps1` | 包装 `codex exec` 的 PowerShell 脚本。任意张图在**一个** codex 会话里画完，存到你指定的位置。 |
| `skill/genimg/SKILL.md` | Claude Code 技能：告诉 Claude 什么时候、怎么调用脚本，什么时候要自己检查图，以及脚本坏了怎么修。 |
| `mod/genimg-set/` | Claude Code 插件，提供 `/genimg-set`：从 codex 自己的列表里选模型和推理强度，并自检脚本。可选。 |

## 和别的方案有什么不同

已经有好几个项目把 `codex exec` 包装成生图工具。这个围绕三点来做：

- **会话少，图多。** 生图工具有频率限制，每个 codex 会话还有固定开销（实测约 35k token 上下文，大部分命中缓存），所以一次调用在一个会话里依次画完整批。多次运行用互斥锁排队，不并行。
- **codex 更新把它弄坏时会发现。** 脚本靠 codex 的会话记录知道哪个文件对应哪张图，这是内部格式，任何版本都可能改。每次运行都把记录和 codex 实际存到图片库里的文件对一遍。对不上时照样交图（按时间从图片库取），同时打印 `DRIFT …`、退出码 2，并把发现写进 `genimg.state.json`。修复流程写在技能里，Claude 可以直接改脚本，再用 `-SelfTest` 确认修好。
- **Claude 的上下文只花在该花的地方。** codex 自己看每张结果，不符合提示词就重画，所以普通图 Claude 不用看。例外是讲道理的图（原理图、流程图、带标注的示意图）：codex 只对照提示词检查，不对照事实，所以技能要求 Claude 画之前先列出要素、连接和标注并核对，画完对照清单看一次，有错就把所有修改合并成一次 `-Edit`。

## 环境要求

- Windows，Windows PowerShell 5.1。
- `PATH` 里有 codex CLI，并用包含生图功能的 ChatGPT 套餐登录（`codex login`）。在 codex `0.159.2` 上验证过。
- Claude Code。技能本身不需要别的；`genimg-set` 插件基于 Claude Code `2.1.288` 的函数钩子插件 API，其他版本可能要改。

## 安装

```powershell
git clone https://github.com/EthanXing-xyy/ClaudeCode-genimg
Copy-Item -Recurse .\ClaudeCode-genimg\skill\genimg "$env:USERPROFILE\.claude\skills\"
```

要用 `/genimg-set`，把插件目录加载进来：

```powershell
claude --plugin-dir <path>\ClaudeCode-genimg\mod\genimg-set
# 或者每个会话都加载：
setx CLAUDE_CODE_PLUGIN_DIRS "<path>\ClaudeCode-genimg\mod\genimg-set"
```

如果脚本不在 `%USERPROFILE%\.claude\skills\genimg`，把 `GENIMG_HOME` 设成脚本所在目录，插件才能找到它，同时改一下 `SKILL.md` 里的路径。

## 用法

在 Claude Code 里直接说要图就行（“画一张…”、“生成一个图标”、“image2.5”）。也可以自己调用脚本：

```powershell
# 一张图
.\genimg.ps1 -Prompt 'a red paper lantern, flat vector' -Out D:\art\lantern.png

# 三张图，一个 codex 会话
.\genimg.ps1 -Prompt 'a red lantern','a blue kite','a green frog' -Out a.png,b.png,c.png

# 同一提示词画四版，透明背景：fox-1.png .. fox-4.png
.\genimg.ps1 -Prompt 'a fox mascot, sticker style' -Out fox.png -Count 4 -Transparent

# 改一张已有的图；原图不会被覆盖
.\genimg.ps1 -Edit frog.png -Prompt 'give the frog a small red party hat' -Out frog-hat.png

# 用一张一次性的图走一遍所有检查
.\genimg.ps1 -SelfTest
```

其他开关：`-Ref`（参考图）、`-Size`、`-PromptFile`、`-Tries` 和 `-NoCheck`（限制或关掉 codex 自己的重画）、`-Model`、`-Effort`、`-TimeoutSec`、`-Force`。

每张图输出一行，最后一行汇总：

```
OK D:\art\lantern.png 1536x1024 938KB
DONE 1/1 drawn=1 37s tokens=7991 codex 5h=2% week=3% model=default effort=low
```

| 输出 | 含义 |
| --- | --- |
| `OK <path> <W>x<H> <n>KB [alpha\|opaque]` | 图已保存（`alpha`/`opaque` 只在 `-Transparent` 时出现） |
| `MISS <path>` | 这张图没画出来 |
| `DONE <made>/<asked> drawn=<n> … codex 5h=<n>% week=<n>%` | 汇总行；`drawn` 含重画次数，百分比是 codex 自己的用量表 |
| `FAIL <reason> log=<folder>` | 什么都没交出来 |
| `DRIFT <what did not hold> codex=<verified>-><now> log=<folder>` | 图交了，但 codex 变了；退出码 2 |

### /genimg-set

- `/genimg-set` 显示当前设置，并在输入框上方打开列表：先列 codex 的模型（读它的 `models_cache.json`，所以列表跟着 codex 更新），再列所选模型的推理强度。按行首数字选择。
- `/genimg-set <model> <effort>` 直接设置；只给一个也行，`default` 表示跟随 codex 自己的配置。
- `/genimg-set test` 运行自检。发现脚本坏了，会把输出交给 Claude 并请它修复。

设置保存在脚本旁边的 `genimg.json`。默认推理强度 `low`。`ultra` 会被拒绝：它把活分给子代理，脚本跟踪不到子代理画的图。

## 诚实的局限

- 这条路走不了的（需要 API）：蒙版、精确像素尺寸、画质档位、PNG 以外的格式。生图模型本身选不了；`-Model` 选的是指挥画图的 codex 模型。
- 一张一张地画。ChatGPT Plus 实测：一张约 40 s，一个会话三张约 106 s。
- 工作目录（提示词和日志）放在 `%TEMP%\genimg`，设置了 `GENIMG_TMP` 就放那里。成功后删除，FAIL 或 DRIFT 时保留。codex 自己的副本留在 `<codex home>\generated_images`。
- 只支持 Windows：脚本是 Windows PowerShell 5.1，插件调用的是 `powershell.exe`。

## 测试情况

在 codex 0.159.2 上测过：单张、三张一批、透明背景、改图、提示词里的非 ASCII 文字、`-SelfTest`、设置能传到 codex，以及在打过补丁的脚本副本上模拟的两条失败路径（改名的命令行参数给出 `FAIL`，不消耗图；改名的记录字段给出交图 + `DRIFT`）。

还没实际跑过：codex 真的重画的一次运行、`-Count`、`-Ref`、`-Size`、超时路径、`MISS`、真实终端里的 `/genimg-set` 列表（只有挂载测试），以及 codex 真正更新后把坏脚本交给 Claude 修的完整过程。

## 目录

```
skill/genimg/
  SKILL.md               Claude Code 技能：何时调用、怎么合批、何时检查、怎么修
  genimg.ps1             包装脚本（只用 ASCII：PowerShell 5.1 按 ANSI 读脚本）
mod/genimg-set/          /genimg-set 插件
  hooks/register.tsx     命令本身、模型/强度选择列表、自检失败时交给 Claude
  hooks/settings.ts      解析 genimg.json、漂移状态和 codex 的模型列表
  tests/                 挂载测试（claude plugin test）
media/examples.png       本页顶部的示例图
```

`genimg.json`（设置）和 `genimg.state.json`（漂移状态）在运行时写到脚本旁边，不进 git。

## 开发

```powershell
claude plugin validate .\mod\genimg-set
claude plugin test .\mod\genimg-set
```

## 贡献者

- [EthanXing-xyy](https://github.com/EthanXing-xyy)
- [Claude](https://claude.com/claude-code)（Anthropic），在 Claude Code 中工作

代码采用 MIT 许可。图片由 OpenAI 的生图工具经 codex 生成，计入 codex 所登录的 ChatGPT 套餐额度。
