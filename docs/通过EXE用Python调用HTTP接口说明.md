# 通过 EXE 用 Python 调用 HTTP 接口说明

## 1. 先说结论

这个项目的编译版 `.exe` 可以通过 HTTP 被 Python 调用，但要分清两件事：

1. 现有接口可以“触发分析”
2. 现有接口不能“把每一打的结构化结果直接通过 HTTP 返回给 Python”

原因是当前已有的 `/analysis` 接口在成功时只返回：

- `204 No Content`

真正的分析结果，也就是“推荐切哪张、每一打的进张/速度/打点”等内容，会打印在这个 `.exe` 自己的控制台窗口里。

所以现阶段你可以做到的是：

- Python 发请求给 `.exe`
- `.exe` 在自己的终端里输出判断结果

如果你想让 Python 直接拿到“每一打”的机器可读结果，就需要后续改造接口。

## 2. 现有 HTTP 接口是怎么工作的

在 `server.go` 里，已经注册了这些接口：

- `POST /analysis`
- `POST /tenhou`
- `POST /majsoul`

其中和“静态手牌分析”直接相关的是：

- `POST /analysis`

它接收 JSON：

```json
{
  "reset": false,
  "tiles": "34068m 5678p 23567s"
}
```

然后会调用：

- `analysisHumanTiles(model.NewSimpleHumanTilesInfo(d.Tiles))`

也就是把字符串形式的手牌交给现有分析器处理。

## 3. 这个接口能判断什么

`tiles` 的格式沿用命令行格式，支持：

### 3.1 普通手牌分析

例如：

```text
34068m 5678p 23567s
```

### 3.2 带副露的手牌分析

副露用 `#` 表示，暗杠用大写：

```text
234688m 34s # 6666P 234p
```

### 3.3 鸣牌分析

用 `+` 表示他家打出的牌，分析是否吃/碰/鸣后打什么：

```text
33567789m 46s + 6m
```

也就是说，现有接口已经能覆盖：

- 何切分析
- 带副露的何切分析
- 鸣牌后的推荐分析

## 4. 这个接口不能做什么

当前 `/analysis` 有几个重要限制。

### 4.1 不返回结构化推荐结果

成功时只会返回：

- HTTP 204

不会返回：

- 最优打牌
- 所有候选打法
- 进张数
- 综合分
- 和率
- 打点

这些都只打印到控制台。

### 4.2 `reset` 字段目前没有实际作用

虽然请求体里定义了：

```json
{
  "reset": true
}
```

但在当前 `analysis` 处理函数里，这个字段没有被真正使用。

### 4.3 不能通过这个接口传入宝牌参数

命令行支持 `-dora`，但 `/analysis` 当前只接收：

- `tiles`
- `reset`

所以如果你希望 Python 请求时同时指定宝牌，现有接口还不支持。

## 5. 如何启动 EXE 供 Python 调用

最稳妥的方式，是让这个 `.exe` 以服务模式启动。

建议使用：

```powershell
mahjong-helper.exe -analysis -p 12121
```

这里有个很重要的细节：

- `-analysis` 会启动本地服务
- 这个模式走的是 HTTPS
- 证书是程序内置的自签名证书

所以 Python 请求时通常需要：

- 使用 `https://127.0.0.1:12121/analysis`
- 并设置 `verify=False`

## 6. Python 请求最小样例

先安装：

```bash
pip install requests
```

然后这样调用：

```python
import requests

url = "https://127.0.0.1:12121/analysis"
payload = {
    "reset": False,
    "tiles": "34068m 5678p 23567s"
}

resp = requests.post(url, json=payload, verify=False, timeout=10)

print("status:", resp.status_code)
print("body:", resp.text)
```

成功时通常会看到：

```text
status: 204
body:
```

而真正的“每一打推荐”会出现在 `mahjong-helper.exe` 的控制台窗口里。

## 7. 几个可直接用的请求样例

### 7.1 样例一：普通何切分析

```python
import requests

resp = requests.post(
    "https://127.0.0.1:12121/analysis",
    json={
        "reset": False,
        "tiles": "34568m 5678p 23567s"
    },
    verify=False,
    timeout=10,
)

print(resp.status_code)
```

用途：

- 让 `.exe` 分析一副 14 张手牌
- 在控制台显示所有候选切牌及推荐顺序

