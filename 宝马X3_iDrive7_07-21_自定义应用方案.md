# 宝马 X3（G01）iDrive 7 / MGU 07/21 自定义应用方案

> 基于 `1031527554/idrive7-apps-zh`（中文 fork）与上游 `cauliflowermossgrape/idrive7-apps` 的公开实现整理。
> 结论先说：**技术上可行，但 07/21 是该项目未验证的老版本；真正危险的是第一步开 root，不是装 App。**
> 建议先做只读体检，再决定是否动手。

---

## 0. 先把概念对齐：这不是“刷第三方固件”

这套方案实际做三件事：

1. **用 ENET 线刷 BMW 官方 debug 镜像，开 root SSH**（唯一一次“刷”的动作，刷的是官方调试镜像，不是自制固件）。
2. **安装 OAP 自定义应用**（BMW 原生 JS 小程序）和 GPU 渲染器 `mgu-renderd`。
3. **用 bind-mount 的 `main.js` 补丁**，让 MainApp 显示这些没有 BMW 签名的应用。

它**不会**：

- 把车机变成安卓大屏；
- 安装 YouTube / 爱奇艺 / 高德车机版等安卓 App；
- 解除行车视频限制；
- 改地图、改 VO、刷隐藏功能。

如果你说的“刷机”是**刷隐藏**（行车视频、启停记忆、Sport Plus 等），别走这套，BimmerCode/E-Sys 更简单、风险低得多。这套方案的目标是**在 iDrive 里多出一批车辆数据/性能/诊断类 App**。

---

## 1. 针对 07/21 的可行性判断

| 检查项 | 判断 |
|---|---|
| 主机必须是 **MGU**（iDrive 7），不是 NBT EVO | X3 G01 早期有 iDrive 6/NBT EVO。先确认 ECU 列表里有 `HU_MGU`，工具只认 MGU。 |
| 开 root 的工具是否支持 07/21 | 支持。工具按主机类型 `BTLD 00003F36` 判断，不按 I-Step；iDrive 7 全版本可用。 |
| App 项目是否验证过 07/21 | 没有。作者只在 `P18_23w44.2`（2023 年第 44 周）上开发验证，README 说“其他 MGU 版本很可能可用”。 |
| 有没有更老版本的成功案例 | 有。社区在 `G_20w28.3` / MainApp `7.1.2`（2020 年第 28 周）的 G20 上成功装过 `mgu-store` 和 `idrive-apps`，只有依赖新版 CarPlay 二进制的 `waze-on-cluster` 失败。**07/21 比它新，概率不低。** |
| 07/21 的 `oap-sdk` 是否够新 | 可能偏老。项目完整 API 对应 SDK 24.10.x；老 SDK 缺的模块如果 App 不需要就没事，需要的话构建/安装会提示，安装器会跳过该 App。 |
| 安装器会不会弄坏东西 | `deploy-car.sh check` 是只读的，会按**你这台车的 `main.js`**现场试补丁；锚点对不上会在改任何文件之前停止。`install` 会先备份再改。 |

**结论：**

- **不升级 i-Level 也可以先试**，因为第一步 `check` 是只读的，能提前告诉你补丁是否匹配。
- 但 07/21 的 MainApp、SDK、HMI 都比作者验证版本老，**升级到 2023 年以后的 I-Step 会明显提高成功率**。
- 如果升级，顺序必须是：**先升级 → 再开 root → 再装 App**。反过来，升级会把 root 和 App 全部清掉。

---

## 2. 风险清单（按严重度）

| 风险 | 概率 | 后果 | 缓解 |
|---|---|---|---|
| 主机变砖 | 低 | 车机黑屏，可能要 ISTA 恢复/拖车 | 稳定电源、原厂 ENET 线、全程别断电、严格按工具流程；不在低电量/雷雨天做 |
| HU_MGU 编码/CAFD 被清空 | **高（工具明确说明）** | 语言变英文、倒车影像/驾驶模式/图标异常、部分功能丢失 | **先用 E-Sys 备份 HU_MGU 的 NCD**；BimmerCode 备份不够，恢复可能只恢复一部分且不报错 |
| Digital Key 配对被删除 | **高** | 手机数字钥匙失效 | 提前知道怎么重新配对；带实体钥匙 |
| 保修 / 经销商检测 | 中 | 主机相关保修可能产生争议 | 能接受再动手；无法保证完全无痕 |
| 车机 OTA / 经销商编程后失效 | 高 | root 和 App 都没了，编码又清一次 | 每次编程后重做；**先升级再 root** |
| 脚本供应链风险 | 低-中 | 以 root 执行第三方脚本 | 固定 commit、和上游 diff、自己读一遍 `deploy/` 脚本；不要用来路不明的打包版 |
| 行车安全 / 违法 | 中 | 分心驾驶；爆闪/警灯在公共道路违法 | 只在停车/封闭场地用；Strobe/Police 不上公共道路 |
| 故障码清除 | 中 | 掩盖真实故障，影响保修/诊断 | 先读码、记录，再决定是否清；不要随手 clear |
| 写类功能（排气阀、xDrive off、BM3/MHD） | 中 | 影响排放/操控/保修 | 第一轮只装只读 App；写类功能保持 Disarmed/Off |

