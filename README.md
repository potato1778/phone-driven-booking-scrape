# phone-driven-booking-scrape

> **家里有闲置的安卓旧手机吗？**
> 只需一台电脑，打开 USB 调试 —— 开了无线调试之后甚至连 USB 线都不需要，
> 就能让那台吃灰的旧手机替你跑遍 Booking、Agoda、Kayak、Vrbo、天巡……
> 全网比价机票酒店，还不占用你的电脑。实乃居家旅行的好帮手~（￣▽￣）

**用真手机绕过反爬，而不是无头浏览器。**

A Hermes Agent skill that beats flight & hotel bot-walls by driving the real Chrome app on an Android phone via `adb` — genuine browser fingerprint, real cookies, residential IP — while your desktop stays free.

---

# 中文

## 这玩意儿解决什么问题

在旅行网站上比价，第一步就卡住：

| 网站 | 无头浏览器拿到的结果 |
|---|---|
| Booking.com | `0 properties found`（日期参数被静默忽略） |
| Kayak | 「KAYAK tror att du är en bot」整页拦截 |
| Agoda | 「抱歉，无法完成搜索」 |
| Vrbo | `Show us your human side…` |
| Momondo / 天巡 | PerimeterX「按住 3 秒」验证 |
| 青旅官网 | `Web Page Blocked! Attack ID: …`（WAF） |
| Omio | Cloudflare 验证页 |

**关键认知：这不是 IP 问题。** 实测里自动化浏览器的出口 IP 就是用户家的宽带，和手机完全一样。拦截来自**自动化指纹 + 空 profile（没有 cookies、没有会话）**。

所以换代理没用，**换真实浏览器才有用**。

## 原理

```
adb → am start（在真 Chrome 里开新标签）→ uiautomator dump → 解析文本+坐标 → 按坐标点击
```

- **真 Chrome** = 真指纹、真 cookies、真会话
- **uiautomator dump** = 一次调用拿到整页文字 + 像素坐标，比截图便宜一个数量级
- **截图**只在需要判断布局、弹窗位置时用
- **批量点击**写在一条 shell 里，省往返

电脑全程不用管，一切跑在你旁边那台手机上。

## 三步上手

```bash
# 1. 找 adb（这里用 Escrcpy 自带的，装了 scrcpy/Escrcpy 就有）
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$("$ADB" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1

# 2. 在真 Chrome 里开新标签（注意是 Main activity，见下方「踩过的坑」）
"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html'"
sleep 20

# 3. 把整页读成文本 + 坐标
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

## 三条铁律

1. **永不点付款。** 遇到结账页（`Pay now` / `Betala` / `Köp` / `预订`）立刻退出，并告诉用户你没碰。
2. **永不输入账号、密码、验证码。** 碰到登录墙就让用户自己登，或者放弃这个站。
3. **只认最终含税总额。** 卡片价会骗人（见下）。

## 价格陷阱（真实踩过的坑）

Vrbo 的卡片写：

> `$91/晚 · $182 for 2 nights · All fees included`

实际结账明细：

```
$96.57 × 2      = $193.14
折扣            = -$20.20
税前            = $173.82
税费            = +$43.46
TOTAL           = $217.28   ← 比卡片价高 19%
```

**永远点开「价格详情 / Price details」看 `Total`。**

各平台的计价口径：

| 平台 | 列表显示 | 怎么验证 |
|---|---|---|
| Hostelworld | **每晚**（per bed / per room） | 加入购物车看 `SEK X x 2 nights` |
| Vrbo | **税前** | 价格详情里的 Total |
| Booking.com | 「2 晚价格」，一般已含税 | 结账页总额 |
| Agoda | **每晚**含税 | × 晚数 |
| Airbnb | 「总价」含清洁费税费 | 价格明细 |

## 货币也不是随便切就能省钱

别假设换币种更便宜，**实测**。拿当地货币价对市场汇率：

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=DKK&symbols=SEK,USD,CNY"
```

Booking 实测：`DKK 1,215` → `SEK 1,791.59`（隐含汇率 1.4746，市场 1.4703，**只高 0.3%，等于没加价**），而同一间房报 `USD 191` 时**贵了 4.5%**。**当地货币才是最便宜的。**

## 平台支持

手机端能过（无头被拦）：
`Booking.com` · `Agoda` · `Kayak` · `Momondo` · `Vrbo` · `Hotels.com` · `Omio` · `Flixbus` · 各家青旅官网

无头就能用（不用掏手机）：
`Airbnb` · `Hostelworld` · `Trip.com`（走 `r.jina.ai` 代理）

全线阵亡（得装 App）：
`天巡` 网页版（PerimeterX）· `Kiwi.com`

## 踩过的坑

