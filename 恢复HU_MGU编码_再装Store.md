# 下一步：先恢复 HU_MGU 编码，再继续装 Store / App

> 车辆：X3 G01，MGU（iDrive 7），原软件 07/21（`J18_21w31.5`，I-Step 21-07）
> VIN：`LBV31DU03NS273096`
> 更新：2026-11-22 18:34

---

## 1. 现在的实际状态（已现场核实）

| 项目 | 结果 |
|---|---|
| root SSH（22 端口） | 正常，`root@169.254.227.133` 可登录 |
| 调试状态 / DTC `B7F8E5` | 已关闭 / `absent` |
| 主机应用平台 | `oapcontroller` 处于 `active` |
| **HU_MGU 编码（CAFD）** | **`FFFFFFFF_255.255.255` —— 编码已丢失，处于默认值** |
| mgu-store 安装 | **被拒绝**，原话：`The head unit's coding is missing: it holds default values...` |

刷写后 CAFD 被清空是**已知且必然**的结果，不是这次安装出错。Store 的安装器检测到“未设码”就主动停下——这是保护，不是 bug。

另外留意：`status` 里 `SWFL 000056EA` 已从 `018.027.001` 变成 `030.016.002`，这是调试包写入带来的差异，属正常现象。

### 编码丢失会导致什么

主机所有功能都回落到“默认值”，常见表现（不一定全出现）：

- 界面语言退回英文，部分 HMI 图标缺失或错乱；
- **原厂应用列表为空**——自定义 App 就是挂在这个列表里的；
- 倒车影像不显示、驾驶模式切换弹窗消失；
- 个人设置（座椅/空调/驾驶模式记忆等）复位；
- BMW 数字钥匙配对丢失（需要重新配对）。

所以顺序只能是：**先恢复编码 → 再装 Store → 再装 App**。在编码恢复前不要装 App、不要改 `mainapp`。

---

## 2. 恢复编码：两条路

### 路线 1：E-Sys 按车辆 VO 重新设码（自己动手，需要 Windows）

**为什么是“按 VO 设码”而不是“写回备份”**：`preflight` 备份的 `coding_MGU.txt` 是字节级记录，适合做验收对照；但直接整块写回在某些车上会出现“部分设置仍复位”。按车辆 VO 重新生成编码最可靠（root 工具 README 的 issue #10 就是这个结论）。

**需要准备：**

| 项目 | 说明 |
|---|---|
| Windows 10/11 x86 笔记本 | E-Sys 只有 Windows 版。Apple Silicon 上用 Windows 11 ARM 虚拟机 + x86 模拟有成功案例，但不保证，别把唯一希望押在它上面 |
| E-Sys 3.30 / 3.40 + Launcher（token） | 社区常见组合；版本不必最新，能连车即可 |
| **psdzdata Full**（必须 Full，Lite 没有 CAFD） | 版本要覆盖 21-07，建议用 2022 年以后的 Full 数据（约 40 GB） |
| ENET 线 + 稳定供电 | 已有；设码期间车辆保持唤醒、接充电器 |
| 备份文件 | `bmw/idrive7-8-ssh-root/data/preflight/20261122-102752-LBV31DU03NS273096/`（另有 `bmw/backups/mgu-coding-LBV31DU03NS273096-20261122.tar.gz`） |

**操作顺序（G01 的 E-Sys target 是 `S15A`，以软件里的车型列表为准）：**

1. ENET 连接，ZGW 搜索，确认网关 `169.254.5.114`、MGU `169.254.227.133`。
2. E-Sys 连接 target `S15A`。
3. `Read FA` → `Activate FA` → `Read SVT (VCM)`。
4. 左侧选中 **HU_MGU** → `Detect CAF for SWE` → 选与当前 I-Step（21-07）匹配的 CAFD。原车是 `CAFD 00003E52_008.007.023`，正常应能选到同一版本或该 I-Step 对应版本 → **注入 CAFD**。
5. 对 HU_MGU 执行 **Code**（VO 设码 / code by FA）。不需要整车编程，不需要动其他 ECU。
6. 等主机重启。用只读命令确认 CAFD 不再是 `FFFFFFFF`：
   ```sh
   cd bmw/idrive7-8-ssh-root
   python3 ssh_enable.py status --gateway 169.254.5.114
   ```
7. **验收（最关键）**：再跑一次只读 preflight，生成一份新备份，和旧备份逐块比对：
   ```sh
   python3 ssh_enable.py preflight --gateway 169.254.5.114
   ```
   新目录里的 `coding_MGU.txt` 与旧文件对比（忽略第一行表头）。23 个块全部一致 = 完整恢复；个别块不同通常只是个人设置差异，不是失败。

### 路线 2：去店里用 ISTA 设码（最省事，推荐给不想折腾 Windows 的人）

跟宝马专修店 / 4S 说清楚这句话就行：

> “主机刷过机，HU_MGU 设码丢了，CAFD 显示 FFFFFFFF，需要按车辆 VO 重新设码 HU_MGU。只做设码，不要做整车编程升级。”