---

## 3. 两条路线，先选一条

### 路线 A：不升级 i-Level，直接试

适合：不想去 4S、能接受“check 可能拒绝”、车本来就在保外。

流程：只读体检 → 开 root → `deploy-car.sh check` → 通过就装，不通过就停。

优点：省钱、省时间。
缺点：07/21 的 MainApp/SDK 较老，可能卡在 check 或部分 App 装不上。

### 路线 B：先去 4S/ISTA 升级到 2023+，再开 root（推荐）

适合：打算长期用、希望一次成功、车机本来就该升级。

流程：恢复原厂编码 → 4S 编程升级 → 确认版本/地图/Digital Key → 开 root → 装 App。

优点：与作者验证版本更接近；顺带拿到新地图、Android Auto、稳定性修复。
缺点：花钱/花时间；升级前如果有 BimmerCode 编码，要先恢复原厂，否则可能编程失败。

> **顺序不能反。** 先 root 再升级 = root 丢失 + App 隐藏 + 编码再清一次。

---

## 4. 物料与准备

### 车上

- BMW **ENET 线**（OBD 以太网诊断线），G 系列用。
- 稳定电源：编程电源或大电流充电器；电量充足。
- 全程：车辆停稳、点火 ON、车门关闭、别让车睡；笔记本别休眠。

### 电脑

- **Windows 笔记本最稳**：开 root 的工具是 Windows 优先，需要管理员权限。
- macOS 也能跑：Python 3.9+、OpenSSH、`sudo`；USB-C 转以太网网卡。工具的网络层已处理 macOS/Linux 的 link-local 发现，但没有 Windows 那么多人验证。求稳可以用 Parallels/VMware + USB 网卡直通。
- 构建 App：Python 3.6+、Node.js 14+。
- 备份编码：**E-Sys + PSdZData + ENET**；BimmerCode 只作辅助。

### 软件/资料

- `idrive7-8-ssh-root`：开 root。
- `idrive7-apps-zh`：中文 App（本方案）或上游英文版。
- 16GB+ U 盘：备用离线部署（车机没有网络时）。
- 记下 VIN、当前 I-Step、车机 IP（工具会读）。

---

## 5. 阶段 1：只读体检（不写任何东西）

### 5.1 确认主机是 MGU

任选其一：

- E-Sys 读 ECU 列表，确认有 `HU_MGU`；
- BimmerCode 连接后看控制单元列表；
- 用 VIN 解码确认生产批次和主机类型。

如果显示 `HU_NBT2` / EVO，**停止**，这套方案不适用。

### 5.2 备份 HU_MGU 编码

- 用 E-Sys 读取 `HU_MGU` 的 CAFD，保存 `CAFD_..._...._....ncd`；
- 记录 SVT/FA（至少记下，方便恢复）；
- 用 BimmerCode 再做一份备份作为辅助，但**不要只依赖 BimmerCode**。

### 5.3 用 root 工具做只读状态检查

```sh
git clone https://github.com/cauliflowermossgrape/idrive7-8-ssh-root.git
cd idrive7-8-ssh-root
python3 -m pip install -r requirements.txt

# 只读：读软件版本、MGU IP、B7F8E5 故障码、22 端口，不刷写
python3 ssh_enable.py status

# 如果自动发现失败，再试 sudo（macOS 的 link-local/网卡权限）
sudo python3 ssh_enable.py status
```

macOS 上网卡建议设为 DHCP（没有服务器时会自分配 `169.254.x.x`），或手动给一个 `169.254.1.2/16`。插 ENET、点火 ON 再跑。

**通过标志：** 能读到 VIN、MGU IP、软件版本，22 端口状态符合预期。只有网关发现成功不算成功。

---

## 6. 阶段 2：开 root（唯一一次刷写）

