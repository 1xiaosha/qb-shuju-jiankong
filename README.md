# qb-shuju-jiankong

监控群晖 Docker 中多个 qBittorrent 实例的每日上传下载总量，并且可以查看30 天历史数据，同时要求轻量化、支持大量种子（5 万），以下是一个稳定、轻量、易部署的方案
用于 qBittorrent 的数据监控方案，使用自定义脚本将每日统计写入轻量数据库，再通过 Web 页面展示
## 使用方法

目录结构
/volume1/docker/qb-shuju-jiankong/
├── data.db                      # SQLite数据库（首次自动创建）
├── fetch_traffic.sh             # 每日收集脚本
├── export_to_json.py            # 导出 JSON 脚本
├── docker-compose.yml           # Nginx 静态网页服务
└── public/
    ├── index.html               # 图表网页
    └── traffic.json             # 每天导出的数据文件


1. fetch_traffic.sh（收集多个 qB 实例每日上传下载数据）
#!/bin/bash

# 定义 qBittorrent 实例列表（名称=地址）
declare -A QBS
QBS["qb-movie"]="http://ip:端口"
QBS["qb-vt"]="http://ip:端口"

# 登录凭据
USERNAME="你的用户名"
PASSWORD="你的密码"

# 数据库路径
DB="/volume1/docker/qb-shuju-jiankong/data.db" #改为你的实际文件地址
TODAY=$(date +%Y-%m-%d)

# 初始化数据库（如未创建）
if [ ! -f "$DB" ]; then
  echo "数据库不存在，创建中..."
  sqlite3 "$DB" "CREATE TABLE traffic (
    date TEXT,
    client TEXT,
    upload REAL,
    download REAL
  );"
fi

# 遍历每个 qB 实例收集数据
for name in "${!QBS[@]}"; do
  URL="${QBS[$name]}"
  echo "正在处理 $name ($URL)"

  # 登录获取 SID
  COOKIE=$(curl -c - -s -X POST -d "username=$USERNAME&password=$PASSWORD" "$URL/api/v2/auth/login" | grep SID | awk '{print $7}')
  
  if [ -z "$COOKIE" ]; then
    echo "⚠️ 登录失败：$name"
    continue
  fi

  # 获取传输信息
  JSON=$(curl -s --cookie "SID=$COOKIE" "$URL/api/v2/transfer/info")
  UP=$(echo "$JSON" | jq .up_info_data)
  DOWN=$(echo "$JSON" | jq .dl_info_data)

  # 处理 null 情况
  UP=${UP:-0}
  DOWN=${DOWN:-0}

  # 字节转 GB（保留2位小数）
  UP_GB=$(awk "BEGIN {printf \"%.2f\", $UP/1024/1024/1024}")
  DOWN_GB=$(awk "BEGIN {printf \"%.2f\", $DOWN/1024/1024/1024}")

  echo "✅ $name 上传: $UP_GB GB，下载: $DOWN_GB GB"

  # 写入数据库
  sqlite3 "$DB" "INSERT INTO traffic (date, client, upload, download) VALUES ('$TODAY', '$name', $UP_GB, $DOWN_GB);"
done
   
2. export_to_json.py（导出近30天数据到 public/traffic.json）
import sqlite3, json
from collections import defaultdict

DB = '/volume1/docker/qb-shuju-jiankong/data.db' #改为你的实际文件地址
OUTPUT = '/volume1/docker/qb-shuju-jiankong/public/traffic.json' #改为你的实际文件地址

conn = sqlite3.connect(DB)
cursor = conn.cursor()

cursor.execute("""
SELECT date, client, upload, download FROM traffic
WHERE date >= date('now', '-30 day')
ORDER BY date ASC;
""")

rows = cursor.fetchall()
result = defaultdict(lambda: defaultdict(dict))

for date, client, up, down in rows:
    result[client][date] = {'upload': up, 'download': down}

with open(OUTPUT, 'w') as f:
    json.dump(result, f, indent=2)

conn.close()


3. public/index.html（前端图表页面）
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>qB 数据监控 - 上传下载流量</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
  <h2>qB 每日上传 / 下载流量图表（近30天）</h2>
  <canvas id="chart" width="1000" height="500"></canvas>

  <script>
    fetch("traffic.json")
      .then(res => res.json())
      .then(data => {
        const labels = Array.from(new Set(Object.values(data).flatMap(x => Object.keys(x)))).sort()
        const datasets = []

        Object.entries(data).forEach(([client, dates], i) => {
          const up = labels.map(date => dates[date]?.upload || 0)
          const down = labels.map(date => dates[date]?.download || 0)

          datasets.push({
            label: `${client} 上传`,
            data: up,
            borderColor: `hsl(${i * 50}, 70%, 50%)`,
            fill: false
          })
          datasets.push({
            label: `${client} 下载`,
            data: down,
            borderColor: `hsl(${i * 50 + 25}, 70%, 50%)`,
            borderDash: [5, 5],
            fill: false
          })
        })

        new Chart(document.getElementById("chart"), {
          type: 'line',
          data: { labels, datasets },
          options: {
            responsive: true,
            scales: {
              y: { beginAtZero: true, title: { display: true, text: "GB" } },
              x: { title: { display: true, text: "日期" } }
            }
          }
        })
      })
  </script>
</body>
</html>

   
4. docker-compose.yml（Nginx 显示图表网页）
version: '3'
services:
  qb-shuju-jiankong-web:
    image: nginx:alpine
    container_name: qb-shuju-jiankong-web
    ports:
      - "8181:80"
    volumes:
      - /volume1/docker/qb-shuju-jiankong/public:/usr/share/nginx/html:ro
    restart: unless-stopped

浏览器访问：http://IP:8181

使用群晖 DSM 自带的“计划任务”图形界面添加
打开群晖 DSM 管理后台
进入：“控制面板” > “任务计划”
点击 “创建” > “计划的任务” > “用户定义的脚本”
配置如下：
第一个任务：每日收集流量
| 字段       | 设置内容                                                           |
| -------- | -------------------------------------------------------------- |
| **任务名称** | `qb收集流量`（可自定义）                                                 |
| **用户**   | `root`                                                         |
| **计划**   | 每天 23:59 执行（或你希望的任意时间）                                         |
| **任务设置** | `/bin/bash /volume1/docker/qb-shuju-jiankong/fetch_traffic.sh` |

第二个任务：每日导出 JSON
| 字段       | 设置内容                                                                   |
| -------- | ---------------------------------------------------------------------- |
| **任务名称** | `qb导出json`（可自定义）                                                       |
| **用户**   | `root`                                                                 |
| **计划**   | 每天 00:00 执行                                                            |
| **任务设置** | `/usr/bin/python3 /volume1/docker/qb-shuju-jiankong/export_to_json.py` |



