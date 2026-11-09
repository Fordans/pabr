# PABR — Pack-level Adaptive Bitrate for DASH Video

> 使用 TCP 包到达时间级细粒度信号进行码率决策的 ABR 智能体框架与数据集。

PABR（**P**ack-level **A**daptive **B**it**R**ate）是一个面向 DASH 流式视频的 ABR 研究项目。与传统 ABR 算法（Pensieve、MPC、BOLA）依赖秒级 / chunk 级吞吐量估计不同，PABR 利用 **TCP 包到达时间序列**这一更细粒度的信号来感知网络状态，目标是更早检测拥塞、更准预测未来吞吐量、在移动网络切换等场景下反应更快。

本仓库包含：

- **数据采集工具**：基于 Linux network namespace + `tc netem` + `tcpdump` 的可复现采集流水线，产出包级 trace。
- **开放数据集**：覆盖多种网络条件（稳定带宽、丢包、jitter、带宽振荡、突发塌缩）的包级 DASH trace，附带 chunk 级元数据与网络条件标签。
- **训练框架**：trace loader、sim env、baseline 算法与 RL 训练脚本。

## 与传统 ABR 的区别

| 维度             | 传统 ABR                    | PABR                                             |
| ---------------- | --------------------------- | ------------------------------------------------ |
| 状态信号         | 吞吐量滑动平均、buffer 占用 | 包到达时间序列、IAT 分布、RTT sample、burst 模式 |
| 粒度             | 秒 / chunk 级               | 包级（毫秒）                                     |
| 对带宽振荡的响应 | 滞后于滑动窗口              | 可在窗口内感知                                   |
| 信号噪声         | 低                          | 高，需特征工程                                   |

## 仓库结构（规划中）

```
pabr/
├── README.md
├── LICENSE
├── .gitignore
├── pyproject.toml              # 待加
├── collect/                    # 数据采集脚本
│   ├── netem_profiles/         # 网络条件配置
│   ├── dash_client/            # 极简顺序下载 client
│   └── run_capture.sh
├── pabr/                       # 训练框架核心包
│   ├── data/                   # trace loader / 格式转换
│   ├── env/                    # sim env（replay packet trace）
│   ├── features/               # 包序列 → 特征向量
│   ├── baselines/              # MPC / BOLA / Pensieve wrapper
│   └── agents/                 # RL 策略
├── data/
│   └── samples/                # 少量小样本 trace（显式 track）
├── docker/                     # 可复现采集环境
└── docs/
```

## 数据集

数据集以**包级 trace 为源、吞吐量 trace 为派生**的方式组织，与 Pensieve / FCC 数据集可互转：

```
trace_xxx/
├── packets.parquet      # 包级：ts, dir, size, tcp_flags, rtt_ms, stream_id
├── chunks.jsonl         # chunk 级元数据
├── network_label.json   # 该 trace 的 netem 配置标签
└── throughput.csv       # 派生：每秒吞吐量（兼容 Pensieve）
```

- **小样本**：随仓库发布，位于 `data/samples/`。
- **全量数据集**：通过 HuggingFace Datasets / Zenodo 发布（链接待补），仓库只放采集脚本以保证可复现。

## 环境要求

- 采集与训练：**Linux**（需 `tc`、`ip`、`tcpdump`；推荐 Ubuntu 22.04+）
- 开发与调试：Windows / macOS 均可
- Python 3.10+

> Windows 下无法使用 `tc netem` 与 network namespace 进行可控采集，建议在远程 Linux 或 WSL2 中运行采集脚本。

## 快速开始

> 采集流水线与训练脚本尚在开发中，以下为预期用法。

```bash
# 克隆
git clone https://github.com/<owner>/pabr.git
cd pabr

# 安装
pip install -e .

# 采集一条 trace（Linux）
bash collect/run_capture.sh --profile collect/netem_profiles/wifi_stable.yaml

# 加载 trace
python -c "from pabr.data import load_packet_trace; t = load_packet_trace('data/samples/demo'); print(len(t.packets))"
```

## 路线图

- [ ] 极简 DASH 顺序下载 client + 包打点
- [ ] network namespace + netem 采集脚本（docker 化）
- [ ] trace 格式 schema 与 loader（含 Pensieve 互转）
- [ ] 包级特征工程（IAT 直方图、burst、RTT 序列）
- [ ] sim env（replay packet trace）
- [ ] baseline：MPC / BOLA / Pensieve wrapper
- [ ] 科学验证实验：包级特征预测吞吐量是否优于 throughput baseline
- [ ] RL 训练（A3C / PPO）
- [ ] 全量数据集发布（HuggingFace / Zenodo）

## 相关工作

- **Pensieve** (Mao et al., SIGCOMM 2017) — 基于 throughput trace 的 RL ABR，本项目的 sim env 设计参考其 `fixed_env.py`。
- **FCC / Oboe** — 已有 throughput 级数据集，本数据集提供更细粒度的包级信号，并与它们可互转。

## 贡献

欢迎 issue / PR。采集脚本与 netem 配置的改进尤其欢迎——可复现性是本数据集的核心价值。

## 许可证

[MIT](LICENSE)
