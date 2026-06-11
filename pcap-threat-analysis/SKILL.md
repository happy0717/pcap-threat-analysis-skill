---
name: "pcap-threat-analysis"
description: "PCAP 威胁分析技能。解析 pcap 流量文件，识别协议、载荷、流问题，并以网络安全工程师视角逐流分析威胁内容。Invoke when user wants to analyze pcap files for security threats, detect malicious traffic, or perform network forensics."
---

# PCAP Threat Analysis

对 pcap 流量文件进行深度解析和威胁分析。该 skill 包含两个阶段：
1. **自动解析阶段**：使用 `parse_pcap` 工具解析 pcap 生成结构化 JSON
2. **AI 分析阶段**：模型逐文件、逐流分析 JSON 中的流量数据，识别安全威胁

## 工具准备

### 二进制文件说明

Skill 附带两个平台的 `parse_pcap` 二进制文件：

| 文件 | 平台 |
|------|------|
| `parse_pcap_win.exe` | Windows amd64 |
| `parse_pcap_linux` | Linux amd64 |

> Skill 安装目录即这些二进制文件所在的目录。调用方需将 `{skill_path}` 替换为实际路径。

### 工具可用性检查（重要）

在执行 `parse_pcap` 之前，**必须**检查工具二进制文件是否存在于工作目录中：

1. 先尝试在 skill 安装目录下直接运行二进制（路径参考：`{skill_path}/parse_pcap_win.exe` 或 `{skill_path}/parse_pcap_linux`）。
2. 如果提示 "not found" 或找不到文件，说明工具二进制不在当前工作区的可访问范围内。此时 agent 需要**自行将二进制文件从 skill 目录复制到当前工作区**：
   - 使用文件复制命令将 `{skill_path}/parse_pcap_win.exe` 或 `{skill_path}/parse_pcap_linux` 复制到工作区根目录
   - 然后再从工作区执行该工具
3. 如果 agent 的文件操作权限受限导致复制失败，则**引导用户手动复制**：
   > "请将 `{skill_path}` 目录下的 `parse_pcap_win.exe` / `parse_pcap_linux` 复制到当前工作目录下，然后告诉我继续。"

## 执行流程

### 步骤 0：确保工具可用

根据操作系统选择对应的二进制名称，检查当前目录下是否存在该文件。若不存在，按上述"工具可用性检查"流程处理。

### 步骤 1：接收输入路径

从用户消息中获取 pcap 文件或目录路径。如果用户未提供，主动询问用户要分析的 pcap 路径。

### 步骤 2：执行 parse_pcap 工具

根据操作系统选择正确的二进制，传入 pcap 路径运行：

```bash
# Windows
.\parse_pcap_win.exe <pcap文件或目录路径>

# Linux
./parse_pcap_linux <pcap文件或目录路径>
```

该工具会在当前工作目录下生成一个 `pcap_parse_YYYYMMDD_HHMMSS/` 文件夹，包含：
- 每个 pcap 对应的 `<pcap名称>.json` 解析结果
- `pcap_parse_summary.json` 汇总文件

### 步骤 3：读取解析结果

首先读取 `pcap_parse_summary.json` 了解整体情况，然后逐个读取各 pcap 的 JSON 文件。

**summary 文件结构参考：**
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

**单个 pcap JSON 结构参考（关键字段）：**
```json
{
  "pcap_name": "sample.pcap",
  "total_packets": 150,
  "total_flows": 4,
  "file_problems": ["缺少eth层"],
  "flows": [
    {
      "flow_id": 1,
      "src_ip": "192.168.1.100",
      "src_port": 49152,
      "dst_ip": "10.0.0.5",
      "dst_port": 80,
      "transport_protocol": "TCP",
      "app_protocols": ["HTTP"],
      "tcp_handshake": "complete",
      "tcp_teardown": "complete",
      "problems": [],
      "payloads": [
        {
          "direction": "request",
          "hex": "474554202f...",
          "text": "GET /admin/exec?cmd=whoami HTTP/1.1\r\n..."
        }
      ]
    }
  ]
}
```

### 步骤 4：逐流安全分析

以**资深网络安全工程师**的视角，对每个 pcap 的每一条流进行威胁分析。重点关注：

1. **协议异常**：协议字段是否合法、是否存在畸形报文
2. **已知攻击模式**：payload 中是否包含攻击特征
3. **可疑通信行为**：异常端口、异常数据流向、心跳/信标模式
4. **敏感内容泄露**：明文密码、内网 IP、密钥信息
5. **加密流量风险**：纯 TLS 流量无法分析内容，但可根据握手特征（SNI、证书）判断可疑连接

**威胁类型分类参考（分析时请从以下类型中选择）：**

