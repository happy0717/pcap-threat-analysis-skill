# PCAP Threat Analysis

基于 AI 的 pcap 流量安全威胁分析技能。自动解析 pcap 文件的协议、载荷、流状态，并以网络安全工程师视角逐个分析流量中是否存在攻击行为。

## 目录结构

```
pcap-threat-analysis/
├── SKILL.md              # Skill 定义文件（AI Agent 使用）
├── parse_pcap_win.exe    # Windows amd64 解析工具
└── parse_pcap_linux      # Linux amd64 解析工具
```

## 工作原理

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│  pcap 文件   │ ──▶ │  parse_pcap 工具  │ ──▶ │   JSON 结果   │
│  或目录      │     │  (自动解析)       │     │  + summary    │
└──────────────┘     └─────────────────┘     └──────┬───────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │  AI 威胁分析  │
                                            │ (模型逐流检查) │
                                            └──────┬───────┘
                                                   │
                                                   ▼
                                           ┌───────────────┐
                                           │  威胁分析报告  │
                                           └───────────────┘
```

1. **自动解析阶段**：`parse_pcap` 工具读取 pcap，输出结构化 JSON
   - 提取每条流（四元组：源IP、源端口、目的IP、目的端口、传输协议）
   - 识别应用层协议：HTTP、TLS、DNS、SSH、SMB、DCERPC、PostgreSQL、Telnet、MSSQL、Redis、MySQL、WebSocket
   - 提取每条流的 payload（HEX + 可读文本）
   - HTTP 流自动重组跨包载荷，按 `requestHeader` / `requestBody` / `responseHeader` / `responseBody` 四段拆分
   - 检查流问题：TCP 握手/挥手完整性、MTU、回环流量、SEQ 丢包、抓包截断、流量乱序、HTTP 协议异常等
2. **AI 分析阶段**：模型作为安全工程师逐流审查，覆盖 40+ 种威胁类型

## 快速开始

### 前置要求

- Windows 或 Linux (amd64)
- 无需安装任何运行时依赖

### 使用方式

通过 AI Agent 调用 skill 时，只需提供 pcap 路径：

> "帮我分析这个 pcap 目录 `G:\data\pcaps`"

Agent 会自动完成解析和分析。

### 手动使用 parse_pcap 工具

```bash
# Windows
.\parse_pcap_win.exe <pcap文件或目录路径>

# Linux
./parse_pcap_linux <pcap文件或目录路径>
```

工具输出：

```
当前工作目录/
└── pcap_parse_20260611_172028/
    ├── pcap_parse_summary.json   # 汇总文件
    ├── sample1.json              # 每个 pcap 的解析结果
    ├── sample2.json
    └── ...