- `am start -a VIEW -d URL -p com.android.chrome` 会把**前台那个标签**提到前面 —— 有一次直接把用户的付款页翻了出来。要用 `-n …chrome.Main`。
- `vision_analyze` 报的坐标是缩放后的基准（常见 954×2048），实际屏幕 1080×2340，**要乘 ~1.132**，否则会点错格（我就是这样把 `wwww` 输进了优惠码框）。
- 截图全黑 = 屏幕睡了 → `input keyevent KEYCODE_WAKEUP`。
- Booking / Agoda 会忽略 URL 里的日期参数并返回「0 条结果」。**那不是没房**，得去点日历。
- 某个房型显示 `Sold out` ≠ 整店没房。
- 只有评分没有评价数的别信：`10 Superb (1 review)`。

---

# English

## The problem

Price-comparison work on travel sites fails at the very first request:

| Site | What a headless browser gets |
|---|---|
| Booking.com | `0 properties found` (dates silently ignored) |
| Kayak | full-page "we think you're a bot" block |
| Agoda | "Sorry, search could not be completed" |
| Vrbo | `Show us your human side…` |
| Momondo / Skyscanner | PerimeterX press-and-hold challenge |
| Hostel brand sites | `Web Page Blocked! Attack ID: …` (WAF) |
| Omio | Cloudflare interstitial |

**The key insight: it is not an IP problem.** In the reference setup the automated browser's egress IP was the user's own home broadband — identical to the phone. The blocks come from **automation fingerprint + an empty profile (no cookies, no session)**.

So a proxy won't help. **A real browser will.**

## How it works

```
adb → am start (new tab in real Chrome) → uiautomator dump → parse text + bounds → tap by coordinate
```

- **Real Chrome** = real fingerprint, real cookies, real session
- **uiautomator dump** = whole page as text plus pixel coordinates in one call, far cheaper than screenshots
- **Screenshots** only when layout or popup position matters
- **Batch taps** inside one shell call to cut round-trips

Your computer stays free the whole time.

## Quick start

```bash
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$("$ADB" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1

"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html'"
sleep 20

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

## Hard rules

1. **Never tap payment.** On any checkout page (`Pay now`, `Betala`, `Köp`) — back out immediately and tell the user you did not touch it.
2. **Never type credentials or OTPs.** Hit a login wall → let the user log in, or skip that site.
3. **Only trust the final all-in total.** Card prices lie.

## The price trap that cost a wrong answer

Vrbo's card said `$91/night · $182 for 2 nights · All fees included`. The real breakdown:

```
$96.57 × 2      = $193.14
discount        = -$20.20
pre-tax         = $173.82
tax             = +$43.46
TOTAL           = $217.28   ← 19% above the card
```

**Always open "Price details" and read `Total`.**

| Platform | Listing shows | Verify with |
|---|---|---|
| Hostelworld | **per night** (per bed / per room) | Add to cart → `SEK X x 2 nights` |
| Vrbo | **pre-tax** | Price details → Total |
| Booking.com | "2 nights" price, usually tax-inclusive | Checkout total |
| Agoda | **per night**, tax-inclusive | × nights |
| Airbnb | "total" incl. cleaning + fees | Price breakdown |

## Currency is not a free lunch

Don't assume switching currency is cheaper — measure it. Compare the local-currency price against the market rate:

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=DKK&symbols=SEK,USD,CNY"
```

Measured on Booking.com: `DKK 1,215` → `SEK 1,791.59` (implied 1.4746 vs market 1.4703, **+0.3% — no markup**), while the same room in `USD 191` was **4.5% more expensive**. Local currency won.

## Platform support

Works through the phone (blocked headless):
`Booking.com` · `Agoda` · `Kayak` · `Momondo` · `Vrbo` · `Hotels.com` · `Omio` · `Flixbus` · hostel brand sites

Works headless (no phone needed):
`Airbnb` · `Hostelworld` · `Trip.com` (via `r.jina.ai`)

Blocked everywhere (needs the app):
`Skyscanner` web (PerimeterX) · `Kiwi.com`

## Gotchas worth the read

- `am start -a VIEW -d URL -p com.android.chrome` reuses the **foreground tab** — it once surfaced the user's payment page. Use `-n …chrome.Main`.
- `vision_analyze` coordinates come back on a scaled basis (e.g. 954×2048) while the screen is 1080×2340 — **scale by ~1.132** or you'll tap the wrong field (I typed `wwww` into a promo-code box this way).
- Black screenshots = screen asleep → `input keyevent KEYCODE_WAKEUP`.
- Booking / Agoda ignore date params in the URL and return "0 results". That is **not** "no availability" — drive the calendar UI.
- `Sold out` on one room type ≠ hotel sold out.
- Ratings without review counts are noise: `10 Superb (1 review)`.

---

## What's inside

```
SKILL.md                          # workflow, rules, pitfalls
references/platform-playbook.md   # 15+ sites: entry URLs, params, quirks
references/adb-recipes.md         # copy-paste adb commands + parsers
```

## License

MIT © 2026 Reese
