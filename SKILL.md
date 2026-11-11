---
name: phone-driven-booking-scrape
description: "Use when 机票酒店比价被反爬拦截，改用手机 adb 抓取。"
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [Travel, Scraping, ADB, AntiBot, Flights, Hotels]
    related_skills: [flight-fare-hunting, budget-trip-planning, product-price-monitor, android-adb-probe-and-control]
---

# 手机驱动的机票 / 酒店比价

用 adb 驱动用户手机上的**真实 Chrome** 绕过风控，拿到多平台的最终含税价、房型、行李额度。不占用用户电脑。

## When to Use

- 自动化浏览器被 Cloudflare / PerimeterX / WAF 挡住（Booking、Agoda、Kayak、Vrbo、Momondo、青旅官网）
- 要在多个 OTA 之间比价，且**不能占用用户电脑**
- 必须拿到**最终含税总额**，而不是列表卡片上的宣传价
- 必须确认**房型（几张床 / 共用还是独立卫浴 / 面积）**和**机票行李额度**

## 核心原理

真实 Chrome + 真实 cookies + 住宅 IP = 过风控。

自动化浏览器（patchright / agent-browser）用的是**空 profile + webdriver 指纹**，Booking 会直接返回「0 家住宿」，Kayak 会返回「我们认为你是 bot」整页，青旅官网返回 `Web Page Blocked! Attack ID`。

**关键认知：出口 IP 往往就是用户家宽带，拦截不是 IP 问题，是指纹和会话问题。** 所以「换代理」没用，「换真实浏览器」才有用。

## 三条铁律

1. **永不点付款**。出现 `Betala` / `Köp` / `Pay now` / `预订` 的结账页，立刻 `KEYCODE_BACK` 退出，并告诉用户「我没碰付款页」。
2. **永不输入账号、密码、验证码**。遇到登录墙，让用户自己登，或者放弃该平台。
3. **价格只认最终含税总额**。列表卡片价、`All fees included` 标语都可能是税前价（见下）。

## 标准流程

### 1. 找 adb

```bash
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$(\"$ADB\" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1
```

### 2. 开页面（必须用 Main activity）

```bash
"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'https://www.booking.com/...'"
```

**不要**只用 `am start -a VIEW -d URL -p com.android.chrome`——Chrome 会把**已经打开的旧标签**提到前台（曾把用户的付款页翻出来）。加 `-n ...chrome.Main` 才是新标签。

### 3. 读页面：优先 uiautomator，不用截图

```bash
"$ADB" -s $S shell "uiautomator dump /sdcard/ui.xml >/dev/null 2>&1"
"$ADB" -s $S pull /sdcard/ui.xml /tmp/ui.xml >/dev/null 2>&1
```

解析（同时拿文本和坐标）：

```python
import re
x = open("/tmp/ui.xml", encoding="utf-8", errors="ignore").read()
for m in re.finditer(r'(?:text|content-desc)="([^"]{2,90})"[^>]*bounds="\[(\d+),(\d+)\]\[(\d+),(\d+)\]"', x):
    t, a, b, c, d = m.groups()
    print(f"{t!r} @ {(int(a)+int(c))//2},{(int(b)+int(d))//2}")
```

**一次 dump 能拿到整页文字，比截图 + vision 便宜一个数量级。** 只有需要判断布局、弹窗位置、颜色状态时才截图。

### 4. 截图兜底

```bash
"$ADB" -s $S shell "screencap -p /sdcard/_s.png"; "$ADB" -s $S pull /sdcard/_s.png "$OUT/x.png"; "$ADB" -s $S shell "rm -f /sdcard/_s.png"
```

**黑图 = 屏幕睡了**，先 `input keyevent KEYCODE_WAKEUP` 再截。

### 5. 批量操作省调用

一次 shell 里串多个 tap + sleep，只在关键节点截图/dump。例：

```bash
for i in 1 2 3 4; do "$ADB" -s $S shell "input tap 979 1283"; sleep 1.5; done   # 日期 +4 天
"$ADB" -s $S shell "input tap 540 1728"; sleep 18                                  # 搜索
```

## 坐标陷阱（最容易翻车）

- **vision 报的坐标是缩放后的基准**（常见 954×2048），实际屏幕是 **1080×2340**。把 vision 给的坐标 × 1.132 再 tap，否则会点到旁边的输入框（实测把「成人数量 −」点到优惠码框里，输入了一串 `wwww`）。
- **uiautomator 的 bounds 是真实像素**，优先用它，不要用 vision 估。
- 懒加载列表（Hostelworld / Booking 房型）在 dump 里可能是 `bounds="[0,0][0,0]"`，先 `window.scrollTo` / `input swipe` 再 dump。
- 弹窗出现后布局会位移，**每次状态变化重新 dump**，不要复用旧坐标。