### 6.1 操作

```sh
# 确认 VIN 与车上一致，架构选 id7
python3 ssh_enable.py flash --vin <你的17位VIN> --arch id7 --persist
```

或 GUI：

```sh
python3 gui.py
# 读取 VIN → 架构 id7 → 勾选“启用后关闭调试模式” → Flash + Enable
```

### 6.2 成功标志

```sh
ssh -i data/enrollments/<VIN>/id_rsa root@<MGU-IP>
# 看到 uid=0(root) 即成功
```

工具会把连接信息写到 `data/enrollments/<VIN>/connect.txt`，私钥在同目录。**整个目录备份好，不要上传公开仓库。**

### 6.3 必须知道的副作用

- **会清空主机编码/CAFD**：语言可能变英文，倒车影像、驾驶模式、HMI 图标等可能异常。
- **会删除 BMW Digital Key 配对**：手机钥匙需要重新配对。
- 刷写后如果功能异常，用之前备份的 NCD 通过 E-Sys 回写 `HU_MGU`；如果 E-Sys 显示 CAFD 为 `FFFFFFFF`，先注入备份的 CAFD，再写 NCD。
- **只跑一次 flash。** SSH 连不上用 `enable` 修复，不要重复 `flash`；重复刷会再次清编码、删数字钥匙，没有好处。
- 如果屏幕黑屏约 1 分钟：这是主机重启，等它自己回来，不要断电。

### 6.4 失败时

- 不要连续点“重试”、不要拔线、不要关电源。
- 按音量/电源旋钮 20–30 秒强制重启主机。
- 仍不行：用 ISTA/E-Sys 恢复；必要时联系能处理 MGU 的店。
- 记住：这一步是整套方案里唯一可能“变砖”的环节，后面的装 App 都有备份和 restore。

---

## 7. 阶段 3：主机侧兼容性审计（07/21 的关键）

SSH 上去后，先**只读**检查，不要急着装。

```sh
uname -a
cat /etc/version 2>/dev/null
cat /etc/os-release 2>/dev/null

# OAP 平台是否存在
ls -l /usr/lib/oap-mainapp/src/main.js
ls -l /usr/lib/oap-mainapp/preinstalled_apps/ | head -20
ls -l /var/opt/public/oap/
df -h /var/opt

# node 版本（补丁/安装脚本要用）
node -v

# MainApp 补丁的两个锚点：必须各出现 1 次
grep -c 'class ProvisioningNotifier extends events_1.EventEmitter{' /usr/lib/oap-mainapp/src/main.js
grep -c 'isMainapp&&section===mainAppSection&&(newProvisioning.installable_apps=this.modifyInstallableApps(newProvisioning.installable_apps))' /usr/lib/oap-mainapp/src/main.js

# SOME/IP 目录、HMI 二进制、平台状态
ls -l /etc/oap/someip/someip-interfaces.json
ls -l /usr/bin/HmiMain3D /usr/bin/Hmi8 2>/dev/null
systemctl status oapcontroller --no-pager
```

**判断标准：**

- `main.js` 存在，且两个 `grep -c` 都返回 **1** → 补丁锚点匹配，可以继续。
- 任一返回 0 或大于 1 → 07/21 的 MainApp 版本不同，**不要手工改锚点**；先升级 i-Level，或放弃。
- `/var/opt/public/oap/oap_12201` 不存在，或 uid 范围不是 `12203..12450` → 平台形态不同，停止。
- 剩余空间建议 **≥ 300 MB**（App + 渲染器 + 备份）。
- 如果 `HmiMain3D` 不存在但有 `Hmi8`，说明可能被识别成 iDrive 8 路径，和你的车不符，先确认。

### 7.1 提取 SDK

```sh
# 在电脑上，回到 apps 仓库根目录
scp root@<MGU-IP>:'/usr/lib/oap-mainapp/preinstalled_apps/x_call@*.tgz' .

mkdir -p x_call
tar xzf x_call@*.tgz -C x_call
node kit/tools/extract_sdk.js x_call/javascript/app.js
ls oap-sdk/
```

- 如果没有 `x_call`，从 `preinstalled_apps/` 里挑另一个 BMW 自带 App 的 `.tgz`，用它的 `javascript/app.js` 提取。
- 提取后看 `oap-sdk/source/` 是否有你需要的模块；构建时缺模块会报出来。
- 如果老 SDK 缺得太多，优先升级 i-Level，而不是去网上找别人的 `oap-sdk`（版权 + 来源风险）。