- **只要设码**：不动其他 ECU，root 和后面要装的 App 都不受影响。
- **不要接受“整车编程 / 软件升级”**：那会把 root 和自定义 App 全部清掉，还得从头再来。
- 做完同样用 `status` + 二次 `preflight` 验收。

### 为什么 BimmerCode 不行

- BimmerCode 只能改**已有 CAFD** 的编码，**不能注入缺失的 CAFD**；本车现在 CAFD 是 `FFFFFFFF`，它没有可用的编码模板。
- root 工具文档也写明：BimmerCode 的恢复可能只写回一部分且**不报错**，症状是“恢复后个别功能仍然没有回来”。

结论：这台车要恢复编码，只有 **E-Sys** 或 **ISTA（店里）** 两条实际路径。

---

## 3. 编码恢复之后：继续路线 A

**当前状态（2026-11-22）**：已用仓库直装路线装好 `car_vitals 2.27.3`（uid 12450）与 MainApp 服务配置挂钩（`20261122-105051` 备份），但主机重启后**仍看不到磁贴**。原因已查明：

- `oapcontroller` 与 `hmi` 日志都只有 `Coding base: Default values flag is set, default values will be used`；
- 控制器内含 `checkCodings` / `CodingBase::isDefaultCoding`，编码为默认值时**不会拉起 MainApp**；
- 没有 MainApp，HMI 无法生成应用列表，磁贴就不会出现。**多次重启无效**（安装后已重启两次验证）。

所以直装并没有失败，文件和挂钩都已就位，只差编码这一环；编码恢复后 MainApp 会启动，`custom_apps.json` 里的条目即生效，磁贴出现（HMI 刷新约 1–2 分钟）。应用在 `/var/opt/public`，恢复编码不会清掉它。

**开源写回编码尝试（已失败，勿重复）**：2026-11-22 晚用 `deathwalker80/mgu-plus` 的 `coding-restore`，以刷前 preflight 备份做 dry-run 通过；真实执行在 `setCodingMode (31 01 10 0D)` 被 ECU 以 `7F 31 31 requestOutOfRange` 拒绝，**0 个编码块写入**，工具复位 MGU，车机恢复原状。日志：`bmw/mgu-plus/data/logs/session-20261122T111332Z-coding-restore.log`。在本车 2021 年固件（BTLD `018.027.040`）上，该序列不成立；继续走开源路线前必须先拿到同固件的 E-Sys 编码 trace。完整交接见 [交接_当前进度与执行路径.md](交接_当前进度与执行路径.md)。

1. 编码恢复、主机重启完成后，先看是否出现磁贴；若没有，再执行 Store 安装（原命令，已确认可用）：
   ```sh
   cd bmw/mgu-store
   ./mgu-install-macos-arm64 first-install \
     --identity ../idrive7-8-ssh-root/data/enrollments/LBV31DU03NS273096/id_rsa \
     --host 169.254.227.133 --port 22 --no-restart-prompt
   ```
2. 查状态：
   ```sh
   ./mgu-install-macos-arm64 status \
     --identity ../idrive7-8-ssh-root/data/enrollments/LBV31DU03NS273096/id_rsa \
     --host 169.254.227.133 --port 22
   ```
   需要重启就重启主机，再跑一次 `status`。
3. 车机 Store → `Settings` → `Add source` → `1031527554/idrive7-apps-zh` → 分支 `main`。
4. **先装 1–2 个试水**（`car_vitals` 或 `service`），确认中文界面、能正常打开；再装其余。
5. `doom` 放最后，且需要车机联网。

---

## 4. 不要做

- **不要再 `flash`**。编码不会因为再刷一次而回来，只会再清一次。
- 编码恢复前，直装路线已铺好的文件保持不动即可（已备份，可用 `deploy-car.sh restore` 回退）；不要再新增应用或改动 MainApp 补丁。
- 不要用新生成的 preflight 覆盖旧备份；旧的 `coding_MGU.txt` 是验收基准。
- 别把 `Restore` 理解成“恢复出厂”：那是恢复刷写前的编码，不是重置车机。
- 编码恢复后，数字钥匙要重新配对；语言等个人设置需要重新调一遍。

---

## 5. 相关文件

- 只读备份：`bmw/idrive7-8-ssh-root/data/preflight/20261122-102752-LBV31DU03NS273096/`
  （`RESTORE.txt`、`coding_MGU.txt`、`backup.json`、`SHA256SUMS`）
- 压缩归档：`bmw/backups/mgu-coding-LBV31DU03NS273096-20261122.tar.gz`
- 上车前准备清单：`宝马idrive7-apps_本地准备清单.md`
- Store 安装器：`bmw/mgu-store/mgu-install-macos-arm64`

参考：

- CAFD 为 `FFFFFFFF` 的含义与处理（注入 CAFD 后再写 NCD）：  
  https://www.bmwonlinetool.com/en/guides/cafd-ffffffff/
- E-Sys 缺失 CAFD 的标准修法（Detect CAF for SWE → 注入 → 设码）：  
  https://f80.bimmerpost.com/forums/showthread/1082681/coding-missing-cafd-files/page/8
- root 工具关于刷写清空编码、BimmerCode 可能部分恢复的说明：  
  `bmw/idrive7-8-ssh-root/README.zh-CN.md`
