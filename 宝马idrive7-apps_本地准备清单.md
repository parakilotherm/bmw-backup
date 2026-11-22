# idrive7-apps-zh 上车前：本地准备清单

> 目标：把 `1031527554/idrive7-apps-zh` 装到 07/21 的 X3（MGU / iDrive 7）上。
> 先说清楚：这个仓库装完是车机里的**中文自定义 App**（车辆数据、计时器、氛围灯、故障扫描等），
> **不会**把 CarPlay/高德投到仪表盘。CarPlay 投屏是另一个项目（Maps on cluster）。
>
> 本地结论：**这台 Mac 的软件环境已经够用，不需要再装 Python/Node。真正缺的是三样硬件/环境：ENET 线、稳定供电、Windows + E-Sys 备份环境。**
>
> **2026-11-22 进展**：root SSH 已开通并持久化，刷写完成；HU_MGU 编码被清空（CAFD `FFFFFFFF`），Store 安装因此暂停。下一步见 `恢复HU_MGU编码_再装Store.md`。

---

## 1. 已经准备好的（2026-11-17 已在本机完成）

| 项目 | 状态 |
|---|---|
| macOS 26.3.2（arm64），剩余磁盘 791 GB | ✅ |
| Python 3.12.12 + pip 26.1.1 | ✅ 满足仓库要求（Python 3.6+） |
| `cryptography` 46.0.7 | ✅ 已安装，满足 root 工具要求 |
| Node.js 24.14.1 / npm 11.11.0 | ✅ 满足要求（Node 14+） |
| OpenSSH 10.2p1、git 2.50.1、curl、jq | ✅ |
| `bmw/idrive7-apps-zh` | ✅ 已克隆（94 MB，15 个应用，含中文语料和预编译渲染器） |
| `bmw/idrive7-8-ssh-root` | ✅ 已克隆（1.7 MB，已装依赖） |
| 工具自检 | ✅ `ssh_enable.py --help` 正常；`kit/tools`、`deploy` 全部通过语法编译 |
| npm 依赖 | 不需要，仓库只用 Node/Python 标准库 |

目录：

```
/Users/mengda/Documents/Qoder/2026-11-16/60313d93/bmw/
├── idrive7-apps-zh/      # 要装的应用 + 构建/部署工具
└── idrive7-8-ssh-root/   # 开 root SSH 的工具（含只读编码备份 preflight）
```

---

## 2. 还需要准备的（按重要性排序）

### 2.1 必买：ENET 线（OBD → RJ45）

- 宝马 F/G 系列 ENET 诊断线，**买质量好的**；劣质线在刷写中途掉链路会很麻烦。
- Mac 没有原生网口。你机器上已有 `en3/en5/en6` 三个 “Ethernet Adapter”，大概率来自扩展坞或 USB 网卡。
  插上 ENET 后确认系统设置 → 网络里**多出一个“已连接”的有线接口**；没有就说明转接器/线有问题。
- ENET 走 `169.254.x.x` 链路本地地址，macOS 会自动分配，**不需要手动设 IP**（这点比 Windows 省事）。
- 第一次运行扫描时，macOS 可能弹“本地网络”权限，给 Terminal（或 Qoder）允许即可。

### 2.2 强烈建议：稳定供电

- 刷写期间车辆必须保持唤醒，电压不能掉。准备**电瓶充电器/稳压电源**（如 CTEK 一类），夹在机舱搭铁柱上。
- 别在电量低、自动启停频繁、或车辆会休眠的状态下刷。

### 2.3 必须解决：Windows + E-Sys 备份环境（本机目前没有）

刷写会清空 `HU_MGU` 编码/CAFD，root 工具自带的 `preflight` 只能做**只读备份**，恢复仍要编码工具：

- **首选**：借一台 Windows 10/11 笔记本，装 **E-Sys + 匹配 07/21 的 PSdZData**，提前学会 `HU_MGU` 按 VO（Code by FA）重新编码。
- 只有 Mac 的话：装 **UTM（免费）+ Windows 11 ARM**，E-Sys 走 x64 模拟。能跑，但驱动/运行库折腾成本高，**不建议第一次刷机就用这个**。
- BimmerCode 可以备份/恢复，但社区实测**可能只写回一部分编码且不报错**（症状：语言退回英文、图标缺失、倒车影像不显示）。可以当第二份备份，别当唯一方案。
- 数字钥匙（Digital Key）**不在编码备份里**，刷完必须重新配对所有钥匙。

### 2.4 可选：BimmerCode + 蓝牙/WiFi OBD 适配器

- 用于刷前快速看一眼车辆状态、刷后对照设置。不是必需，也不能替代 E-Sys。

---

## 3. 两条安装路线，本地要做的事不一样

### 路线 A：Store 路线（这个 fork 作者推荐，真车验证过）

车机端：

1. 先有 root SSH。
2. 在电脑上跑一次 mgu-store 安装脚本（本地不需要预装任何东西，用系统 ssh）：
   ```sh
   curl -fsSL https://raw.githubusercontent.com/robertnisipeanu/mgu-store/main/deploy/install.sh \
     | sh -s -- first-install --identity <root私钥路径>
   ```
3. 车机 Store → 设置 → **Add source** → 输入 `1031527554/idrive7-apps-zh` → 选分支 `main`。
4. 在 Store 里逐个安装；车机会**现构建**（mgu-store 会给车机装 Python）。
5. `doom` 构建要联网，没 uplink 就跳过。

