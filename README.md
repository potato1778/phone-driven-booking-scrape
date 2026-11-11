# phone-driven-booking-scrape

**Beat flight & hotel bot-walls by driving the user's real phone — not a headless browser.**

A Hermes Agent skill for scraping flight and hotel prices across OTAs when every automated browser gets blocked. Uses `adb` to drive the real Chrome app on an Android phone, so requests carry a genuine browser fingerprint, real cookies, and a residential IP — while leaving the user's desktop free.

---

## The problem

Price-comparison work on travel sites fails at the very first request:

| Site | What a headless browser gets |
|---|---|
| Booking.com | `0 properties found` (dates silently ignored) |
| Kayak | "KAYAK tror att du är en bot" full-page block |
| Agoda | `Sorry, search could not be completed` |
| Vrbo | `Show us your human side…` |
| Momondo / Skyscanner | PerimeterX "press and hold" challenge |
| Hostel brand sites | `Web Page Blocked! Attack ID: …` (WAF) |
| Omio | Cloudflare interstitial |

**The key insight: it is not an IP problem.** In the reference environment the automated browser's egress IP was the user's own home broadband — identical to the phone. The blocks come from **automation fingerprint + empty profile (no cookies, no session)**.

So the fix is not a proxy. The fix is **a real browser**.

## The approach

```
adb → am start (new tab in real Chrome) → uiautomator dump → parse text+bounds → tap by coordinate
```

- **Real Chrome** = real fingerprint, real cookies, real session
- **uiautomator dump** = full page text + pixel coordinates in one call, far cheaper than screenshots
- **Screenshots** only when layout/popup position matters
- **Batch taps** in a single shell call to cut round-trips

The user's computer stays free — everything runs on a phone sitting next to them.

## What's inside

```
SKILL.md                          # workflow, rules, pitfalls
references/platform-playbook.md   # 15+ sites: entry URLs, params, quirks
references/adb-recipes.md         # copy-paste adb commands + parsers
```

## Quick start

```bash
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$("$ADB" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1

# open a NEW tab (not the foreground one — see pitfalls)
"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html'"
sleep 20

# read the page as text + coordinates
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

1. **Never tap payment.** On any checkout page (`Pay now`, `Betala`, `Köp`, `预订`) — back out immediately and tell the user you did not touch it.
2. **Never type credentials or OTPs.** Hit a login wall → ask the user to log in themselves, or skip that site.
3. **Only trust the final all-in total.** Card prices lie (see below).

## The price trap that cost me a wrong answer

Vrbo's listing card said:

> `$91/night · $182 for 2 nights · All fees included`

The actual checkout breakdown:

```
$96.57 × 2      = $193.14
discount        = -$20.20
pre-tax         = $173.82
tax             = +$43.46
TOTAL           = $217.28   ← 19% above the card price
```

**Always open "Price details" and read `Total`.**

Per-platform unit conventions:

| Platform | Listing shows | Verify with |
|---|---|---|
| Hostelworld | **per night** (per bed / per room) | Add to cart → `SEK X x 2 nights` |
| Vrbo | **pre-tax** | Price details → Total |
| Booking.com | "2 nights" price, usually tax-inclusive | Checkout total |
| Agoda | **per night**, tax-inclusive | × nights |
| Airbnb | "total" incl. cleaning + fees | Price breakdown |

## Currency is not a free lunch either

Don't assume switching currency is cheaper — measure it. Compare the **local currency** price against the market rate:

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=DKK&symbols=SEK,USD,CNY"
```

Measured on Booking.com: `DKK 1,215` → `SEK 1,791.59` (implied 1.4746 vs market 1.4703, **+0.3% — no markup**), while the same room in `USD 191` was **4.5% more expensive**. The local-currency display was the cheapest.

## Platform support

Works through the phone (blocked in headless browsers):
`Booking.com` · `Agoda` · `Kayak` · `Momondo` · `Vrbo` · `Hotels.com` · `Omio` · `Flixbus` · hostel brand sites

Works headless (no phone needed):
`Airbnb` · `Hostelworld` · `Trip.com` (via `r.jina.ai`)

Blocked everywhere (needs the app):
`Skyscanner` web (PerimeterX) · `Kiwi.com`

## Gotchas worth the read

- `am start -a VIEW -d URL -p com.android.chrome` reuses the **foreground tab** — it once surfaced the user's payment page. Use `-n …chrome.Main`.
- `vision_analyze` coordinates come back on a scaled basis (e.g. 954×2048) while the screen is 1080×2340 — **scale by ~1.132** or you'll tap the wrong field (I typed `wwww` into a promo-code box this way).
- Black screenshots = screen asleep → `input keyevent KEYCODE_WAKEUP`.
- Booking/Agoda ignore date params in the URL and return "0 results". That is **not** "no availability" — drive the calendar UI.
- `Sold out` on a room type ≠ hotel sold out.
- Ratings without review counts are noise: `10 Superb (1 review)`.

## 中文说明

用 adb 驱动手机上的**真实 Chrome** 绕过反爬，抓机票/酒店的最终含税价、房型、行李额度，不占用电脑。

核心结论：**拦截不是 IP 问题，是指纹和会话问题**——自动化浏览器用的是空 profile + webdriver 特征。所以换代理没用，换真实浏览器才有用。

三条铁律：永不点付款、永不输入账号密码、只认最终含税总额。

详细方法见 `SKILL.md`，各平台入口和坑见 `references/platform-playbook.md`。

## License

MIT