| 类别 | 说明 |
|------|------|
| SQL 注入 | SQL 注入攻击 payload |
| 命令执行 | 系统命令执行 payload |
| 代码执行 | 远程代码执行 payload |
| 跨站脚本攻击（XSS） | XSS 注入 payload |
| 跨站请求伪造（CSRF） | CSRF 攻击 |
| 目录遍历 | 路径遍历攻击 |
| 文件包含 | LFI/RFI 文件包含 |
| 文件上传 | 恶意文件上传 |
| 文件下载 | 非授权文件下载 |
| 文件读取 | 任意文件读取 |
| 文件写入 | 任意文件写入 |
| webshell 上传 | WebShell 上传 |
| webshell 利用 | WebShell 交互流量 |
| 溢出攻击 | 缓冲区溢出攻击 |
| 拒绝服务 | DoS/DDoS 攻击 |
| 信息泄露 | 敏感信息泄露 |
| 敏感信息/重要文件泄漏 | 密钥、凭证、配置文件泄露 |
| 弱口令 | 弱密码或默认口令 |
| 暴力猜解 | 暴力破解尝试 |
| 非授权访问/权限绕过 | 越权访问 |
| 后门程序 | 后门通信 |
| 窃密木马 | 数据窃取木马 |
| 间谍软件 | 间谍软件行为 |
| 键盘记录 | 键盘记录器 |
| 僵尸网络 | Botnet C2 通信 |
| 网络蠕虫 | 蠕虫传播 |
| 网络钓鱼 | 钓鱼攻击 |
| 挖矿 | 加密货币挖矿 |
| 黑市工具 | 黑市工具流量 |
| 浏览器劫持 | 浏览器劫持 |
| 逻辑/设计错误 | 逻辑漏洞 |
| 配置不当/错误 | 配置错误 |
| 默认配置不当 | 默认配置未修改 |
| 系统/服务配置不当 | 服务配置缺陷 |
| 权限许可和访问控制 | ACL/权限问题 |
| URL 跳转 | 未验证的 URL 跳转 |
| 工控漏洞 | ICS/SCADA 工控协议漏洞 |
| 协议异常 | 协议字段异常或违规 |
| 其他攻击 | 无法归类的攻击行为 |

> 如果未发现任何威胁，不要强行归类。正常流量应标记为"无威胁"。

### 步骤 5：汇总分析结果

所有 pcap 分析完成后，向用户呈现完整的分析报告。报告格式如下：

---

## PCAP 威胁分析报告

**分析时间**：`YYYY-MM-DD HH:MM:SS`
**分析文件数**：`N` 个
**有威胁的 pcap**：`M` 个
**纯 TLS 流量**：`K` 个

### 分析明细

#### 1. `pcap_name_1.pcap`
- **文件问题**：缺少eth层, MTU大于1514
- **协议类型**：HTTP, TLS
- **威胁内容**：**有威胁**
  - **流 1** (192.168.1.100:54321 → 10.0.0.5:8080, HTTP)
    - **威胁类型**：命令执行, 目录遍历
    - **判断依据**：HTTP GET 请求中包含 `/../../etc/passwd` 路径遍历和 `cmd=whoami` 命令执行 payload，payload hex 为 `...`
  - **流 3** (192.168.1.100:54322 → 10.0.0.5:443, TLS)
    - **威胁类型**：无威胁
    - **判断依据**：纯 TLS 加密流量，握手正常，SNI 为 `example.org`，无异常特征

#### 2. `pcap_name_2.pcap`
- **文件问题**：无
- **协议类型**：TLS
- **威胁内容**：**仅 TLS 加密流量，无法进行内容层面分析**
- **流量概况**：3 条 TLS 流，握手均完整，目标 SNI: `api.example.com`, `cdn.cloud.com`

#### 3. `pcap_name_3.pcap`
- **文件问题**：无
- **协议类型**：DNS, HTTP
- **威胁内容**：**无威胁**
- **流量概况**：正常的 DNS 查询和 HTTP Web 浏览流量

---

### 汇总统计

| 指标 | 数值 |
|------|------|
| 总 pcap 数 | N |
| 有 file_problems | X |
| 有威胁内容 | Y |
| 纯 TLS（无法分析） | K |
| 无威胁 | Z |

### 威胁类型分布

| 威胁类型 | 出现次数 |
|----------|----------|
| 命令执行 | 3 |
| SQL 注入 | 1 |
| ... | ... |

---

## 注意事项

1. **parse_pcap 工具超时**：大文件解析可能需要较长时间，等待工具完成后再进行下一步。
2. **加密流量处理**：TLS/加密流量中无法分析 payload 内容，仅根据元数据（SNI、证书、端口、握手模式）做辅助判断。
3. **误报可能性**：AI 基于 payload 文本做出的判断仅供参考，建议人工复核确认。
4. **JSON 文件较大时**：如果单个 pcap 的 JSON 内容过大（超过5000行），优先分析前几条流的关键 payload。
5. **分析顺序**：先看 summary 了解全貌，再对有问题的或有明文协议的 pcap 优先深入分析。
