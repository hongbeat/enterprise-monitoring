# 🛰️ 企业级主机监控系统（Prometheus + Alertmanager + Grafana）

> 在 **CentOS Stream 9 / SELinux Enforcing** 全防护环境下，从零搭建的端到端主机监控栈：
> **指标采集 → 阈值告警 → 邮件通知（firing / resolved 全生命周期）→ 可视化大屏**。
> 全程未关闭 SELinux，所有组件在最高安全策略下跑通。

---

## 🧩 技术栈

| 组件 | 版本 | 职责 |
|---|---|---|
| **Prometheus** | 2.x（二进制部署） | 指标采集与存储（15s 抓取间隔） |
| **node_exporter** | 1.x（二进制部署） | 主机指标暴露（:9100/metrics） |
| **Alertmanager** | 0.x（二进制部署） | 告警聚合 + SMTP 邮件通道（QQ 邮箱 465） |
| **Grafana** | 11.x（RPM 部署） | 可视化大屏（社区 1860 模板） |
| **SELinux** | Enforcing | 安全加固（自定义策略模块 + 官方布尔值） |

## 🏗️ 架构
node_exporter(:9100)
│ /metrics
▼
Prometheus(:9090) ──15s scrape──► rules/host_alerts.yml（五条告警规则）
│ │
│ alert │
▼ ▼
Alertmanager(:9093) ──SMTP──► 📬 邮件（firing 红 / resolved 绿）
│
│ datasource
▼
Grafana(:3000) ── 1860 主机大屏


## ✅ 告警规则（`prometheus/rules/host_alerts.yml`）

| 规则 | 触发条件 | 级别 | 说明 |
|---|---|---|---|
| `HostHighCpuLoad` | CPU > 80% 持续 5m | warning | 高负载预警 |
| `HostOutOfMemory` | 可用内存 < 15% | critical | 内存耗尽 |
| `HostOutOfDiskSpace` | 磁盘使用 > 85% | critical | 磁盘告急 |
| `HostSwapFillingUp` | Swap 持续增长 | warning | 交换分区膨胀 |
| **`HostDown`** | 目标 1 分钟无法抓取 | **critical** | **节点宕机（闭环验证：停 exporter → 红邮件 → 恢复 → 绿邮件）** |

## 🛡️ 亮点：SELinux Enforcing 下不关安全的加固

> **全程 `getenforce = Enforcing`，未执行 `setenforce 0`，未修改 `/etc/selinux/config`。**

- **自定义策略模块**（`selinux/grafana-custom.te` → 编译为 `grafana-custom.pp`）：
  为 Prometheus / Alertmanager 的网络访问编写并加载 SELinux 策略，
  用 `audit2allow` 从拒绝日志生成规则，`semodule -i` 加载生效。
- **官方布尔值走"正门"**：
  `setsebool -P grafana_can_tcp_connect_prometheus_port on`
  让 Grafana 在 Enforcing 下合法连接 Prometheus :9090，**不写一条自定义规则**。
- **可复现**：仓库内附 `.te` 源文件 + `.pp` 编译产物 + `booleans_status.txt` 状态快照，
  任何人 clone 后可在自己的 Enforcing 系统上完整复现加固过程。

## 🔧 实战排错记录

| 问题 | 根因 | 解法 |
|---|---|---|
| 二进制下载断流 | 网络不稳 / 源不可达 | 校验 SHA256 + 换源重下 |
| 跨版本升级后 Prometheus 启动失败 | 配置格式不兼容 | `promtool check config` 取证定位 |
| **真实磁盘告警**：/var 88% | PCP 日志堆积 | 定位 + 清理 → 42%，告警自动 resolved |
| **HostDown 闭环验证** | — | 手动停 node_exporter → 收 firing 红邮件 → 恢复 → 5min 后收 resolved 绿邮件 |
| Grafana 连不上 Prometheus | SELinux 拒绝 TCP 连接 | 启用官方布尔值（见上） |

## 🔒 安全实践：配置模板化与密钥脱敏

- 仓库内 `alertmanager/alertmanager.yml.example` 为**脱敏模板**，
  授权码已替换为 `<YOUR_QQ_MAIL_AUTH_CODE>` 占位符。
- 真实配置留在服务器 `/etc/alertmanager/alertmanager.yml`，**不入库**。
- `.gitignore` 拦截 `*.bak` 备份文件与真实配置，杜绝敏感信息泄露。

> **部署时**：复制 `alertmanager.yml.example` 为 `alertmanager.yml`，
> 将 `<YOUR_QQ_MAIL_AUTH_CODE>` 替换为你自己的 QQ 邮箱授权码即可。

## 📂 目录结构
enterprise-monitoring/
├── README.md ← 本文件
├── .gitignore ← 拦截备份文件与真实密钥
├── prometheus/
│ ├── prometheus.yml ← 主配置（scrape 间隔、rules 加载路径）
│ └── rules/
│ ├── host_alerts.yml ← 五条告警规则
│ └── placeholder.yml ← 空目录占位（避免 Prometheus 启动报 no rule files）
├── alertmanager/
│ └── alertmanager.yml.example ← 脱敏模板（授权码已替换为占位符）
├── grafana/
│ └── dashboards/
│ └── host-dashboard.json ← 1860 主机大屏导出（可复现）
├── systemd/
│ ├── prometheus.service ← 四个服务均 enabled（开机自启）
│ ├── node_exporter.service
│ ├── alertmanager.service
│ └── grafana-server.service
├── selinux/
│ ├── grafana-custom.te ← 自定义策略源文件
│ ├── grafana-custom.pp ← 编译产物（semodule -i 可直接加载）
│ └── booleans_status.txt ← 布尔值 + getenforce 状态快照
├── docs/
│ └── troubleshooting.md ← 排错实录（待补充）
└── screenshots/ ← 告警邮件 + Grafana 大屏截图（简历视觉证据）


## 🚀 快速部署

```bash
# 1. 安装二进制（prometheus / node_exporter / alertmanager → /usr/local/bin/）
# 2. 安装 Grafana（RPM）
# 3. 复制配置
cp prometheus/prometheus.yml       /etc/prometheus/
cp prometheus/rules/*.yml          /etc/prometheus/rules/
cp alertmanager/alertmanager.yml.example  /etc/alertmanager/alertmanager.yml
#    ↑ 编辑，替换 <YOUR_QQ_MAIL_AUTH_CODE> 为你的授权码
# 4. 复制 systemd 服务文件 → /etc/systemd/system/
cp systemd/*.service               /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now prometheus node_exporter alertmanager grafana-server
# 5. SELinux 加固（Enforcing 下）
semodule -i selinux/grafana-custom.pp
setsebool -P grafana_can_tcp_connect_prometheus_port on
# 6. Grafana 导入大屏
#    浏览器 :3000 → Dashboards → Import → 上传 grafana/dashboards/host-dashboard.json

📬 告警邮件样例
Firing（红色）：[FIRING] HostDown @ localhost:9100 — 节点 1 分钟不可达
Resolved（绿色）：[RESOLVED] HostDown @ localhost:9100 — 恢复后 5 分钟自动发送
六封邮件（CPU / 内存 / 磁盘 / Swap / HostDown × firing+resolved）均已实测收到。
