# realtime_pipeline.py 调用方式说明

## 1. 文件来源

已将你给的文件拉到当前项目的 `docs` 目录下：

- `docs/realtime_pipeline.py`

来源仓库：

- [yuzeis/StarResonanceMahjongAutoAna](https://github.com/yuzeis/StarResonanceMahjongAutoAna)

原始文件页面：

- [realtime_pipeline.py](https://github.com/yuzeis/StarResonanceMahjongAutoAna/blob/main/realtime_pipeline.py)

## 2. 先说结论

这个 `realtime_pipeline.py` 并不是通过 HTTP 调用 `mahjong-helper` 的。

它的调用方式是：

1. 先抓游戏网络包
2. 从 protobuf 数据里解析出当前手牌
3. 把手牌转成 `mahjong-helper.exe` 能识别的字符串
4. 直接用 `subprocess.run(...)` 启动 `mahjong-helper.exe`
5. 读取标准输出
6. 把输出显示到它自己的 GUI 文本框里

也就是说，它走的是：

- `Python -> subprocess -> mahjong-helper.exe`

而不是：

- `Python -> HTTP -> mahjong-helper server`

## 3. 它的整体工作流

这个脚本可以分成四层。

### 3.1 抓包层

它使用：

- `PacketCapture`

去监听目标进程的网络流量，默认参数里目标进程名是：

- `Star.exe`

抓到的数据会落到 `bins` 目录。

### 3.2 解析层

它调用：

- `parse_pb_bin(...)`

把抓到的二进制包解析成 protobuf 数据。

然后再通过：

- `HM.extract_hand_cards(parsed)`

把当前手牌和副露提取出来。

这里的 `HM` 来自它动态加载的：

- `mahjong-hand-monitor.py`
- 或 `mahjong_hand_monitor.py`

## 3.3 牌串构造层

拿到手牌后，它会调用：

- `render_compact(cards, melds)`

把牌面转成 `mahjong-helper.exe` 能直接吃的表达式格式。

例如最终会形成类似：

- `34568m5678p23567s`
- `24688m34s#6666P234p`

这个格式本质上就是 `mahjong-helper` 命令行支持的那套输入语法。

### 3.4 调用层

当牌数满足条件时，它会调用：

- `run_helper(helper_path, compact, dora_spec=...)`

而 `run_helper(...)` 内部真正做的事情是：

- 组装命令行参数
- `subprocess.run(...)`
- 捕获 stdout/stderr
- 解码输出文本

## 4. 它具体怎么调用 mahjong-helper.exe

核心函数是：

- `run_helper(helper_path: Path, expr: str, dora_spec: str = None) -> str`

它会构造命令：

```text
mahjong-helper.exe [可选 -d=宝牌] 表达式
```

也就是：

```text
mahjong-helper.exe 34568m5678p23567s
```

或者：

```text
mahjong-helper.exe -d=5s 34568m5678p23567s
```

然后用：

```python
subprocess.run(cmd, capture_output=True, check=False, timeout=10, ...)
```

去直接执行这个 EXE。

执行完成后：

- 标准输出和标准错误会被拼起来
- 按 `utf-8`、`gbk`、`cp936`、`latin1` 依次尝试解码
- 最终把文本返回给 GUI

## 5. 什么时候会自动触发分析

在 `on_bin(info)` 里，脚本每次抓到并解析出新手牌后，会计算：

- 当前手牌 + 副露总牌数

对应函数是：

- `total_tiles(cards, melds)`

然后它只在下面这个条件成立时自动调用 `mahjong-helper.exe`：

- `t_total == 14`

也就是：

- 当它认为“现在轮到自己出牌，应该做何切分析”时，才自动触发

否则只更新 GUI，不自动分析。

它显示的提示就是：

- 合计 14 张 -> 自动分析
- 不等于 14 张 -> 仅显示当前牌串，不触发分析

## 6. 它是怎么避免重复调用的

这个脚本做了几层去重和防抖：

1. `extract_hand(...)` 后，如果状态和上次完全一样，就跳过
2. 如果牌串和上次一样，并且时间间隔很短，也跳过
3. 只有抓到的新状态满足 14 张时，才执行 helper

相关变量包括：

- `last_state`
- `last_compact`
- `last_ts`
- `last_all_tiles`

所以它不是每抓到一个包就疯狂起一个 EXE，而是尽量只在手牌真正变化时调用。

## 7. 宝牌是怎么传给 helper 的

这个脚本没有走 HTTP，所以也不受你前面提到的 `/analysis` 接口限制。

它是直接用命令行参数把宝牌传给 `mahjong-helper.exe` 的：

```text
-d=...
```

GUI 里支持用户输入：

- `d 牌`

例如：

- `d 5s`
- `d 东南白`

脚本会先用 `normalize_tile_spec(...)` 把中文牌面转成标准格式，再追加到 `dora_spec` 里。

之后每次调用 helper 时，都会变成：

```text
mahjong-helper.exe -d=... 表达式
```

## 8. 它还支持哪些手动调用方式

GUI 输入框里除了自动分析，还支持几种手动命令。

### 8.1 `d`

追加宝牌：

```text
d 5s
d 东南白
```

### 8.2 `dc`

清空宝牌：

```text
dc
```

### 8.3 `fl`

做副露分析：

```text
fl 3s
fl 3索
fl 1饼
```

这里它会把当前自动抓到的手牌拼上 `+牌`，构造成：

```text
当前牌串 + 来牌
```

例如：

```text
33567789m46s+6m
```

然后再交给 `mahjong-helper.exe` 去分析鸣牌。

### 8.4 `h`

手动直接输入完整表达式：

```text
h 234688m34s#6666p+3m
```

或者你也可以直接输入一个带数字的牌串，它会默认当成表达式去跑。

## 9. 它和你当前想走的 HTTP 方案有什么区别

这是最重要的一点。

### 9.1 `realtime_pipeline.py` 的方式

优点：

- 不需要启动 helper 的 HTTP 服务
- 能直接读取 helper 的完整控制台输出
- 能用命令行参数传 `-d=...`
- 对现有 `mahjong-helper.exe` 兼容得很好

缺点：

- 每次分析都要起一个子进程
- 结果是文本，不是结构化 JSON
- 更适合桌面辅助工具，不太适合服务化调用

### 9.2 你前面想用的 HTTP 方式

优点：

- Python 调用简单
- 可以常驻服务，不用每次起 EXE
- 更适合程序间集成

缺点：

- 当前项目现有 `/analysis` 接口不会返回结构化结果
- 目前也不能通过 HTTP 传 dora

### 9.3 结论

`realtime_pipeline.py` 选 CLI 子进程而不是 HTTP，是合理的，因为它需要：

- 快速接入现有 EXE
- 直接展示完整分析文本
- 顺手带上 `-d=...`

它更像是“把现成命令行工具包进一个实时 GUI 管道里”。

## 10. 一个简化后的伪代码

它的核心逻辑可以简化成下面这样：

```python
抓包 -> 解析 protobuf -> 提取当前手牌

if 手牌状态变化:
    compact = 转成 mahjong-helper 表达式
    GUI显示当前牌串

    if 总牌数 == 14:
        out = subprocess.run([
            "mahjong-helper.exe",
            "-d=宝牌(可选)",
            compact
        ])
        GUI显示 out
    else:
        GUI显示“未触发自动分析”
```

## 11. 对你有什么直接参考价值

如果你只是想“把 mahjong-helper 接进 Python 程序里”，这个文件其实已经给了你一条非常现实的参考路线：

1. 不改 `mahjong-helper.exe`
2. 在 Python 中准备好牌串
3. 用 `subprocess.run(...)` 直接调用
4. 读取输出文本

如果你的需求是：

- 做实时辅助工具
- 做 GUI 小工具
- 先快速跑通

那么这条路线非常实用。

但如果你的需求是：

- Python 程序要拿到结构化推荐结果
- 后端服务要稳定调用
- 想做 API 化集成

那么 HTTP/JSON 改造仍然是更好的长期方案。

## 12. 一句话总结

`realtime_pipeline.py` 调用 `mahjong-helper` 的方式不是 HTTP，而是把 `mahjong-helper.exe` 当成命令行工具，通过 `subprocess.run(...)` 直接执行，并把输出文本显示到自己的 GUI 里。
