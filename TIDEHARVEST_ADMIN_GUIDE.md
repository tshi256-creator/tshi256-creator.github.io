# TideHarvest 管理员操作手册

## 系统架构概览

| 组件 | 说明 |
|---|---|
| 前端 | `index.html` — 订阅表单 + 留言板，托管于 GitHub Pages |
| 订阅数据 | Google Sheets `TideHarvest Subscribers` → `Subscribers` 标签页 |
| 建议数据 | Google Sheets `TideHarvest Subscribers` → `Suggestions` 标签页 |
| 规则配置 | Google Sheets → `Species Rules` + `Location Rules` 标签页 |
| 自动推送 | Google Apps Script，三个触发器自动运行 |

---

## 邮件推送规则

| 邮件类型 | 触发时间 | 内容 |
|---|---|---|
| 月度预报 | 每月1号 6-7am | 未来30天潮汐+季节符合的日期 |
| 每周毒素+Razor Clam | 每周一 6-7am | DOH毒素状态 + WDFW Razor Clam开放日 |
| 新订阅欢迎邮件 | 表单提交后立即 | 当前毒素状态 + 本周Razor Clam + 订阅日到月底的符合日期 |

**重要说明：**
- 月度预报**不包含**毒素状态，因为毒素每周变化
- 毒素状态每周一单独推送，用户出发前仍需自查 DOH 地图
- Razor Clam 只推送给订阅了 `razor` 物种的用户

---

## Apps Script 触发器设置

打开 Google Sheets → Extensions → Apps Script → 左侧点 ⏱ Triggers → 右下角 + Add Trigger

**触发器 1 — 月度预报**
- Function: `monthlyTideForecast`
- Event source: `Time-driven`
- Type: `Month timer`
- Day: `1`
- Hour: `6am to 7am`

**触发器 2 — 每周毒素+Razor Clam**
- Function: `weeklyBiotoxinAndRazor`
- Event source: `Time-driven`
- Type: `Week timer`
- Day: `Every Monday`
- Hour: `6am to 7am`

**触发器 3 — 新订阅欢迎邮件**
- Function: `onNewSubscriber`
- Event source: `From spreadsheet`
- Event type: `On form submit`
- 注意：需选择订阅表单对应的 Sheet（Subscribers）

---

## 新增地点操作流程

### 第一步：确认 DOH 是否有记录

1. 打开 [DOH Shellfish Safety Map](https://fortress.wa.gov/doh/biotoxin/biotoxin.html)
2. 找到目标海滩，点击
3. 查看弹窗：
   - 显示海滩名（如 `DOSEWALLIPS SP`）→ DOH 有记录，可以自动查毒素
   - 显示 `No Public Shellfish Beach Found` → DOH 无记录，毒素只能手动自查

### 第二步：在 Location Rules 表里添加一行

| 字段 | 说明 | 示例 |
|---|---|---|
| `location_id` | 短代码，小写，前端下拉菜单用 | `dosewallips` |
| `display_name` | 显示名称 | `Dosewallips State Park` |
| `doh_beach_name` | DOH 地图弹窗里的海滩名（大写） | `DOSEWALLIPS SP` |
| `noaa_station` | NOAA 潮汐站编号 | `9447130` |
| `biotoxin_check` | DOH 有记录填 `yes`，否则填 `no` | `yes` |
| `notes` | 备注 | `Hood Canal shellfish beach` |

**NOAA 站点查询：** [tidesandcurrents.noaa.gov](https://tidesandcurrents.noaa.gov/map/) 找最近的潮汐站

### 第三步：在前端 index.html 里添加下拉选项

找到 `renderLocationSelect` 函数，在 locations 数组里添加：
```javascript
{ id: 'dosewallips', label_en: 'Dosewallips State Park', label_zh: '...' }
```

### 第四步：验证

1. 在 Apps Script 里跑 `weeklyBiotoxinAndRazor`
2. 确认邮件里该地点的毒素状态显示正确
3. 如果显示 "unavailable"，检查 `doh_beach_name` 是否和 DOH 地图弹窗一致

---

## 新增物种操作流程

### 第一步：在 Species Rules 表里添加一行

| 字段 | 说明 | 示例 |
|---|---|---|
| `id` | 短代码，小写 | `geoduck` |
| `check_type` | `hardcoded`（季节固定）或 `scrape_wdfw`（需抓取WDFW） | `hardcoded` |
| `season_months` | 开放月份，逗号分隔，或 `all` | `1,2,3,10,11,12` |
| `wdfw_url` | WDFW 物种页面链接 | `https://wdfw.wa.gov/...` |
| `notes` | 备注 | `Hood Canal only` |

**check_type 说明：**
- `hardcoded` — 季节固定，只检查月份
- `scrape_wdfw` — 需要抓取 WDFW 页面（目前只有 Razor Clam 用这个）

### 第二步：在前端 index.html 里添加下拉选项

找到 `renderSpeciesSelect` 函数，在 species 数组里添加。

---

## DOH 毒素查询说明

系统使用 DOH ArcGIS REST API（Layer 3）查询毒素状态：
- API：`https://fortress.wa.gov/doh/arcgis/arcgis/rest/services/Biotoxin/Biotoxin_v2/MapServer/3`
- 查询字段：`BEACHNAME`
- 状态字段：`GISDB.sde.vBeachStatus.finalstatus`（`Closed` 或 `Open`）

**已确认可查询的地点：**
- `WOLFE PROPERTY SP` ✅

**DOH 无记录（biotoxin_check = no）：**
- Luhr Beach（Nisqually Reach，非公共贝类海滩）
- KVI Beach
- Twin Harbors、Long Beach、Copalis（海岸 Razor Clam 专属，毒素由 WDFW 单独管理）

---

## 常见问题排查

**邮件没有发出？**
1. 检查 Apps Script 触发器是否设置
2. 检查 Subscribers 表是否有数据
3. 手动跑对应函数，查看 Execution Log

**毒素状态显示 "unavailable"？**
1. 打开 DOH 地图点击该海滩
2. 确认弹窗里的海滩名
3. 更新 Location Rules 表的 `doh_beach_name`（必须大写）

**NOAA 潮汐数据获取失败？**
- 检查 `noaa_station` 编号是否正确
- 访问 `https://api.tidesandcurrents.noaa.gov` 确认站点可用

**Razor Clam 开放日找不到？**
- WDFW 页面格式可能变更
- 手动访问 `https://wdfw.wa.gov/fishing/shellfishing-regulations/razor-clams` 确认

---

## 重要链接

| 资源 | 链接 |
|---|---|
| DOH Shellfish Safety Map | https://fortress.wa.gov/doh/biotoxin/biotoxin.html |
| DOH Hotline (24/7) | 1-800-562-5632 |
| WDFW Razor Clam | https://wdfw.wa.gov/fishing/shellfishing-regulations/razor-clams |
| WDFW Public Beaches | https://wdfw.wa.gov/places-to-go/shellfish-beaches |
| NOAA Tides Map | https://tidesandcurrents.noaa.gov/map/ |
| DOH ArcGIS API | https://fortress.wa.gov/doh/arcgis/arcgis/rest/services/Biotoxin/Biotoxin_v2/MapServer/3 |