```

## JSON 输出格式

### 汇总文件 `pcap_parse_summary.json`

```json
{
  "source_path": "/path/to/pcaps",
  "total_pcap": 5,
  "pcap_with_problem": 2,
  "pure_tls_count": 1,
  "items": [
    {
      "pcap_name": "sample.pcap",
      "file_problems": ["缺少eth层"],
      "protocols": ["HTTP", "TLS"],
      "only_tls": false
    }
  ]
}
```

### 单个 pcap 分析结果 `<name>.json`

```json
{
  "pcap_name": "sample.pcap",
  "total_packets": 150,
  "total_flows": 4,
  "file_problems": ["缺少eth层", "MTU大于1514"],
  "flows": [
    {
      "flow_id": 1,
      "src_ip": "192.168.1.100",
      "src_port": 49152,
      "dst_ip": "93.184.216.34",
      "dst_port": 80,
      "transport_protocol": "TCP",
      "app_protocols": ["HTTP"],
      "packet_count": 45,
      "total_bytes": 32800,
      "tcp_handshake": "complete",
      "tcp_teardown": "incomplete",
      "payloads": [
        {
          "direction": "request",
          "section": "requestHeader",
          "hex": "474554202f20485454502f312e31...",
          "text": "GET / HTTP/1.1\r\nHost: example.org\r\n..."
        },
        {
          "direction": "request",
          "section": "requestBody",
          "hex": "636d643d77686f616d69",
          "text": "cmd=whoami"
        },
        {
          "direction": "response",
          "section": "responseHeader",
          "hex": "485454502f312e3120323030...",
          "text": "HTTP/1.1 200 OK\r\n..."
        },
        {
          "direction": "response",
          "section": "responseBody",
          "hex": "726f6f74",
          "text": "root"
        }
      ],
      "problems": ["不完整的挥手"]
    }
  ]
}
```

**payload 字段说明：**

| 字段 | 说明 |
|------|------|
| `direction` | 载荷方向：`request` / `response` |
| `section` | HTTP 载荷分段标识（`requestHeader`/`requestBody`/`responseHeader`/`responseBody`），仅 HTTP 流存在；非 HTTP 流按原始 TCP 包顺序输出 |
| `hex` | 载荷的十六进制编码 |
| `text` | 载荷的可读文本（非可打印字符以 `.` 替代） |
| `no_separator` | 布尔标记，HTTP 头部未以 `\r\n\r\n` 结束时为 true |
| `seq_gap` | 同方向前后两个 TCP 段之间的序列号缺失字节数 |
| `truncated` / `truncated_len` / `declared_len` | 抓包截断标记 |

**tcp_handshake / tcp_teardown 取值：** `complete` / `incomplete` / `n/a`

## 支持的协议检测

所有协议检测均为**纯内容/结构特征识别**，不依赖端口号：

| 协议 | 检测特征 |
|------|----------|
| HTTP | 方法前缀 (GET/POST/...) + HTTP 响应行 + HTTP/2 前导帧 |
| TLS | ContentType (0x14-0x17) + 版本号 + 记录长度校验 |
| DNS | DNS 报文头 (ID/Flags/计数) + QNAME 标签解析 |
| SSH | SSH banner + SSHv2 二进制协议识别 |
| SMB | `\xFFSMB`/`\xFESMB` 魔数 + NetBIOS 封装 |
| DCERPC | 版本字段 + packet_type 校验 |
| PostgreSQL | 变长长度前缀 + 协议版本 3.0 + 消息 tag |
| Telnet | IAC 命令序列 + 登录提示词 |
| MSSQL (TDS) | Token 类型 + TDS 长度前缀 |
| Redis | RESP 协议标记 + 内联命令前缀 |
| MySQL | 协议版本 + 命令包识别 |
| WebSocket | Upgrade 头 + Sec-WebSocket-* 头 |

## 威胁类型（40+ 类）

| 类别 | 类别 | 类别 |
|------|------|------|
| SQL 注入 | 命令执行 | 代码执行 |
| 跨站脚本攻击（XSS） | 跨站请求伪造（CSRF） | 目录遍历 |
| 文件包含 | 文件上传 | 文件下载 |
| 文件读取 | 文件写入 | webshell 上传 |
| webshell 利用 | 溢出攻击 | 拒绝服务 |
| 信息泄露 | 敏感信息/重要文件泄漏 | 弱口令 |
| 暴力猜解 | 非授权访问/权限绕过 | 后门程序 |
| 窃密木马 | 间谍软件 | 键盘记录 |
| 僵尸网络 | 网络蠕虫 | 网络钓鱼 |
| 挖矿 | 黑市工具 | 浏览器劫持 |
| 逻辑/设计错误 | 配置不当/错误 | 默认配置不当 |
| 系统/服务配置不当 | 权限许可和访问控制 | URL 跳转 |
| 工控漏洞 | 协议异常 | 其他攻击 |

## 文件级问题检测

- 文件过小 / 无法读取
- 不是 tcpdump/pcap/pcapng 格式（魔数校验）
- 缺少 eth 层
- 非 TCP/UDP 流量
- 存在回环流量（127.0.0.0/8, ::1/128, 0.0.0.0/8）

## 流级问题检测

| 问题提示 | 说明 |
|----------|------|
| `不完整的握手` | TCP 三次握手不完整（SYN → SYN+ACK → ACK） |
| `不完整的挥手` | TCP 四次挥手不完整（FIN+ACK → ACK → FIN+ACK → ACK） |
| `MTU大于1514` | 存在超过 1514 字节的超大包 |
| `N个包抓包截断/丢包` | IP Length 声明的长度大于实际捕获字节数 |
| `N处seq丢包` | 同方向前后 TCP 段序列号存在间隙 |
| `pcap不完整 tcp连接未关闭` | 握手完整但无 FIN/RST，抓包截止时连接仍在进行 |
| `pcap不完整 缺少tcp握手` | 无 SYN 包，未捕获到握手 |
| `流量乱序` | 请求载荷出现在响应载荷之后（请求体滞后于响应到达） |
| `HTTP头部行分隔未使用CRLF` | HTTP 头部各字段行使用 LF 而非 CRLF 分隔 |
| `HTTP头部未以CRLFCRLF结束` | HTTP 头部缺少 `\r\n\r\n` 终止符（头部与体部之间无空行分隔） |
| `HTTP协议有问题(缺少Content-Length)` | 请求有 body 但头部缺少 Content-Length（仅研判请求方向） |

## 安装到 AI Agent

1. 将 `SKILL.md` 放入 Agent 的 skills 目录（通常为 `.trae/skills/pcap-threat-analysis/`）
2. 确保 `parse_pcap_win.exe` 和 `parse_pcap_linux` 与 `SKILL.md` 在同一目录
3. Agent 运行时根据操作系统自动选择对应二进制

如果 Agent 无法直接访问 skill 目录下的二进制，会被引导将工具复制到工作区：

```
请将 pcap-threat-analysis 目录下的 parse_pcap_win.exe / parse_pcap_linux
复制到当前工作目录下，然后告诉我继续。
```

## 使用
1.下载 https://github.com/happy0717/pcap-threat-analysis-skill/tree/main/pcap-threat-analysis 文件夹中的skill内容(包括两个可执行文件)，根据Agent需要直接放在工作区或者打包成zip上传到Agent工具。

2.提示词：

仅分析pcap：```帮我使用 pcap-threat-analysis skill 分析一下这个 pcap```

对流量检测设备告警进行研判：```帮我使用 pcap-threat-analysis skill 分析一下这个 pcap 是否符合告警"{告警名称}", 判断依据是什么？，可以参考"{规则描述}"进行分析和解释。```

注意：该skill主要用途是将pcap转成模型可读的文本数据，如果仅对流量检测设备的告警研判，正常Agent可以支持传入文本格式的流量日志、图片OCR等，这种情况可以不用使用该SKILL，直接使用提示词```分析一下这个流量是否符合告警"{告警名称}", 判断依据是什么？，可以参考"{规则描述}"进行分析和解释。```


## License

Happy0717's friends internal use.