---

## 8. 阶段 4：PC 构建 + 部署

### 8.1 选代码版本

```sh
git clone https://github.com/1031527554/idrive7-apps-zh.git
cd idrive7-apps-zh
git log --oneline -10
```

中文 fork 的注意事项：

- 该 fork 基于上游 **2026-11-07** 的快照；
- **2026-11-09 在真车上验证过 14 个 App 中文安装**；
- 之后 11-11、11-12、11-16 的同步批次**尚未上车验证**（尤其 11-16 加了 strobe/远光爆闪等新功能）。

**建议：** 首次安装固定到 2026-11-09 附近的 commit，成功后再考虑升级到最新。求稳也可以用上游英文最新版，功能更全、验证更多。

### 8.2 构建中文包

```sh
# 全部 App（Doom 首次构建需要联网下载 Freedoom WAD）
python3 kit/tools/build.py --language zh-CN

# 或只构建你需要的 App（推荐第一轮）
python3 kit/tools/make_kit.py service car_vitals live_data dtc_scanner ambient_config --language zh-CN
```

产物在 `dist/`。`--language zh-CN` 会生成 `dist/mgu-custom-apps-zh-CN/`。

> 如果只想先试水，第一轮不要装 Doom、不要装 Engine Config 的写类功能。只读 App 足够验证整条链路。

### 8.3 部署

```sh
# KEY 用 root 工具生成的私钥，先 chmod 600
chmod 600 ~/.../data/enrollments/<VIN>/id_rsa

KIT=dist/mgu-custom-apps-zh-CN \
HU=root@<MGU-IP> \
KEY=~/.../data/enrollments/<VIN>/id_rsa \
deploy/push-to-car.sh check
```

**`check` 重点看：**

- 是否识别到主机；
- 是否成功试构建 MainApp 补丁；
- 列出将要改动的文件；
- 是否有 App 因为接口/版本被拒绝。

通过后再：

```sh
KIT=dist/mgu-custom-apps-zh-CN HU=root@<MGU-IP> KEY=... deploy/push-to-car.sh install
```

### 8.4 install 会做什么

1. 检查主机 + 按当前固件试构建补丁；不对就停。
2. 备份所有要改的文件到两处，并校验。
3. 启用 provisioning hook（bind-mount `main.js`，重启后仍生效）。
4. 安装每个 `.mgu` 到 `/var/opt/public/oap/oap_<uid>/`，用 BMW 自己的生成器生成 SOME/IP 和 ACL。
5. 安装 GPU 渲染器 `mgu-renderd`。
6. 设置 controller gate（Doom 用）；设置成功后**主机可能重启一次**，黑屏约 1 分钟。
7. 重启 `oapcontroller`，HMI 在 1–2 分钟后刷新 Apps 列表。

如果只想先不重启：`NO_REBOOT=1`；不想装 controller gate：`NO_INPUTGATE=1`。

---

## 9. 阶段 5：验收

```sh
# 车机上
cd /var/opt/mgu-custom-apps
sh deploy-car.sh status
sh deploy-car.sh backups
```

检查：

- Apps 列表里出现新图标，能打开；
- 每个 App 的日志在 `/var/opt/public/oap/oap_<uid>/data/<id>.log`；
- 没有新增异常故障码（尤其 `B7F8E5` 调试激活）；
- HMI 图标、倒车影像、驾驶模式、语言正常；
- 车辆能正常休眠（锁车后等几分钟，看是否反复唤醒）。

### X3 上建议先装/先试的 App

| 类型 | App | 说明 |
|---|---|---|
| 只读、低风险 | Service | 保养项目/里程/日期 |
| 只读 | Live Data | 车辆信号、工况、油耗/行程 |
| 只读 | Car Vitals（M Vitals） | 转速、增压、油温、胎压等；X3 30i/30d/M40i 都适用 |
| 只读 | Warm-Up | 油温/水温暖机与涡轮冷却倒计时 |
| 只读 | Fault Scanner | 读故障码；**不要随手 clear** |
| 只读、实验 | Transmission Config | ZF 8HP 数据；先只看 |
| PHEV 专属 | eDrive | 只有 X3 30e 插混适用；30i/30d 不适用 |
| 性能、封闭场地 | Redline / Dyno / Brake Test / Drag Timer / G-Force | 公共道路别用 |
| 氛围 | Ambient config | 只改颜色/动画可以；Strobe/Police 只在封闭场地 |
| 暂缓 | Engine Config（排气阀/xDrive off/BM3/MHD） | 写类功能，先保持 Disarmed/Off |
| 暂缓 | Doom | 需要 controller gate 和重启，第一轮可不装 |

