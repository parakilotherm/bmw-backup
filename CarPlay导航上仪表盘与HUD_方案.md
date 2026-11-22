# CarPlay 导航上仪表盘 / HUD
## 宝马 X3（G01）iDrive 7 / MGU / 07/21

> 目标：CarPlay 连接后，把高德等导航显示到仪表盘，最好也上 HUD。
> 结论：**仪表盘先试原生路线，免费、零风险；HUD 是另一回事，07/21 原厂只支持 BMW 导航和 Apple 地图，第三方导航上 HUD 需要非常规手段。**

---

## 0. 先把项目用对

你最初发的 `1031527554/idrive7-apps-zh` 是**自定义 App**项目（车辆数据、性能计时、诊断、氛围灯），**不做 CarPlay 投屏**。真正相关的是另一个项目：

- `robertnisipeanu/idrive-apps` 里的 **Maps on cluster**：把 CarPlay / Android Auto 的导航画面投到仪表盘和 HUD。

但注意：**仪表盘的高德投屏，2024 年起高德已经原生支持，不需要 root，也不需要这个项目。** 先试原生的，不要一上来就刷机。

---

## 0.5 你发的三条小红书笔记，核对结果

三条笔记我都打开看了。它们不是同一个项目，也不是同一件事，其中最关键的一条需要更正一下：

| 笔记 | 实际内容 | 用的是哪个项目 | 和你目标的差距 |
|---|---|---|---|
| **「id7 carplay映射仪表盘，尝尝鲜」**（11-01，江苏） | 演示 CarPlay 地图上仪表盘 | `robertnisipeanu/idrive-apps`（Maps on cluster） | **演示的是苹果自带地图，不是高德**。作者自己在评论区回复：「我是指 carplay 的苹果自带地图 不是高德 我没说清」。全文没提 HUD，也没写车型/版本 |
| **「纯小白靠 AI 装 ID7 组件 carplay 仪表投屏」** | 个人踩坑记录，不是教程 | 同样是社区 patch + ID7 组件 | 有 CarPlay 仪表投屏，也有别的小组件；没给可复现步骤，没提高德/HUD |
| **第三条** | 自定义 App 的安装演示 | `cauliflowermossgrape/idrive7-8-ssh-root` + `idrive7-apps`（就是你最初发的那个中文 fork 的上游） | **这是装 App，不是 CarPlay 投屏**，和仪表盘/HUD 无关 |

所以「他们都是通过上面的 git repo 实现」这句要拆成两半：

- 第 1、2 条走的是**另一个 repo**（`robertnisipeanu/idrive-apps`），不是你最初发的 `idrive7-apps-zh`；
- 第 3 条才是 `idrive7-apps-zh`，但它做的是车机 App，不碰 CarPlay 投屏。

**这三条笔记能证明的是**：社区 patch 的视频链路在部分 ID7 车上确实跑通了，CarPlay 地图能进仪表盘。
**不能证明的是**：高德能走这条路、07/21 能跑、HUD 能用。这三点目前都还是未验证项，得自己上车试。

---

## 1. 结论表

| 目标 | 07/21 X3 是否可行 | 需要什么 | 风险 |
|---|---|---|---|
| 中控屏显示高德 CarPlay | 一直可以 | 无 | 无 |
| **仪表盘显示高德地图** | **大概率可以，原生** | iPhone 12+、iOS 17.4+、高德最新版、车机支持 CarPlay 仪表盘投屏 | 无 |
| 仪表盘显示转弯箭头/车道信息 | 高德原生只保证地图；箭头/车道看车机 | 同上 | 无 |
| **HUD 显示高德全地图** | 原厂不行 | 社区 patch（root） | 高：清编码、Digital Key、boot loop |
| HUD 显示高德转弯箭头 | 原厂不行 | 升级 i-Level 到 2025+（高德未确认）或社区 patch | 中-高 |
| HUD 显示 Apple 地图箭头/车道 | **原生可以** | iOS + Apple 地图导航 | 无 |
| HUD 显示 BMW 原车导航 | 原生可以 | 用 BMW 导航 | 无 |

---

