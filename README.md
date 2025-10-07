# 网络数据包分析工具

## 项目概述

这是一个功能完整的网络数据包分析工具，提供实时数据包捕获、协议分析、数据可视化和导出功能。工具采用Python开发，包含Web界面、RESTful API和完整的数据处理管道。

## 项目架构

### 核心组件

```
network/
├── web_interface.py          # Flask Web服务和API接口
├── packet_analyzer.py        # 数据包分析和处理核心
├── network_monitor.py        # 网络监控和捕获模块
├── traffic_visualizer.py     # 流量可视化工具
├── templates/index.html      # Web界面前端
├── packet_capture.db        # SQLite数据库
└── requirements.txt         # 项目依赖
```

### 技术栈

- **后端**: Python 3.7+, Flask, SQLite
- **前端**: HTML5, CSS3, JavaScript, Chart.js
- **网络库**: scapy, psutil
- **数据可视化**: Chart.js, D3.js
- **数据库**: SQLite3

## 功能特性

### 1. 实时数据包捕获

#### 核心功能
- **多协议支持**: TCP, UDP, ICMP, ARP, IP v4/v6
- **智能过滤**: 基于协议、端口、IP地址的过滤器
- **性能优化**: 异步捕获，高效数据包处理
- **模拟模式**: Windows环境下自动生成模拟数据包

#### 数据包处理流程
```
网络接口 → 数据包捕获 → 协议解析 → 数据存储 → 实时显示
```

#### 捕获配置
```python
{
    "max_packets": 1000,      # 最大捕获数量
    "filter_protocol": "TCP", # 协议过滤器
    "capture_duration": 60,   # 捕获时长(秒)
    "buffer_size": 65536      # 缓冲区大小
}
```

### 2. 协议分析与识别

#### 支持的协议
- **网络层**: IP, IPv6, ARP, ICMP
- **传输层**: TCP, UDP
- **应用层**: HTTP, HTTPS, DNS, DHCP

#### 分析指标
- **流量统计**: 数据包数量、字节数、速率
- **协议分布**: 各协议占比分析
- **端口分析**: 常用端口识别
- **IP地理**: 源IP和目标IP统计

### 3. Web界面与可视化

#### 界面组件
- **实时仪表盘**: 动态更新的统计图表
- **数据包列表**: 详细的数据包信息展示
- **协议分布图**: 饼图显示协议占比
- **流量趋势图**: 时间序列流量图表
- **网络拓扑**: 网络连接关系图

#### 实时更新机制
```javascript
// 使用WebSocket或定时轮询实现实时更新
setInterval(() => {
    fetch('/api/get_stats')
        .then(response => response.json())
        .then(data => updateCharts(data));
}, 1000);
```

### 4. 数据导出与报告

#### 导出格式
- **JSON**: 结构化数据，便于程序处理
- **CSV**: 表格格式，适合Excel分析
- **PCAP**: 标准网络数据包格式，可用Wireshark打开
- **PDF**: 自动生成的分析报告

#### 导出内容
- **数据包详情**: 完整的数据包信息
- **统计摘要**: 流量统计和协议分析
- **会话记录**: 历史捕获会话信息
- **图表截图**: 可视化图表导出

### 5. API接口

#### RESTful API设计
```
GET  /api/get_stats              # 获取统计信息
GET  /api/get_network_info       # 获取网络信息
GET  /api/get_recent_packets     # 获取最近数据包
POST /api/start_capture          # 开始捕获
POST /api/stop_capture           # 停止捕获
GET  /api/export_packets/<format> # 导出数据
GET  /api/get_capture_sessions   # 获取会话列表
```

#### API响应格式
```json
{
    "status": "success",
    "data": {
        "total_packets": 100,
        "protocol_counts": {"TCP": 60, "UDP": 40},
        "capture_rate": "1.5 packets/sec"
    },
    "timestamp": "2025-10-07T10:30:00Z"
}
```

## 技术实现细节

### 数据包捕获模块

