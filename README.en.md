<h1 align="center">phone-driven-booking-scrape</h1>

<p align="center">
  <b>English</b> | <a href="README.md">简体中文</a>
</p>

<p align="center">
  <b>Got an old Android phone gathering dust?</b><br>
  One computer, USB debugging on — and with wireless debugging you don't even need the cable.<br>
  Let that spare phone do the rounds of Booking, Agoda, Kayak, Vrbo, Skyscanner…<br>
  Comparing flight and hotel prices across every platform, without tying up your computer.
</p>

<p align="center">
  Bypass bot detection using the browser on a real device, rather than a headless one.
</p>

---

## Problem

Price comparison on travel sites typically fails at the first request:

| Site | What a headless browser gets |
|---|---|
| Booking.com | `0 properties found` (date parameters silently ignored) |
| Kayak | full-page "we think you're a bot" block |
| Agoda | "Sorry, search could not be completed" |
| Vrbo | `Show us your human side…` |
| Momondo / Skyscanner | PerimeterX press-and-hold challenge |
| Hostel brand sites | `Web Page Blocked! Attack ID: …` (WAF) |
| Omio | Cloudflare interstitial |

**Key finding: this is not an IP problem.** In the reference environment the automated browser's egress IP was identical to the phone's — the user's own home broadband. The blocks come from **automation fingerprinting plus an empty profile (no cookies, no session)**.

Changing proxy does not help. **Using a real browser does.**

## How it works

```
adb → am start (new tab in real Chrome) → uiautomator dump → parse text + bounds → tap by coordinate
```

- **Real Chrome** — genuine fingerprint, real cookies, real session
- **uiautomator dump** — whole page as text plus pixel coordinates in one call, an order of magnitude cheaper than screenshots
- **Screenshots** — only when layout or popup position must be judged
- **Batched taps** — merged into a single shell call to reduce round-trips

The desktop stays free throughout; all interaction happens on the phone.

## Quick start

```bash
# 1. Locate adb (the copy bundled with Escrcpy; installing scrcpy / Escrcpy provides it)
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$("$ADB" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1

# 2. Open a new tab in real Chrome (Main activity is required — see Known pitfalls)
"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html'"
sleep 20

# 3. Read the page as text plus coordinates
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

1. **Never tap payment.** On any checkout page (`Pay now`, `Betala`, `Köp`), exit immediately and state explicitly that nothing was touched.
2. **Never enter credentials or OTPs.** On a login wall, hand control back to the user or abandon that platform.
3. **Trust only the final all-in total.** Listing card prices carry systematic bias (see below).

## Price unit pitfalls

Vrbo's listing card displayed:

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
| Vrbo | **pre-tax** | Total in price details |
| Booking.com | "2 nights" price, usually tax-inclusive | Checkout total |
| Agoda | **per night**, tax-inclusive | × nights |
| Airbnb | "total", incl. cleaning and fees | Price breakdown |

## Currency selection

Do not assume a currency switch is cheaper — measure it. Compare the local-currency price against the market rate:

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=DKK&symbols=SEK,USD,CNY"
```

Measured on Booking.com: `DKK 1,215` → `SEK 1,791.59` (implied 1.4746 vs market 1.4703, **+0.3% — effectively no markup**); the same room quoted at `USD 191` was **4.5% higher**. Local currency was the lowest.

## Platform support

Works through the phone (headless is blocked):
`Booking.com` · `Agoda` · `Kayak` · `Momondo` · `Vrbo` · `Hotels.com` · `Omio` · `Flixbus` · hostel brand sites

Works headless (no phone required):
`Airbnb` · `Hostelworld` · `Trip.com` (via the `r.jina.ai` proxy)

Unavailable (app required):
`Skyscanner` web (PerimeterX) · `Kiwi.com`

## Known pitfalls

- `am start -a VIEW -d URL -p com.android.chrome` brings the **current foreground tab** forward; in testing this surfaced the user's payment page. Use `-n …chrome.Main` instead.
- `vision_analyze` returns coordinates on a scaled basis (commonly 954×2048) while the screen is 1080×2340 — **multiply by ~1.132**, or taps land on the wrong control (in testing this typed `wwww` into a promo-code field).
- An all-black screenshot means the screen is asleep → `input keyevent KEYCODE_WAKEUP`.
- Booking / Agoda ignore date parameters in the URL and return "0 results". This is **not** an absence of availability — drive the calendar control.
- `Sold out` on a single room type does not mean the property is sold out.
- Data with a rating but no review count is unreliable: `10 Superb (1 review)`.

## Does it work with an iPhone?

**`adb` does not support iOS** — but macOS + iPhone has a different route: Appium's **Safari Driver**. No WebDriverAgent, no Xcode build and signing, no certificate expiry.

| Setup | Available |
|---|---|
| Windows + Android | ✅ main path of this project |
| macOS + Android | ✅ same |
| macOS + iPhone | ✅ Safari Driver (see [`references/ios-safari-driver.md`](references/ios-safari-driver.md)) |
| Windows + iPhone | ❌ no viable channel |

iOS exposes no shell and no coordinate-injection interface. The only way to genuinely drive iOS Safari is Appium, and the `safaridriver` binary it wraps ships with macOS — so a Mac is required.

The methodology itself is platform-agnostic: the blocks come from automation fingerprinting plus an empty profile, not the IP. Real Safari on iOS works the same way, and `driver.page_source` hands you the real DOM, making price extraction more precise than Android's coordinate parsing.

**Windows + iPhone has no solution.** In that case treat the phone as a manual browser: the agent reads screenshots, checks price conventions and assembles the comparison, while a human performs the taps.

## Repository layout

```
SKILL.md                          # workflow, rules, pitfalls
references/platform-playbook.md   # 15+ sites: entry URLs, params, quirks
references/adb-recipes.md         # copy-paste adb commands and parsers
references/ios-safari-driver.md   # macOS + iPhone via Safari Driver (untested)
```

## License

MIT © 2026 Reese