## 2. 先试免费原生路线（30 分钟，零风险）

### 2.1 条件

- **iPhone 12 或更新**（高德官方要求；老 iPhone 不支持这个投屏能力）。
- **iOS 17.4 或更新**。
- **高德地图更新到最新版**。
- 车机支持 CarPlay 仪表盘投屏。宝马在支持品牌列表里；07/21 属于较老车机，可能显示完整地图，也可能只显示简化引导。

### 2.2 步骤

1. iPhone 升级 iOS，高德升级到最新版。
2. 连接 CarPlay（有线/无线均可）。
3. 在高德里设好目的地，**开始导航**。
4. 在方向盘右侧按**齿轮状/菜单按键**，把「显示内容」切到「**地图**」。
   - 或在 iDrive 里找：**仪表盘 > 中央显示区域 > 地图视图**（不同 i-Step 措辞略有差异）。
5. 看仪表盘中央是否出现高德地图。

### 2.3 如果出现黑块/卡顿

这是已知问题。按顺序试：

- 重新连接 CarPlay；
- 杀掉高德重开；
- 长按音量旋钮约 30 秒，重启车机。

### 2.4 如果仪表盘不显示

先做对照实验：

1. 用 **Apple 地图**导航，看仪表盘是否显示。Apple 地图从 2020-07 的 BMW OTA 起就支持仪表盘/HUD 的转弯和车道信息。
2. 如果 Apple 地图能显示、高德不能 → 先确认高德/iOS 已最新；仍不行就是高德在这个车机上的适配问题，等更新。
3. 如果 Apple 地图也不能显示 → 车机的 CarPlay 仪表盘流没开/不支持。可以考虑升级 i-Level，或走社区 patch。

---

## 3. HUD 的现实

### 3.1 07/21 X3 原厂 HUD 支持什么

- **BMW 原车导航**：支持。
- **Apple 地图（CarPlay）**：支持转弯指示和车道信息（BMW 2020-07 OTA 加入）。
- **高德 / 百度等第三方 CarPlay 导航**：**不支持**。2021 年 X3 论坛的实测结论就是：HUD 只吃车机导航和 iPhone 自带导航，高德/百度不行。
- 2025 年 07 月的 i-Level 更新后，有车主报告 **Google 地图**的转弯提示上了 HUD；高德是否跟进未确认。

### 3.2 所以 HUD 有四条路

| 方案 | 效果 | 代价/风险 | 建议 |
|---|---|---|---|
| A. HUD 用 Apple 地图，仪表盘用高德 | HUD 有转弯/车道；仪表盘有高德地图 | 无 | **最推荐**，先这么用 |
| B. 升级 i-Level 到 2025+ | 可能让 Google 地图上 HUD；高德未知 | 4S 时间/费用；升级后 root/App 全失效 | 想长期用可以，但别只为高德 |
| C. 社区 patch（Maps on cluster） | 高德全地图上 HUD（实验性） | 要 root；可能清编码、删 Digital Key、boot loop | 只给能接受风险的人 |
| D. 外接 HUD / 手机 HUD | 独立小屏，不是原车 HUD | 额外设备、走线 | 不想动车机就选这个 |

---

## 4. 如果一定要高德上 HUD：社区 patch 路线

项目：`robertnisipeanu/idrive-apps` → **Maps on cluster**。

### 4.1 原理

不是把高德的“导航数据”接进车机，而是：

1. 给 `/usr/bin/carplay` 打一个可逆补丁，解开 CarPlay 的**第二路显示流**（仪表盘流）；
2. 从车机的 AVB/RTP 流里抓 H.264 视频；
3. 用 Intel Media SDK 解码；
4. 通过 `cluster-render` 画到仪表盘，或缩放到 HUD。

所以它是**视频镜像**，不是原生导航数据。手机发什么，它就显示什么。

### 4.2 07/21 的关键前提

