# ADB 操作速查

## 环境

```bash
ADB="/c/Program Files/Escrcpy/resources/extra/win/scrcpy/adb.exe"
S=$(\"$ADB\" devices | awk 'NR==2{print $1}')
export MSYS_NO_PATHCONV=1
OUT="C:/Users/<user>/AppData/Local/hermes/cache/scratch/fl2"
mkdir -p "$OUT"
```

`adb` 不在 PATH 里时，用 Escrcpy 自带的那个。

## 开页面（新标签）

```bash
"$ADB" -s $S shell "am start -n com.android.chrome/com.google.android.apps.chrome.Main \
  -a android.intent.action.VIEW -d 'URL'"
sleep 20   # 首屏 18-25s，SPA 更久
```

## 读文本（首选）

```bash
"$ADB" -s $S shell "uiautomator dump /sdcard/ui.xml >/dev/null 2>&1"
"$ADB" -s $S pull /sdcard/ui.xml "$OUT/ui.xml" >/dev/null 2>&1
```

```python
import re
x = open(r".../ui.xml", encoding="utf-8", errors="ignore").read()
seen = []
for m in re.finditer(r'(?:text|content-desc)="([^"]{1,100})"[^>]*bounds="\[(\d+),(\d+)\]\[(\d+),(\d+)\]"', x):
    t, a, b, c, d = m.groups()
    if t.strip() and t not in [s[0] for s in seen]:
        seen.append((t, (int(a)+int(c))//2, (int(b)+int(d))//2))
for t, x1, y1 in seen[:60]:
    print(f"{t!r} @ {x1},{y1}")
```

## 点击 / 输入 / 滑动

```bash
"$ADB" -s $S shell "input tap 540 1200"
"$ADB" -s $S shell "input swipe 540 1800 540 700 400"      # 上滑一屏
"$ADB" -s $S shell "input swipe 540 1700 540 2250 300"      # 下滑关底部弹窗
"$ADB" -s $S shell "input keyevent KEYCODE_BACK"
"$ADB" -s $S shell "input keyevent KEYCODE_WAKEUP"
```

## 输入文字

```bash
IME=$("$ADB" -s $S shell "settings get secure default_input_method" | tr -d '\r')
"$ADB" -s $S shell "ime disable $IME" >/dev/null 2>&1
"$ADB" -s $S shell "input text 'Harbin'"
"$ADB" -s $S shell "ime enable $IME; ime set $IME" >/dev/null 2>&1
```

禁用输入法可以避免弹出键盘遮挡页面。

## 截图

```bash
"$ADB" -s $S shell "screencap -p /sdcard/_s.png"
"$ADB" -s $S pull /sdcard/_s.png "$OUT/x.png" >/dev/null 2>&1
"$ADB" -s $S shell "rm -f /sdcard/_s.png"
```

黑图 = 屏幕睡了 → `KEYCODE_WAKEUP` + 上滑。

## 按文本找坐标再点（最稳）

```bash
python - <<'EOF' > /tmp/xy.txt
import re
x = open(r".../ui.xml", encoding="utf-8", errors="ignore").read()
for m in re.finditer(r'(?:text|content-desc)="(Search)"[^>]*bounds="\[(\d+),(\d+)\]\[(\d+),(\d+)"', x):
    t, a, b, c, d = m.groups()
    print((int(a)+int(c))//2, (int(b)+int(d))//2)
EOF
XY=$(head -1 /tmp/xy.txt)
[ -n "$XY" ] && "$ADB" -s $S shell "input tap $XY"
```

## 日历日期（Agoda / 阿普汉森 / Vrbo 通用）

```bash
python - <<'EOF' > /tmp/d13.txt
import re
x = open(r".../ui.xml", encoding="utf-8", errors="ignore").read()
for m in re.finditer(r'content-desc="Fri Nov 13 2026"[^>]*bounds="\[(\d+),(\d+)\]\[(\d+),(\d+)"', x):
    a, b, c, d = m.groups()
    print((int(a)+int(c))//2, (int(b)+int(d))//2)
EOF
```

先点入住日，**重新 dump**，再点退房日——日历会在第一次点击后重排。

## 坐标缩放

vision_analyze 报告的坐标基准常见是 **954×2048**，实际屏幕 **1080×2340**：

```
实际 x = vision_x * 1080 / 954
实际 y = vision_y * 2340 / 2048
```

不确定时用 uiautomator 的 bounds，它是真实像素。