```python
class PacketCapture:
    def __init__(self, interface=None, filter_expr=""):
        self.interface = interface
        self.filter_expr = filter_expr
        self.capture_active = False
        self.packet_count = 0
        
    def start_capture(self, callback=None):
        """开始数据包捕获"""
        if os.name == 'nt':  # Windows系统
            self._start_simulation_mode(callback)
        else:
            self._start_real_capture(callback)
    
    def _process_packet(self, packet):
        """处理单个数据包"""
        analysis = {
            'timestamp': time.time(),
            'protocol': self._identify_protocol(packet),
            'src_ip': packet.src if hasattr(packet, 'src') else '',
            'dst_ip': packet.dst if hasattr(packet, 'dst') else '',
            'src_port': packet.sport if hasattr(packet, 'sport') else '',
            'dst_port': packet.dport if hasattr(packet, 'dport') else '',
            'size': len(packet),
            'flags': self._extract_flags(packet)
        }
        return analysis
```

### 数据存储设计

```python
class DatabaseManager:
    def __init__(self, db_path="packet_capture.db"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._create_tables()
    
    def _create_tables(self):
        """创建数据库表结构"""
        self.conn.execute('''
            CREATE TABLE IF NOT EXISTS packets (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                session_id TEXT,
                timestamp REAL,
                protocol TEXT,
                src_ip TEXT,
                dst_ip TEXT,
                src_port INTEGER,
                dst_port INTEGER,
                size INTEGER,
                raw_data BLOB,
                analysis_data TEXT
            )
        ''')
        
        self.conn.execute('''
            CREATE TABLE IF NOT EXISTS sessions (
                session_id TEXT PRIMARY KEY,
                start_time REAL,
                end_time REAL,
                total_packets INTEGER,
                filter_config TEXT
            )
        ''')
```

### 实时通信机制

```python
from flask import Flask, jsonify, render_template
from flask_cors import CORS
import threading
import time

app = Flask(__name__)
CORS(app)

# 全局变量存储实时数据
realtime_stats = {
    'total_packets': 0,
    'protocol_counts': {},
    'recent_packets': []
}

def background_data_collector():
    """后台数据收集线程"""
    while True:
        # 收集最新数据包统计
        stats = analyzer.get_latest_stats()
        realtime_stats.update(stats)
        time.sleep(1)

# 启动后台线程
collector_thread = threading.Thread(target=background_data_collector)
collector_thread.daemon = True
collector_thread.start()

@app.route('/api/get_stats')
def get_stats():
    return jsonify(realtime_stats)
```

## 性能优化

### 1. 内存管理
- **数据包缓存**: 限制内存中缓存的数据包数量
- **数据库优化**: 使用索引和分页查询
- **垃圾回收**: 定期清理过期数据

### 2. 并发处理
- **异步捕获**: 使用独立线程进行数据包捕获
- **线程池**: 处理多个并发请求
- **锁机制**: 确保数据一致性

### 3. 网络优化
- **压缩传输**: 对大量数据进行压缩
- **增量更新**: 只传输变化的数据
- **缓存策略**: 缓存常用查询结果

## 安全考虑

### 1. 数据安全
- **数据加密**: 敏感数据加密存储
- **访问控制**: API访问权限验证
- **审计日志**: 记录重要操作

### 2. 网络安全
- **输入验证**: 严格验证用户输入
- **SQL注入防护**: 使用参数化查询
- **XSS防护**: 过滤输出内容

## 部署指南

### 环境要求
```bash
# Python依赖
pip install -r requirements.txt

# 系统依赖 (Linux)
sudo apt-get install tcpdump libpcap-dev
```

### 启动服务
```bash
# 开发模式
python web_interface.py

# 生产部署
nohup python web_interface.py &
```

### 配置选项
```python
# config.py
DEBUG = False
HOST = '0.0.0.0'
PORT = 5000
DATABASE_URL = 'sqlite:///packet_capture.db'
MAX_PACKETS = 10000
CAPTURE_TIMEOUT = 300
```

## 扩展功能

### 1. 机器学习集成
- **异常检测**: 识别异常网络流量
- **协议识别**: 自动识别未知协议
- **流量预测**: 预测网络流量趋势

### 2. 高级可视化
- **3D网络拓扑**: 三维网络结构图
- **地理定位**: IP地址地理位置显示
- **时间序列分析**: 长期趋势分析图表

### 3. 企业功能
- **用户管理**: 多用户权限系统
- **分布式部署**: 多节点数据收集
- **报表系统**: 自动生成分析报告

## 故障排除

### 常见问题

1. **权限问题**: 需要管理员权限运行
2. **端口冲突**: 确保5000端口可用
3. **依赖缺失**: 安装所有必需的依赖包

### 调试工具
```python
# 启用调试模式
DEBUG = True

# 日志配置
import logging
logging.basicConfig(level=logging.DEBUG)
```