这条路线**不在 Mac 上构建**，所以本地不需要 `oap-sdk`，也不需要跑 `build.py`。

### 路线 B：直推路线（README 3.x，本地构建后 SSH 推上去）

顺序不能反：`oap-sdk` 必须从**你自己的车机**提取，所以要先有 root。

1. 开 root 后，从车机取一个原厂应用并提取 SDK：
   ```sh
   cd bmw/idrive7-apps-zh
   scp -i <key> root@<MGU-IP>:'/usr/lib/oap-mainapp/preinstalled_apps/x_call@*.tgz' .
   mkdir -p x_call && tar xzf x_call@*.tgz -C x_call
   node kit/tools/extract_sdk.js x_call/javascript/app.js
   ```
2. 构建中文套件：
   ```sh
   python3 kit/tools/make_kit.py --language zh-CN
   ```
   产物：`dist/mgu-custom-apps-zh-CN/` 和 `.tar.gz`。
3. 只读检查 → 安装：
   ```sh
   cd bmw/idrive7-apps-zh
   KIT=dist/mgu-custom-apps-zh-CN HU=root@<MGU-IP> KEY=<key> sh deploy/push-to-car.sh check
   KIT=dist/mgu-custom-apps-zh-CN HU=root@<MGU-IP> KEY=<key> sh deploy/push-to-car.sh install
   ```
   撤销：
   ```sh
   KIT=dist/mgu-custom-apps-zh-CN HU=root@<MGU-IP> KEY=<key> sh deploy/push-to-car.sh restore
   ```

> 注意：克隆下来的 `deploy/*.sh` 没有可执行位，用 `sh deploy/...` 调用，或先 `chmod +x`。
> 私钥必须 `chmod 600`，否则脚本会拒绝。

**建议**：第一次走路线 A。它绕开本地构建和 SDK 提取，出错面小；等车机能稳定装一个应用后，再考虑直推。

---

## 4. 版本选择：别一上来就装最新

仓库今天（11-17）还在提交，但真车验证过的是更早的状态：

| 提交 | 日期 | 状态 |
|---|---|---|
| `550f311` | 2026-11-17 | 最新，**未上车验证** |
| `4e3e3f3` | 2026-11-16 | 同步上游（ambient 2.5.0 爆闪、依赖修复），**未验证** |
| `e7205d0` | 2026-11-11 | 15 个应用升级，**未验证** |
| `7fe88ef` | 2026-11-09 | **真车验证过：14 个应用经 Store 安装成功，中文界面** |

想稳：`git checkout 7fe88ef`（或先在 Store 里只装 1–2 个应用试水，别一次全装）。
想要 redline 的 HUD、氛围灯爆闪：用最新，但先装单个应用验证，再批量。

---

## 5. 上车当天的执行顺序（不跳步）

- [ ] 车辆停稳、点火、接充电器。
- [ ] 插 ENET，确认 macOS 多出一个有线接口且拿到 `169.254.x.x`。
- [ ] **只读体检**：
  ```sh
  cd bmw/idrive7-8-ssh-root
  python3 ssh_enable.py status
  ```
  读取软件版本、MGU IP、调试状态，不写任何东西。
- [ ] **只读编码备份**（重要，先做）：
  ```sh
  python3 ssh_enable.py preflight --vin <17位VIN>
  ```
  产物在 `data/preflight/…`，含 `backup.json`、`coding_MGU.txt`、`RESTORE.txt`、`SHA256SUMS`。
- [ ] 用 E-Sys 另做一份 NCD 备份（Windows）。
- [ ] 刷写并启用（**只跑一次**，成功后不要重复刷）：
  ```sh
  python3 ssh_enable.py flash --vin <VIN> --arch id7 --persist
  ```
- [ ] 确认 SSH：
  ```sh
  ssh -i data/enrollments/<VIN>/id_rsa root@<MGU-IP>
  ```
- [ ] 装 mgu-store，或走直推路线。
- [ ] 先装一个应用（建议 `car_vitals` 或 `service`），重启车机确认能打开、中文正常。
- [ ] 再装其余应用；`doom` 最后装。
- [ ] 记下撤销入口：Store 里 Remove，或 `push-to-car.sh restore`。

---

## 6. 出问题时的优先级

1. **SSH 连不上**：先查 ENET 链路和 `169.254.x.x`，再跑 `status`；不要重新 flash。已刷过的机器用 `enable`（只修 SSH，不刷写）。
2. **车机卡在蓝色编程界面**：`python3 ssh_enable.py exit-prog`。
3. **编码被清（语言变英文、图标缺失、倒车影像没了）**：E-Sys 按 VO 重编 `HU_MGU`；不要用重新刷写解决。
4. **应用导致 HMI 异常**：Store Remove，或 `push-to-car.sh restore`。
5. **Digital Key 失效**：刷写后的预期行为，重新配对。

---

## 7. 一句话总结

本地软件**已经齐了**，仓库和依赖都装好了。你现在只差：**ENET 线、充电器、一台能跑 E-Sys 的 Windows（或愿意折腾 UTM）**。把这三样凑齐，再按第 5 节顺序走；第一次建议只装 1–2 个应用，别 15 个一起上。