## 价格口径：必须验证，不能猜

| 平台 | 房源页显示 | 验证方法 |
|---|---|---|
| **Hostelworld** | **每晚**价（per bed / per room） | 点 Add 进购物车，看 `SEK X x 2 nights` 与 Total |
| **Vrbo** | 卡片价**税前** | 打开「价格详情 / Price details」看 **Total**（含 tax） |
| **Booking** | 房型列表的「2晚价格」通常**已含税** | 进结账页看「总共」 |
| **Agoda** | **每晚**含税价 | ×晚数才是总额 |
| **Airbnb** | 卡片「总价」= 已含清洁费税费 | 点「价格明细」确认 |

**实测案例（务必记住）**：Vrbo 卡片写 `$91/晚 · $182 for 2 nights · All fees included`，结账明细是 `$96.57 × 2 = $193.14 − 折扣 $20.20 = 税前 $173.82 + 税 $43.46 = 最终 $217.28`。**卡片价和最终价差 19%。**

## 货币陷阱

- 不同币种可能对应不同价格档。**不要假设哪个便宜，实测**。
- 判断有没有加价：拿**当地货币价**（如 DKK 1,215）按市场汇率换算，和展示价对比。
  ```bash
  curl -s "https://api.frankfurter.dev/v1/latest?base=DKK&symbols=SEK,CNY,USD"
  ```
- 实测：Booking 的 SEK 价 = DKK 直接换算（隐含汇率 1.4746 vs 市场 1.4703，+0.3%，等于没加价）；**USD 反而贵 4.5%**。Hostelworld 的 SEK 与 EUR 差 0.6 kr，也无加价。
- 结论：**先比当地货币价，再决定用哪个币种结算**，别默认「切美元更便宜」。

## 信息颗粒度（用户明确要的）

拿到价格只是及格线。每条报价至少补齐：

**酒店**
- 房型全名（`Studio, 2 Single Beds, Kitchenette` ≠ `Studio, 1 Double Bed`）
- 几张床、什么床（双人床 / 两张单人床 / 上下铺 / 胶囊）
- 卫浴：私人 / 共用（`privat badrum` / `shared bathroom`）
- 面积（m² / sq ft）、是否带厨房
- 评分 **+ 评价条数**（`10 Superb (1 review)` 不可信，<50 条的标注出来）
- 距市中心距离、到最近地铁/车站的步行时间
- 取消政策（免费取消截止日 / 不可退）
- 是否含早餐、早餐加价

**机票**
- 每段航班号、机型、起降时刻、中转时长
- **行李**：托运几件 × 多少 kg、随身几件；注意「1 handbagage · 0 incheckade väskor」这类原文
- **是否联程票**：`Anslutningen genomförs av X`（航司负责中转）= 联程；`Egen transfer` / `Separat biljett` = 自拼，行李不直挂
- 出票方（Sembo / Travellink / Opodo…），同班机不同出票方可能差 1000+ kr

## 平台速查

每个站点的入口、参数、坑 → `references/platform-playbook.md`  
adb 命令与坐标解析代码 → `references/adb-recipes.md`

## 交付

- 价格表**标时间戳**（40 分钟内会变）
- 明确写「未取到」而不是「没有」——被拦 ≠ 没货
- 给**二选一**结论 + 差价折算人民币
- 有文件产物时输出 `MEDIA:/绝对路径`

## 陷阱清单

- Chrome 有几十个标签，`am start` 可能翻出旧页（付款页/过期搜索页）→ 用 Main activity
- 屏幕超时变黑 → `KEYCODE_WAKEUP`
- Google 登录 sheet 会盖住页面 → 按提示向下滑关闭，或找右上角 X，**不要登录**
- Booking 的 URL 日期参数经常被忽略，返回「0 家住宿」→ 必须驱动 UI 选日期
- Agoda 的 URL 参数同样无效 → 点日历格子，`content-desc="Fri Nov 13 2026"` 可直接定位
- 网站显示 `Slutsålt hos oss` / `Sold out` 只针对**该房型**，不代表整店没房
- 同一酒店不同房型差价能到 2 倍（宿舍床位 vs 私人房），**先确认要哪种再比价**

## 用户偏好（本技能默认）

- 少问多做：能自己查的就别问，一轮给出完整结果
- 遇到拦截自动换手机端，不要停下来征求许可
- 结论先行，理由在后
- 数字必须标来源和抓取时间
