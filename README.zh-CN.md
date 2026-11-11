<h1 align="center">phone-driven-booking-scrape</h1>

<p align="center">
  <a href="README.md">English</a> | <b>简体中文</b>
</p>

<p align="center">
  <b>家里有闲置的安卓旧手机吗？</b><br>
  只需一台电脑，打开 USB 调试 —— 开了无线调试之后甚至连 USB 线都不需要，<br>
  就能让那台吃灰的旧手机替你跑遍 Booking、Agoda、Kayak、Vrbo、天巡……<br>
  全网比价机票酒店，还不占用你的电脑。实乃居家旅行的好帮手~（￣▽￣）
</p>

<p align="center">
  以真实设备上的浏览器绕过反爬，而非无头浏览器。
</p>

---

## 问题背景

在旅行网站上做价格比对，通常卡在第一个请求：

| 网站 | 无头浏览器得到的结果 |
|---|---|
| Booking.com | `0 properties found`（日期参数被静默忽略） |
| Kayak | 「KAYAK tror att du är en bot」整页拦截 |
| Agoda | 「抱歉，无法完成搜索」 |
| Vrbo | `Show us your human side…` |
| Momondo / 天巡 | PerimeterX「按住 3 秒」验证 |
| 青旅官网 | `Web Page Blocked! Attack ID: …`（WAF） |
| Omio | Cloudflare 验证页 |

**核心结论：这不是 IP 问题。** 实测环境中，自动化浏览器的出口 IP 与手机完全一致，均为用户家庭宽带。拦截来自**自动化指纹与空 profile（无 cookies、无会话）**。

因此更换代理无效，**更换真实浏览器才有效**。

## 实现原理

```
adb → am start（在真 Chrome 中打开新标签）→ uiautomator dump → 解析文本与坐标 → 按坐标点击
```

- **真实 Chrome**：真实指纹、真实 cookies、真实会话
- **uiautomator dump**：一次调用取得整页文本与像素坐标，开销比截图低一个数量级
- **截图**：仅在需要判断布局、弹窗位置时使用
- **批量点击**：合并进单条 shell 调用，减少往返

整个过程中桌面端保持空闲，全部操作在手机上完成。

## 快速开始

```bash
# 1. 定位 adb（此处使用 Escrcpy 自带版本；安装 scrcpy / Escrcpy 即可获得）
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$("$ADB" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1

# 2. 在真 Chrome 中打开新标签（必须使用 Main activity，原因见「已知陷阱」）
"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html'"
sleep 20

# 3. 读取整页文本与坐标
"$ADB" -s $S shell "uiautomator dump /sdcard/ui.xml"
"$ADB" -s $S pull /sdcard/ui.xml ./ui.xml
```

```python
import re
x = open("ui.xml", encoding="utf-8", errors="ignore").read()
for m in re.finditer(r'(?:text|content-desc)="([^"]{2,90})"[^>]*bounds="\[(\d+),(\d+)\]\[(\d+),(\d+)\]"', x):
    t, a, b, c, d = m.groups()
    print(f"{t!r} @ {(int(a)+int(c))//2},{(int(b)+int(d))//2}")
```

## 三条硬性规则

1. **不得点击付款。** 出现结账页（`Pay now` / `Betala` / `Köp` / `预订`）时立即退出，并明确告知用户未进行任何操作。
2. **不得输入账号、密码或验证码。** 遇到登录墙时交由用户自行登录，或放弃该平台。
3. **只采信最终含税总额。** 列表卡片价存在系统性偏差（见下）。

## 价格口径陷阱

Vrbo 的列表卡片显示：

> `$91/晚 · $182 for 2 nights · All fees included`

实际结账明细：

```
$96.57 × 2      = $193.14
折扣            = -$20.20
税前            = $173.82
税费            = +$43.46
TOTAL           = $217.28   ← 比卡片价高 19%
```

**务必打开「价格详情 / Price details」查看 `Total`。**

各平台计价口径：

| 平台 | 列表显示 | 验证方式 |
|---|---|---|
| Hostelworld | **每晚**（per bed / per room） | 加入购物车查看 `SEK X x 2 nights` |
| Vrbo | **税前** | 价格详情中的 Total |
| Booking.com | 「2 晚价格」，通常已含税 | 结账页总额 |
| Agoda | **每晚**含税 | × 晚数 |
| Airbnb | 「总价」，含清洁费与税费 | 价格明细 |

## 货币选择

不应假设切换币种更便宜，需实测比对。以当地货币价格对照市场汇率：

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=DKK&symbols=SEK,USD,CNY"
```

Booking 实测：`DKK 1,215` → `SEK 1,791.59`（隐含汇率 1.4746，市场 1.4703，**仅高 0.3%，可视为无加价**）；同一房型以 `USD 191` 报价时**高出 4.5%**。**当地货币为最低价。**

## 平台支持情况

手机端可用（无头浏览器被拦）：
`Booking.com` · `Agoda` · `Kayak` · `Momondo` · `Vrbo` · `Hotels.com` · `Omio` · `Flixbus` · 各家青旅官网

无头浏览器即可（无需手机）：
`Airbnb` · `Hostelworld` · `Trip.com`（经 `r.jina.ai` 代理）

均不可用（需安装 App）：
`天巡` 网页版（PerimeterX）· `Kiwi.com`

## 已知陷阱

- `am start -a VIEW -d URL -p com.android.chrome` 会将**当前前台标签**提到最前，实测中曾因此显示用户的付款页面。应使用 `-n …chrome.Main`。
- `vision_analyze` 返回的坐标基于缩放后的基准（常见 954×2048），而实际屏幕为 1080×2340，**需乘以 ~1.132**，否则会命中错误控件（实测中因此把 `wwww` 输入到了优惠码框）。
- 截图为全黑表示屏幕已休眠 → `input keyevent KEYCODE_WAKEUP`。
- Booking / Agoda 会忽略 URL 中的日期参数并返回「0 条结果」。**这并不代表无房**，需通过日历控件操作。
- 单一房型显示 `Sold out` 不等于整店售罄。
- 仅有评分而无评价数的数据不可采信：`10 Superb (1 review)`。

## 仓库结构

```
SKILL.md                          # 工作流、规则、陷阱
references/platform-playbook.md   # 15+ 站点：入口 URL、参数、特性
references/adb-recipes.md         # 可直接复制的 adb 命令与解析代码
```

## License

MIT © 2026 Reese
