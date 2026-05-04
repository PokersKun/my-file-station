# Wi-Fi 7 MLO 2.4G 链路流量调度失效问题反馈报告

**设备平台：** 骁龙 8 Gen 5
**系统版本：** Android 16
**问题类型：** Wi-Fi 7 MLO 功能缺陷
**严重程度：** 高（核心宣传功能完全失效）

---

## 一、问题描述

设备连接 Wi-Fi 7 AP（SSID: PokerS7，WPA3-SAE）后，通过 MLO（Multi-Link Operation）同时建立 2.4GHz 和 5GHz 双链路连接。Framework 层显示两条链路均处于 `MLO_LINK_STATE_ACTIVE` 状态，但 2.4GHz 链路的 Tx/Rx 速率始终为 0Mbps，通过 iperf3 在应用层进行实际流量测试，确认 2.4GHz 链路全程无任何数据传输，所有流量均由 5GHz 单链路承载，MLO 多链路并发能力完全无法发挥。

---

## 二、环境信息

**AP 信息**
- AP MLD Address: 56:f7:70:b8:ab:c8
- 2.4GHz 链路：Channel 11，AP MAC: 44:f7:70:b8:ab:ca
- 5GHz 链路：Channel 44（5220MHz），AP MAC: 44:f7:70:b8:ab:cb
- TID-To-Link Negotiation: AP 侧支持（`Is TID-To-Link negotiation supported by the AP: true`）

**STA 连接状态**
- Wi-Fi Standard: 11be（Wi-Fi 7）
- Security: WPA3-SAE
- 5GHz 链路：RSSI -25dBm，Tx 1441Mbps，Rx 1296Mbps（正常）
- 2.4GHz 链路：RSSI 在不同时间点采样为 -36dBm 和 -31dBm（信号正常），但 Tx 0Mbps，Rx 0Mbps（异常）

---

## 三、调试过程与关键数据

**步骤 1：确认链路状态**

通过 `adb shell cmd wifi status` 确认两条 MLO 链路均为 ACTIVE，排除链路未建立的可能性。注意到 2.4GHz 链路的 RSSI 在第二次采样时变为 -127，这是驱动层的无效值占位符（sentinel value），说明驱动已停止对该链路进行射频测量，是底层将其挂起的信号。

```
MloLink{2.4GHz, channel: 11, id: 0, state: MLO_LINK_STATE_ACTIVE,
        RSSI: -127, Rx: 0Mbps, Tx: 0Mbps}   ← 速率为 0，RSSI 无效
MloLink{5GHz, channel: 44, id: 1, state: MLO_LINK_STATE_ACTIVE,
        RSSI: -27, Rx: 1296Mbps, Tx: 1441Mbps}   ← 正常
```

**步骤 2：排查 HAL 层数据上报**

通过 `adb shell dumpsys wifi` 搜索 `WifiLinkLayerStats`、`txMpdu rxMpdu` 等字段，均无任何输出，说明 HAL 层完全没有向 Framework 上报 per-link 的流量统计数据。

**步骤 3：发现芯片能力声明缺失**

在 `mDebugChipsInfo` 中发现关键异常：

```
chipCapabilities = {7, 8, 26, 30}
radioCombinations = null        ← 未声明频段组合并发能力
bandCombinations  = null        ← 未声明频段组合并发能力
```

`radioCombinations` 和 `bandCombinations` 是 HAL 向 Android Wi-Fi Framework 声明芯片支持哪些频段同时工作的能力描述表。两者均为 null，意味着 Framework 无法感知该芯片具备 2.4G + 5G 同时收发的能力，多链路流量调度器在没有这份能力表的情况下，只能保守地将所有流量分配给主链路（5GHz）。

**步骤 4：定位 QHCP 服务缺失**

在 vendor 日志中发现大量持续性报错：

```
ServiceManagerCppClient: Waited one second for vendor.qti.qhcp.IQHDC/default
Unable to set property "ctl.interface_start" to "aidl/vendor.qti.qhcp.IQHDC/default"
```

执行服务检查：

```
adb shell service check vendor.qti.qhcp.IQHDC/default
→ Service vendor.qti.qhcp.IQHDC/default: not found
```

进一步确认：

```
adb shell find /vendor -name "*qhcp*"
→ （无任何输出）
```

`/vendor` 分区下不存在任何 qhcp 相关文件，确认 **QHCP 服务不是启动失败，而是根本未被集成到 vendor 镜像中**。

---

## 四、根本原因

`vendor.qti.qhcp.IQHDC`（Qualcomm Heterogeneous Connectivity Platform）是高通骁龙平台上负责 Wi-Fi 7 MLO 跨链路流量调度的核心 vendor 服务。该服务缺失导致以下完整故障链：

```
QHCP 服务未集成于 vendor 分区
            ↓
HAL 无法上报 radioCombinations / bandCombinations
            ↓
LinkLayerStats per-link 数据缺失
            ↓
WifiScoreCard per-link 评分表为空，无法进行多链路调度决策
            ↓
Framework 将全部流量调度至主链路（5GHz）
            ↓
2.4GHz 链路协议层 ACTIVE，但永远不承载任何数据
```

这是一个 vendor 集成缺陷，与 Android Framework 层和 AP 侧无关，AP 已正确宣告 TLM 支持能力，Framework 逻辑也正常运行，问题完全在于 vendor 分区缺少必要组件。

---

## 五、影响范围

所有使用该 vendor 镜像、连接 Wi-Fi 7 AP 并启用 MLO 的设备，2.4GHz 辅助链路均不会承载任何流量，MLO 多链路聚合带宽能力完全失效，实际 Wi-Fi 性能退化为单链路 5GHz 水平，与 Wi-Fi 6 无差异。

---

## 六、修复建议

需要在 vendor 分区集成以下高通 QHCP 组件并确保其在系统启动时正常注册：

- `vendor.qti.qhcp.IQHDC` AIDL 服务及其依赖库
- 对应的 SELinux policy 条目（当前 `ctl.interface_start` 的 property 写入也被拒绝）
- HAL 侧 `radioCombinations` 和 `bandCombinations` 的正确填充逻辑

修复验证方式：执行 `adb shell service check vendor.qti.qhcp.IQHDC/default` 返回服务已注册，且 `dumpsys wifi` 中 `radioCombinations` 和 `bandCombinations` 不再为 null，iperf3 测试下 2.4GHz 链路出现实际流量。