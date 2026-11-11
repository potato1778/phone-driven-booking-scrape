# iOS：macOS + iPhone 方案

> **状态说明**：本文依据 Appium 官方文档整理，**未在真机上实测**（编写环境无 macOS 设备）。
> 命令与设置项均来自下方引用的官方页面；实际执行时若遇版本差异，以官方文档为准。

Android 用 `adb` 直接驱动 Chrome。iOS 没有 `adb` 等价物，但**有另一条更省事的路径**：Appium 的 Safari Driver。

## 两条路线

| | Safari Driver（推荐） | XCUITest + WDA |
|---|---|---|
| 用途 | 网页内容自动化 | 网页 + 原生 UI |
| 需要 Xcode 编译 WDA | ❌ | ✅ |
| 需要 Apple 账号签名 | ❌ | ✅（免费账号可，手动配置） |
| 证书有效期 | 无此问题 | 免费账号需定期重签 |
| 需要 Developer Mode | ❌ | ✅（iOS 16+） |
| 能处理系统弹窗/权限框 | ❌ | ✅ |
| 适合比价抓取 | ✅ | 也可以，但重得多 |

**比价只需要网页内容**，所以 Safari Driver 足够，且绕开了 WDA 签名这一整套麻烦。

## Safari Driver 完整步骤

### 1. iPhone 上开启设置

```
设置 → App → Safari → 高级 → 网页检查器        ON
设置 → App → Safari → 高级 → 远程自动化          ON
```

（iOS 16 以下路径为 `设置 → Safari → 高级`）

### 2. Mac 上启用 safaridriver

```bash
safaridriver --enable
# 若从旧版 macOS 升级上来，可能需要 sudo
sudo safaridriver --enable
```

### 3. 安装 Appium 与驱动

```bash
npm i -g appium
appium driver install safari
appium driver list --installed
```

### 4. 连接设备

USB 连接 iPhone → 手机上点「信任此电脑」→ 保持解锁。

```bash
# 确认设备被识别
system_profiler SPUSBDataType | grep -A 5 -i iphone
```

### 5. 启动会话

```bash
appium server --port 4723
```

```python
from appium import webdriver
from appium.options.ios import SafariOptions

opts = SafariOptions()
opts.platform_name = "iOS"
opts.automation_name = "safari"
opts.set_capability("safari:useSimulator", False)
opts.set_capability("safari:deviceUDID", "<你的 iPhone UDID>")

driver = webdriver.Remote("http://127.0.0.1:4723", options=opts)
driver.get("https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html")
print(driver.title)
driver.quit()
```

UDID 获取：`xcrun xctrace list devices` 或「访达 → iPhone → 点序列号切换显示」。

## 比 Android 强的地方

**拿得到真 DOM。**

```python
html = driver.page_source          # 完整 HTML，不是无障碍树
```

Android 的 `uiautomator dump` 给的是文本 + 像素坐标，要自己解析 bounds。iOS 这边可以直接用 CSS 选择器：

```python
price = driver.find_element("css selector", ".prco-valign-middle-helper").text
```

对价格抓取来说，这比坐标点击精确得多 —— 不会因为布局微调而点到隔壁控件。

## 比 Android 弱的地方

| 差异 | 说明 |
|---|---|
| 只能操作网页 | 系统权限弹窗、App 内界面管不到；比价场景基本用不上 |
| Mac 必须开着 | Android 方案里电脑是空闲的；这边 Appium 跑在 Mac 上 |
| 需要 USB | `safaridriver` 走 USB 配对，不像 adb 可以无线调试 |
| 设备必须解锁 | 锁屏状态下无法建立会话 |

## 与 Android 方案的对应关系

| Android | iOS |
|---|---|
| `adb shell am start -d URL` | `driver.get(URL)` |
| `uiautomator dump` + 解析 bounds | `driver.page_source` + CSS 选择器 |
| `input tap X Y` | `element.click()` 或 W3C Actions |
| `input text` | `element.send_keys()` |
| `adb exec-out screencap` | `driver.get_screenshot_as_file()` |
| `input keyevent KEYCODE_BACK` | `driver.back()` |

三条硬性规则（不得点击付款、不得输入凭据、只认含税总额）与平台无关，照搬。

## 已知风险

- **iOS 大版本升级后驱动可能滞后**：`safaridriver` 随 macOS 更新，iPhone 与 Mac 的版本差太大时可能连不上。
- **免费 Apple 账号的 7 天限制只影响 XCUITest/WDA 路线**，Safari Driver 不受影响。
- **页面加载等待**：Safari Driver 对 SPA 的加载完成判定不如 `uiautomator` 直观，建议显式 `WebDriverWait` 等目标元素出现，不要靠 `sleep`。
- 若站点弹系统级「允许定位」等提示，Safari Driver 无法自动处理，需手动点掉或改用 XCUITest 路线。

## 参考

- [Appium Safari Driver — Non-macOS Devices](https://appium.github.io/appium-safari-driver/v5/guides/non-macos/)
- [Appium XCUITest Driver — Device Preparation](https://appium.github.io/appium-xcuitest-driver/latest/getting-started/device-setup/)
- [Appium XCUITest Driver — Provisioning Profile](https://appium.github.io/appium-xcuitest-driver/latest/getting-started/provisioning-profile/)
- [Apple — Enable WebDriver on macOS](https://developer.apple.com/documentation/safari-developer-tools/macos-enabling-webdriver)
