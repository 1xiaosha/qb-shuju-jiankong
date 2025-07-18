# qb-shuju-jiankong

监控群晖 Docker 中多个 qBittorrent 实例的每日上传下载增量总量，并且可以查看30 天历史数据，同时要求轻量化、支持大量种子，以下是一个稳定、轻量、易部署的方案
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
2. export_to_json.py（导出近30天数据到 public/traffic.json）
3. public/index.html（前端图表页面）   
4. docker-compose.yml（Nginx 显示图表网页）


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