- 主机必须是 MGU（iDrive 7）。
- 必须有 root SSH（见上一份方案，风险相同）。
- 需要 `mgu-store`，再装 `Maps on cluster`。
- CarPlay 二进制必须是可打补丁的版本：
  - 项目“已验证”的 build 是 **CarPlay 22.28.3** 和 **21.27.4**；
  - 22.x 起支持第二路显示流；**20.x 不支持**（issue #8 的 2020 年 G20 就是 CarPlay 0.20.20，直接失败）；
  - 07/21 的车机有可能是 21.27.4，**这是好消息**，但要先用 `probe` 确认，不能猜。
- 车机编码 `KI_KOMBI_VARIANT` 必须是 **7**。不是 7 的话，app 会显示 `Not supported on this car`。
- HUD 是 **experimental**：可以调大小/位置，但可读性、稳定性都不如仪表盘。

### 4.3 步骤

1. 按上一份方案开 root（接受清编码、删 Digital Key、可能变砖的风险）。
2. 安装 `mgu-store`，添加 `robertnisipeanu/idrive-apps` 源。
3. 安装 **Maps on cluster**，按提示重启车机。
4. 车机设置：**仪表盘 > 中央显示区域 > 地图视图**。
5. 打开 app 看 **CarPlay Status**：
   - `Patched`：成功；
   - `Not patched` / `Update needed`：按 Patch；
   - `Not supported on this car`：编码或 CarPlay 版本不支持；
   - `Untested version`：可以试，但风险更高。
6. **Show on** 选 `Cluster`、`HUD (experimental)` 或 `Cluster and HUD (experimental)`。
7. 先用 **Apple 地图/Waze** 验证整条视频链路；再试高德。
   - 如果 Apple 地图能上 HUD、高德不能，说明高德没往 CarPlay 仪表盘流发画面，patch 也救不了。
   - 项目文档只列了 Waze / Apple Maps / Google Maps，**没有明确列高德**；高德 2024 年才适配 CarPlay 仪表盘，理论上是同一路流，但属于未验证组合。

### 4.4 Boot loop 怎么救

这个 patch 最实际的风险是车机无限重启。项目自带 boot recovery：连续失败会还原原版 CarPlay。手动救援：

1. 先让手机连不上 CarPlay：拔线，或关掉 iPhone 的 Wi-Fi 和蓝牙。
2. SSH 上车机。
3. 执行：
   ```sh
   sh /var/opt/waze-on-cluster/current/woc-patch.sh revert
   ```
4. 重启车机。

如果 SSH 也进不去，就只能按音量旋钮/断电等办法，严重时 ISTA 恢复。

### 4.5 我的建议

- **不要为了 HUD 直接上 patch。** 先用原生路线把仪表盘跑通。
- 如果 HUD 是硬需求，先用 **Apple 地图**满足 HUD，仪表盘用高德。这是 07/21 上最稳的组合。
- 只有当你愿意接受 root 的全部风险，并且能接受“高德可能不发仪表盘流”的不确定性，才走 patch。

### 4.6 按小红书第 1 条复刻的实操顺序

如果你就想照着那条笔记做，顺序应该是这样（不要跳步）：

1. **先确认基线**：不 root 的情况下，用 Apple 地图导航，看仪表盘中央能不能切到地图。能，说明车机的 CarPlay 第二路流是通的，后面成功率高；不能，先别急着 root。
2. **只读探测**：用上一份方案里的只读审计确认主机是 MGU、能读到 CarPlay 版本号。目标是确认版本是 21.27.4 或 22.x；如果是 20.x，这条路直接放弃。
3. **开 root**：接受清 HU_MGU 编码、Digital Key 失效、潜在 boot loop 的代价，提前用 E-Sys 备份 NCD。
4. **装 mgu-store**，添加 `robertnisipeanu/idrive-apps` 源。
5. **装 Maps on cluster**，重启车机。
6. **看 Status**：`Patched` 才能继续；`Not supported on this car` 先去查 `KI_KOMBI_VARIANT` 是不是 7。
7. **先苹果地图，后高德**：苹果地图通了，只证明视频链路通；再切高德，看仪表盘有没有画面。**高德没画面就是没画面，patch 调不出来**——因为它是镜像手机发来的流，手机不发就没有。
8. **HUD 最后试**：`Show on` 选 `HUD (experimental)`，能显示也只当彩蛋，别当稳定功能用。
9. **留好退路**：装完先记下 `sh /var/opt/waze-on-cluster/current/woc-patch.sh revert` 这条命令，以及关掉 iPhone Wi-Fi/蓝牙让车机不进 CarPlay 的救援顺序。