### 7.2 样例二：带副露的何切分析

```python
import requests

resp = requests.post(
    "https://127.0.0.1:12121/analysis",
    json={
        "reset": False,
        "tiles": "24688m 34s # 6666P 234p"
    },
    verify=False,
    timeout=10,
)

print(resp.status_code)
```

用途：

- 分析副露状态下该打哪张

### 7.3 样例三：鸣牌判断

```python
import requests

resp = requests.post(
    "https://127.0.0.1:12121/analysis",
    json={
        "reset": False,
        "tiles": "33567789m 46s + 6m"
    },
    verify=False,
    timeout=10,
)

print(resp.status_code)
```

用途：

- 分析别人打出 `6m` 时，是否值得吃/碰
- 如果鸣，鸣后推荐打什么

### 7.4 样例四：请求失败时查看错误

如果格式不合法，服务会返回 `400`，错误文本直接在响应体里。

```python
import requests

resp = requests.post(
    "https://127.0.0.1:12121/analysis",
    json={
        "reset": False,
        "tiles": "abc"
    },
    verify=False,
    timeout=10,
)

print("status:", resp.status_code)
print("body:", resp.text)
```

这时你会得到类似：

```text
status: 400
body: 输入错误: ...
```

## 8. 更实用一点的 Python 封装函数

如果你只是想从 Python 里反复触发分析，可以先包一层：

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)


def analyze_tiles(tiles: str, host: str = "127.0.0.1", port: int = 12121):
    url = f"https://{host}:{port}/analysis"
    resp = requests.post(
        url,
        json={"reset": False, "tiles": tiles},
        verify=False,
        timeout=10,
    )

    if resp.status_code == 204:
        return {"ok": True, "message": "analysis triggered; check EXE console output"}

    return {
        "ok": False,
        "status_code": resp.status_code,
        "error": resp.text,
    }


if __name__ == "__main__":
    print(analyze_tiles("34568m 5678p 23567s"))
    print(analyze_tiles("33567789m 46s + 6m"))
```

## 9. 如果我想在 Python 里“拿到每一打结果”，现在怎么办

严格说，现有 HTTP 接口做不到。

因为它没有返回 JSON 结果。

当前你只有三种现实选择：

### 9.1 方案一：只把它当“远程按钮”

Python 只负责发送请求，结果人工看 `.exe` 窗口。

适合：

- 临时工具
- 人工辅助分析

### 9.2 方案二：Python 启动 EXE 并抓控制台输出

你可以不用 HTTP，而是用 `subprocess.Popen(...)` 启动 EXE，然后读取标准输出文本。

但这里有两个问题：

- 输出是给人看的，不是稳定机器接口
- 带颜色/控制台刷新时，文本解析会比较脆

### 9.3 方案三：后续改造 `/analysis` 返回 JSON

这是最推荐的长期方案。

也就是把现有：

- “分析并打印”

改成：

- “分析并返回结构化结果”

这样 Python 才能真正拿到“每一打”的详细判断。

## 10. 一个现实可行的工作流

如果你现在就要先接起来，可以这样做：

1. 手动启动：

```powershell
mahjong-helper.exe -analysis -p 12121
```

2. Python 发请求：

```python
analyze_tiles("34568m 5678p 23567s")
```

3. 去 EXE 控制台看输出结果

这是当前仓库“零改代码”就能实现的接法。

## 11. 如果你想进一步自动化

我建议下一步把接口升级成下面这种形式：

请求：

```json
{
  "tiles": "34568m 5678p 23567s"
}
```

响应：

```json
{
  "shanten": 1,
  "results": [
    {
      "discard": "8s",
      "waits_count": 31,
      "avg_improve_waits_count": 33.62,
      "avg_next_shanten_waits_count": 5.48,
      "mixed_waits_score": 15.00,
      "avg_agari_rate": 44.50,
      "dama_point": 2000,
      "riichi_point": 3900,
      "furiten_rate": 0,
      "yaku_types": ["三色"]
    }
  ]
}
```

这样 Python 才能真正自动使用这个分析器。

## 12. 一句话总结

现有 `.exe` 已经能通过 `POST https://127.0.0.1:12121/analysis` 被 Python 调用，`tiles` 参数也足够触发“每一打”分析；但当前接口只负责触发分析，结果只打印在 EXE 控制台，不会直接返回给 Python。
