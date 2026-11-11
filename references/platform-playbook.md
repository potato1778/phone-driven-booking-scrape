# 平台手册：入口、参数、坑

## 机票

### Kayak（kayak.se）
- 手机 Chrome 可用（需登录才有会员价，能便宜 1000+ kr）
- URL：`https://www.kayak.se/flights/GOT-HRB/2026-11-18`
- **列表页只显示每班机最低价**；点开卡片才出现 `Bokningsalternativ` 出票方列表
- 底部固定显示最便宜那档的行李：`1 handbagage · 0 incheckade väskor`
- 顶部 `egen transfer` 是 Kayak 对「多航司 + OTA 出票」的保守提示，**不代表真的自拼**，要去 OTA 页面确认
- 第一张卡常是 `Annons`/`Sponsrad` 广告，往下找带 `Bäst`/`Billigast` 标签的

### Momondo（momondo.se）
- 与 Kayak 同集团，价格通常差 40 kr 以内，可作交叉验证
- URL：`https://www.momondo.se/flight-search/GOT-HRB/2026-11-18?sort=price_a`

### Trip.com
- 浏览器 `r.jina.ai` 代理可读，适合批量抓日期
  ```
  https://r.jina.ai/https://www.trip.com/flights/showfarefirst?dcity=got&acity=hrb&ddate=2026-11-18&triptype=ow&class=y&quantity=1&locale=en-US&curr=SEK
  ```
- 卡片末尾的 `Included` = 含托运行李
- 价格通常比 Kayak 会员价贵 1200–1400 kr

### Skyscanner App
- 网页版有 PerimeterX「按住 3 秒」，绕不过
- App 表单会莫名重置成「多城市」，需要反复重设
- 价格普遍比 Kayak 贵 1400+ kr，优先级最低

### Expedia App
- 需要装 App（`com.expedia.bookings`），用港币/美元计价时更便宜
- 网页版直接 `Tillgången är tillfälligt begränsad`

### Kiwi.com
- 网页版在本机不通，App 搜不到
- 只能靠手机浏览器日历读全月价，通常贵 950+ kr

### Omio（火车/大巴/飞机聚合）
- 云端 Cloudflare 拦，**手机端可用**
- URL：`https://www.omio.com/trains/gothenburg/copenhagen`
- 结果页顶部有三档概览（火车/巴士/飞机的最低价+时长）
- **日期条直接显示每个日期的最低价**（`Fri Nov 13 kr 372`），一次就能拿到整周价格
- 底部 `Cheapest SJ Price` / `Cheapest Oresundstag Price` 按运营商分列

### Flixbus
- 路由页：`https://global.flixbus.com/bus-routes/bus-gothenburg-copenhagen`
- 表单默认「今天」，用 `Select next day` 箭头逐日推进
- 结果页货币常是 **EUR**，记得换算
- 卡片标 `LOWEST PRICE` 的就是最便宜班次

## 酒店

### Hostelworld
- 浏览器端（非无头）可用，`document.body.innerText` 能读到房型+价格
- 城市页：`https://www.hostelworld.com/hostels/europe/denmark/copenhagen/?from=2026-11-13&to=2026-11-15&guests=2`
- 房源页：`https://www.hostelworld.com/pwa/hosteldetails.php/<Slug>/Copenhagen/<id>?from=...&to=...&guests=2`
- 城市页的 `Privates From` / `Dorms From` 是**每人每晚**
- 房源页的 `Prices are per bed` / `per room` 是**每晚**价，购物车写 `SEK X x 2 nights`
- **收 15% 定金**，尾款到店付
- 有 `Female Dorm` 房型，和混合宿舍同价，优先给女生用户

### Booking.com
- 手机端可用，云端 `web_extract` 返回 `SOURCE_NOT_AVAILABLE`，云端浏览器返回 0 家
- **URL 里的日期参数经常被忽略**，会显示默认日期 + 「0 家住宿」→ 必须驱动 UI
- 手机 URL（酒店页）可直接带参数：
  `https://www.booking.com/hotel/dk/cabinn-copenhagen.en-gb.html?checkin=2026-11-13&checkout=2026-11-15&group_adults=2&no_rooms=1&selected_currency=SEK`
- 房型列表的「2晚价格」+「价格包含税费及其他费用」= 全包价
- 会弹 Google 登录 sheet，**向下滑关闭**，不要登录

### Agoda
- 手机端可用，云端搜索接口返回「无法完成搜索 / 找到 0 个住宿」
- **URL 参数（checkIn/los/checkInYear…）全部无效**，必须点日历
- 日历格子有 `content-desc="Fri Nov 13 2026"`，可直接正则定位点击
- 价格是**每晚含税**，×晚数
- 便宜档全是 2–4 分、1–12 条评价的民宿/车屋，要按评价数过滤

### Vrbo
- 云端返回 `Show us your human side`，**手机端可用**
- URL：`https://www.vrbo.com/search?destination=Copenhagen%2C%20Denmark&startDate=2026-11-13&endDate=2026-11-15&adults=2`
- **卡片价是税前**！必须打开「价格详情」看 Total
- 货币选择器在右上/底部，切 USD/EUR 后价格档可能不同，实测 SEK 与 USD 换算后基本一致

### Hotels.com
- 手机端可用，但**目的地要写 `Copenhagen, Denmark`**，只写 `Copenhagen` 会搜到美国纽约州的 Copenhagen
- 有 Google 登录弹窗，找右上角 X 关闭
- 价格标「inklusive skatter och avgifter」= 含税

### Airbnb
- **云端浏览器可直接用**（少数不被拦的平台）
- URL：`https://www.airbnb.com/s/Copenhagen--Denmark/homes?checkin=2026-11-13&checkout=2026-11-15&adults=2&currency=SEK`
- 卡片上的日期标签（如「11月13日至14日」）表示**该房源可订区间**，不是搜索区间——必须点进去用 `?check_in=&check_out=` 验证
- 页面显示「添加日期以查看价格」= 该日期段不可订
- 房型 `房间` = 独立房间但常与房东共用厨卫；`公寓/整套` = 独立空间
- 详情页找「共用卫生间」「房东住在房源内」「未配备烟雾报警器」这些安全信息

### 青旅官网（Arp-Hansen / Steel House / Next House）
- 云端 WAF 直接 `Web Page Blocked!`，**手机端可用**
- 首页底部 `Get prices` → 预订引擎（5 步流程）
- `Select dates` 打开日历，`content-desc` 里是 `Fri Nov 13 2026`
- 成人数量 +/− 按钮坐标容易点错，**每次点完 dump 确认数值**
- 官网常写 `Prices from DKK 145`，是床位起价，和 Hostelworld 基本一致

## 火车

### SJ（sj.se）
- 学生 15% off（二等座），需 Mecenat / STUK / ISIC + 带照片证件
- 提前订便宜；出发前 24h 有 sista minuten 放票
- 官网买才吃学生折扣，Omio 上是全价

### Öresundståg（oresundstag.se）
- 有学生票，需 Mecenat 里的「Studentresesymbolen」
- GOT→CPH 直达 3h33，固定票价（不打折时 515–585 kr）

## 汇率

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=SEK&symbols=CNY,EUR,USD,DKK"
```
