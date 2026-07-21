# Playwright 被检测怎么办：Cloudflare 检测机制变了，2026 年该怎么应对

用 Playwright 做自动化，被网站识别、弹验证码、请求被拦，是很多人都遇到过的问题。网上大部分教程还停留在"装个 stealth 插件就好"，但 2026 年 7 月之后，情况变了。

这篇讲清楚三件事。Cloudflare 的检测机制最近发生了什么变化，为什么插件伪装这条路越来越走不通，以及分层来看到底该怎么应对。附可运行的 Python 代码。

## 目录

- [先说最新的变化，Cloudflare Precursor](#先说最新的变化cloudflare-precursor)
- [检测是分层的，先搞清楚各层在查什么](#检测是分层的先搞清楚各层在查什么)
- [第一层，先看看你的 Playwright 暴露了什么](#第一层先看看你的-playwright-暴露了什么)
- [插件方案，playwright-stealth 能做什么](#插件方案playwright-stealth-能做什么)
- [插件方案的三个天花板](#插件方案的三个天花板)
- [环境层，用指纹浏览器提供真实环境](#环境层用指纹浏览器提供真实环境)
- [多账号场景，插件方案彻底失效的地方](#多账号场景插件方案彻底失效的地方)
- [行为层，没有任何工具能替你解决](#行为层没有任何工具能替你解决)
- [总结](#总结)

## 先说最新的变化，Cloudflare Precursor

2026 年 7 月 13 日，Cloudflare 正式发布了 Precursor，一套持续性行为验证引擎。

它和以前的检测方式有根本区别。过去的挑战页是在访客到达时测一次，通过了，之后的行为就默认没问题。Precursor 不是这样，它直接跑在浏览器里，把交互信号实时传回 Cloudflare 边缘节点打分，监测的是整个会话。

Cloudflare 自己的说法是，一次性检查太容易伪造。一个机器人可以假装单个动作，但要伪造一整个会话、还带上真人那种不规律的时间节奏，工程成本就高得多。

更关键的一点，传统挑战每个请求重置一次，而 Precursor 是跨页面持续累积评估的，自动化程序没法靠刷新页面重置自己的行为特征。

它明确瞄准的目标，是那种"已经能执行 JavaScript、能开真实浏览器、能绕过传统反爬防御"的 agent 类自动化。

顺带一个数字，Cloudflare 说自动化流量已经首次超过真人，占全网请求约 57%。

对做自动化的人来说，这意味着一件事。**光让浏览器"看起来不像机器人"已经不够了，你的整个操作过程也在被看。**

## 检测是分层的，先搞清楚各层在查什么

要应对，先得知道对方在查什么。检测大致分三层，每层的应对方式完全不同，混在一起谈就会得出"装个插件就完事"这种错误结论。

**第一层，请求层。** IP 归属（机房还是住宅）、请求频率、TLS 指纹、HTTP 头顺序。这层最表面。

**第二层，环境层，也就是浏览器指纹。** 你的浏览器暴露出的一整套特征，User-Agent、WebDriver 标志、Canvas 绘图差异、WebGL 显卡信息、字体、时区、屏幕参数等。这些组合起来几乎独一无二，足以标识一台设备。

**第三层，行为层。** 鼠标怎么移动、有没有停顿、点击间隔是否过于均匀、页面停留时间是否合理、操作路径像不像真人。Precursor 加强的就是这一层。

网上绝大多数教程只处理了第二层的一部分，所以在防护升级之后就不管用了。

## 第一层，先看看你的 Playwright 暴露了什么

动手之前，先看看没做任何处理的 Playwright 暴露了什么。用 bot.sannysoft.com 这个检测页，它会跑一系列测试并标出哪些项没通过。

先装 Playwright：

```bash
pip install playwright
playwright install chromium
```

跑一个最基础的脚本：

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://bot.sannysoft.com/")
    page.screenshot(path="before.png", full_page=True)
    browser.close()
```

打开 before.png，你会看到几项标红。典型的是 WebDriver 那一项直接暴露，User-Agent 里带着 HeadlessChrome 字样，一眼就是自动化。

<img width="1440" height="7273" alt="image" src="https://github.com/user-attachments/assets/d8f75749-74c9-463b-aac9-0616a333c6b9" />

## 插件方案，playwright-stealth 能做什么

最常见的免费方案是 playwright-stealth，它通过修改浏览器暴露的一些属性来降低被识别概率。

安装：

```bash
pip install playwright-stealth
```

用法：

```python
from playwright.sync_api import sync_playwright
from playwright_stealth import stealth_sync

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()

    stealth_sync(page)

    page.goto("https://bot.sannysoft.com/")
    page.screenshot(path="after.png", full_page=True)
    browser.close()
```

再看 after.png，之前标红的几项通常会转绿。

<img width="1440" height="7654" alt="image" src="https://github.com/user-attachments/assets/7d82cacb-00ef-487e-9c7f-a3fe0ef06497" />

对付基础的反爬检测，到这一步通常够用了。**但也就到这里了。**

## 插件方案的三个天花板

**第一，它只覆盖第二层的一部分。** stealth 类插件改的是 JS 层面能改的浏览器属性，但更深的 Canvas 指纹一致性、WebGL 真实性、TLS 指纹这些，它覆盖不到或覆盖不干净。面对 Cloudflare、DataDome 这类系统，容易被识破。

**第二，它完全不处理第三层。** 这是 Precursor 之后最要命的一点。插件让你的浏览器"看起来像真浏览器"，但它管不了你的操作行为像不像真人。而 Precursor 恰恰不太在意你伪装了什么属性，它看的是整个会话的行为节奏。你的脚本如果 0.5 秒一个点击、鼠标直线移动、每次停留时间一模一样，属性伪装得再好也没用。

**第三，多账号场景下彻底失效。** 如果你要用 Playwright 管理多个账号，这些账号都跑在同一套浏览器环境、同一个指纹下。网站通过指纹发现这几个账号背后是同一台设备，直接判定关联，一封一串。插件没法给每个账号一套真正独立的指纹，因为它本质上只是在同一个浏览器上打补丁。

## 环境层，用指纹浏览器提供真实环境

针对第二层，比打补丁更彻底的思路是不再伪装，而是直接用一个真实的、指纹独立的浏览器环境。

指纹浏览器从内核层面提供完整真实的浏览器指纹，而不是在 JS 层面补。每个环境有独立的指纹和 cookie，可以各自配置独立 IP。对检测系统来说，它就是一个真实用户的浏览器，而不是一个被伪装过的自动化工具。

关键是不影响你现有代码。以 AdsPower 为例，它给每个浏览器环境暴露一个 CDP 端口，Playwright 可以用 `connect_over_cdp` 直接接管。

先通过本地接口启动环境：

```python
import requests

# profile_id 是你在 AdsPower 里创建的环境 ID
resp = requests.get(
    "http://local.adspower.net:50325/api/v1/browser/start",
    params={"user_id": "你的_profile_id"},
)
ws_endpoint = resp.json()["data"]["ws"]["puppeteer"]
```

然后用 Playwright 接管这个已启动的环境：

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(ws_endpoint)
    context = browser.contexts[0]
    page = context.pages[0] if context.pages else context.new_page()

    # 下面就是你熟悉的 Playwright 代码
    page.goto("https://bot.sannysoft.com/")
    page.screenshot(path="adspower.png", full_page=True)
```

用完关闭环境：

```python
requests.get(
    "http://local.adspower.net:50325/api/v1/browser/stop",
    params={"user_id": "你的_profile_id"},
)
```

因为这里的浏览器本身就是一个真实的指纹环境，检测页那些项会自然通过，不需要额外伪装。而你的 Playwright 代码几乎原样保留，只是把自己启动浏览器换成了接管已有环境。

<img width="1567" height="8549" alt="image" src="https://github.com/user-attachments/assets/410adb66-67e8-49ed-be56-a407e1361af6" />
<img width="1370" height="762" alt="image" src="https://github.com/user-attachments/assets/34dc75ee-1a93-4aec-828b-47bb0c92e0be" />


## 多账号场景，插件方案彻底失效的地方

如果你的需求涉及多账号，环境隔离的价值会更明显。

在 AdsPower 里为每个账号建一个独立环境，各自独立的指纹和 IP，然后批量启动、用 Playwright 分别接管：

```python
import requests
from playwright.sync_api import sync_playwright

profile_ids = ["profile_1", "profile_2", "profile_3"]

with sync_playwright() as p:
    for pid in profile_ids:
        resp = requests.get(
            "http://local.adspower.net:50325/api/v1/browser/start",
            params={"user_id": pid},
        )
        ws = resp.json()["data"]["ws"]["puppeteer"]

        browser = p.chromium.connect_over_cdp(ws)
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        # 对每个账号执行各自的逻辑
        page.goto("https://example.com")
        # ...
```

这样每个账号在网站看来都是一台独立设备、一个独立用户，不会因为指纹相同被关联。

这是插件方案做不到的。插件能让一个浏览器看起来不像机器人，但没法让多个账号看起来像多个不同的人。

## 行为层，没有任何工具能替你解决

这一节要说句实话。

**指纹浏览器解决的是环境层，它不自动解决行为层。** 如果有人告诉你某个工具能一键搞定 Precursor 这类行为检测，那不是真的。

Precursor 盯的是你整个会话的交互特征，而这取决于你的代码怎么写，不取决于你用什么浏览器。环境再真实，如果操作节奏是机械的，一样会被打上高 bot score。

这一层能做的事，都在你自己的代码里。

操作之间加入不固定的间隔，而不是每次都 sleep 同一个数值。鼠标移动用带轨迹的方式，而不是直接瞬移到目标坐标。页面停留时间跟内容长度有关系，而不是所有页面都停 2 秒。不要在毫秒级完成人类需要几秒才能做完的操作序列。控制单个会话的操作总量，不要一个会话干完几百件事。

说到底，环境层可以靠工具，行为层只能靠设计。两层都做到位，才谈得上稳定。

## 总结

2026 年 7 月 Cloudflare 上线 Precursor 之后，检测的重心从"进门查一次"变成了"全程看着"。这让只处理浏览器属性的插件方案，天花板变得非常明显。

分层来看的话，思路是这样的。

请求层，用合适的 IP 策略，住宅或移动代理，控制频率。

环境层，基础需求用 playwright-stealth 打补丁够用，面对高级防护或者需要多账号隔离时，用指纹浏览器提供真实独立环境更稳，而且能通过 CDP 无缝接入现有 Playwright 代码。指纹浏览器可以参考 [AdsPower](https://www.adspower.net/)。

行为层，没有工具能代劳，只能在自己代码里设计合理的操作节奏。

想更深入理解检测原理，可以看这几份资源。

- [Selenium 被检测怎么办](https://github.com/pencil20388-eng/selenium-anti-detection)，Selenium 侧的完整方案
- [为什么你的脚本一跑就被封](https://github.com/pencil20388-eng/why-bots-get-banned)，面向新手的反检测扫盲
- [awesome-anti-detect](https://github.com/pencil20388-eng/awesome-anti-detect)，指纹浏览器、反检测、代理策略资源汇总

---

以上内容用于说明检测机制原理，以及合规场景下的自动化测试与多账号运营。请遵守目标网站的服务条款和当地法律法规。