---

## 10. 回滚与救砖

### 正常回滚

```sh
KIT=dist/mgu-custom-apps-zh-CN HU=root@<MGU-IP> KEY=... deploy/push-to-car.sh restore
```

或车机上：

```sh
cd /var/opt/mgu-custom-apps
sh deploy-car.sh restore   # 会要求输入 YES
```

`restore` 会：

- 停平台；
- 删除自定义 App、hook、drop-in；
- 按**第一次 install 前的 stock 备份**逐文件恢复并校验；
- 重启 HMI，让图标消失。

### 编码被清空

- 用 E-Sys 把之前备份的 NCD 写回 `HU_MGU`；
- 如果 CAFD 显示 `FFFFFFFF`，先注入备份的 CAFD，再写 NCD；
- 不要用“重新刷写”来解决编码缺失——编码本来就是被刷写清空的。

### 主机黑屏/变砖

- 先等 2–3 分钟，可能是重启；
- 按音量/电源旋钮 20–30 秒；
- 仍不行：ISTA/E-Sys 恢复；必要时拖车/经销商。
- **不要在慌乱中反复刷写**，那会把可恢复的问题变成不可恢复。

---

## 11. 之后的维护

- **车机 OTA / 经销商编程后**：root 和 App 大概率失效，编码也可能再清一次。需要重新 root + 重新 install。
- **不要重复 flash**：只有主机软件变了才需要重新刷 debug 镜像；SSH 连不上用 `enable`。
- **每次 install 前跑 `check`**：尤其是车机升级后，MainApp 变了，补丁可能不再匹配。
- **保留三样东西**：`data/enrollments/<VIN>/` 私钥、`mgu_custom_backup/` 车机备份、E-Sys 的 NCD。
- **中文 fork 会分叉**：上游更新后需要重新同步；固定 commit 比追最新更稳。
- **卖车/回 4S 前**：先 `restore`，再回写编码；不要带着 root 和自定义 App 直接交车。

---

## 12. 我的建议（针对你这台 07/21 X3）

1. **先确认你是不是真的需要。** 如果你只是想要 CarPlay、中文、地图、隐藏功能，这套方案性价比很低，风险和折腾程度都不小。
2. **如果你要做，走路线 B 更稳**：先升级 i-Level，再 root。07/21 太老，升级后与作者验证版本更接近，也能少踩 SDK/HMI 的坑。
3. **第一轮只做“只读链路”**：root → `check` → 只装 Service / Live Data / Car Vitals / Warm-Up / Fault Scanner。验证稳定后，再考虑性能类和氛围类。
4. **不要第一轮装**：Doom、Engine Config 的写功能、xDrive off、排气阀、Strobe/Police。
5. **把“编码备份 + E-Sys 回写能力”当成硬门槛。** 没有这个能力，不建议开 root；因为工具明确会清编码、删 Digital Key。
6. **接受两个现实**：主机保修可能受影响；每次 dealer 编程后要重做。不能接受就别动手。

### 时间/成本粗算

| 项目 | 时间 | 成本 |
|---|---|---|
| 只读体检 + 编码备份 | 1–2 小时 | ENET 线 ¥50–150；E-Sys 环境另算 |
| 开 root | 20–40 分钟 | 免费（工具开源） |
| 构建 + check + install | 30–60 分钟 | 免费 |
| 4S 升级 i-Level（可选） | 半天–1 天 | ¥0–2000，看店/保修 |
| 出问题恢复编码/救砖 | 1 小时–拖车 | 视情况 |

> 提醒：这套 App 本身是免费开源的。有人把 App 打包收费，或声称“远程激活”收费，先核实；为**安装劳务**付费是一回事，为**开源 App 本身**付费是另一回事。

---

## 参考

- 中文 fork（本项目）：https://github.com/1031527554/idrive7-apps-zh
- 上游项目：https://github.com/cauliflowermossgrape/idrive7-apps
- 开 root 工具：https://github.com/cauliflowermossgrape/idrive7-8-ssh-root
- 老固件成功案例（G_20w28.3 / MainApp 7.1.2）：https://github.com/robertnisipeanu/idrive-apps/issues/8
- Bimmerpost 讨论：https://f90.bimmerpost.com/forums/showthread/2261064/turn-your-idrive-7-into-a-race-dash-open-source-custom-apps-for-the-mgu