顺带说一句：第 2 条笔记里那种“花钱找人装”的服务，装的很可能就是这个 patch。你可以自己装，没必要付费；但如果对方能先帮你确认 CarPlay 版本和 `KI_KOMBI_VARIANT`，这个信息本身是有价值的。

---

## 5. 07/21 专属检查清单

动手前逐项确认：

- [ ] 主机是 MGU（`HU_MGU`），不是 NBT EVO。
- [ ] iPhone 是 12 或更新，iOS ≥ 17.4。
- [ ] 高德是最新版。
- [ ] 仪表盘中央显示区域能切到「地图」。
- [ ] Apple 地图导航时，仪表盘/HUD 有反应（建立基线）。
- [ ] 高德导航时，仪表盘是否显示；显示完整地图还是简化引导。
- [ ] 如果走 patch：CarPlay build 是 21.27.4 / 22.x，`KI_KOMBI_VARIANT` = 7。
- [ ] 如果走 patch：已备份 HU_MGU 编码，能接受 Digital Key 重新配对。

---

## 6. 避坑

- **别为“CarPlay 仪表盘投屏”付费买激活。** 高德这个功能是手机 App 原生能力，免费；收费的通常是“社区 patch 安装服务”，先搞清楚他装的是什么。
- **别把“仪表盘”和“HUD”混为一谈。** 仪表盘是彩色液晶屏，可以显示完整地图；HUD 是投影，原厂只给固定格式的导航信息，全地图是实验性玩法。
- **07/21 的 HUD 不要期待高德。** 2021 年 X3 的实测结论就是第三方地图不行；2025 年 i-Level 后才看到 Google 地图上 HUD，高德还没确认。
- **升级 i-Level 会让 root 和自定义 App 全部失效**，顺序永远是：先升级，再 root，再装东西。
- **patch 是视频镜像，不是数据集成。** 它不会让 HUD 显示高德的转向箭头卡片，只会把手机发来的那路地图画面缩放上去。

---

## 参考

- 高德 CarPlay 仪表盘投屏（支持条件：iPhone 12+、iOS 17.4+、车机支持）：  
  https://news.mydrivers.com/1/997/997302.htm  
  https://xinwen.bjd.com.cn/content/s66bc9005e4b07fbcccaecdb2.html
- 宝马 CarPlay 高德仪表盘实测（方向盘齿轮键 → 显示内容 → 地图）：  
  https://chejiahao.autohome.com.cn/info/16226288
- BMW 2020-07 OTA：Apple 地图导航在 HUD/组合仪表显示转弯和车道信息：  
  https://finance.sina.cn/tech/2020-10-20/detail-iiznezxr6938930.d.html
- 2021 X3 论坛：HUD 只支持车机导航和 iPhone 自带导航，高德/百度不行：  
  https://club.autohome.com.cn/bbs/thread-c-4658-99524505-1.html
- 2025-09：i-Level 07.2025 后 Google 地图转弯提示上 HUD：  
  https://g20.bimmerpost.com/forums/showthread/2205209/google-maps-turn-by-turn-now-on-hud-iphone
- Maps on cluster（CarPlay/Android Auto 地图上仪表盘/HUD，需 root）：  
  https://github.com/robertnisipeanu/idrive-apps
- CarPlay 版本兼容性（21.27.4 / 22.28.3 已验证；20.x 不支持第二路流）：  
  https://github.com/robertnisipeanu/idrive-apps/issues/8
- 你发的三条小红书笔记（第 1 条为 `robertnisipeanu/idrive-apps` + 苹果地图；第 2 条为个人记录；第 3 条为自定义 App）：  
  https://xhslink.cn/o/3ltCyO9kJOV  
  https://xhslink.cn/o/75pW9y4aIfz  
  https://xhslink.cn/o/7eNl30MMmZ4
