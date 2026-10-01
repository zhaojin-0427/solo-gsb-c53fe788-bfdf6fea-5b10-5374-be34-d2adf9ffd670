# 建筑无障碍通路核验台

本地 Web 应用，用于核验轮椅使用者在建筑内的无障碍通行路线。基于 Python + FastAPI + SQLite + NetworkX 管理有向路网，浏览器使用原生 JavaScript 呈现楼层图。

## 功能特性

- **路网导入**：导入含楼层坐标、出入口、房间和通道的 JSON 文件；系统拒绝悬空节点和非法跨层连接（跨层仅允许电梯或坡道）。
- **并行通道**：允许起终点相同、ID 不同的多条通道并存。每条通道按各自 ID 独立参与净宽、坡度、楼梯/跨层限制、封闭时段和节点等待判断，后导入的通道不会覆盖先导入的通道。
- **封闭时段登记**：为通道按 ID 登记带时区的封闭时段，支持跨时区；封闭一条通道不影响其并行通道。
- **轮椅路线查询**：按起终点、出发时刻、最小净宽、最大坡度查询路线。楼梯不可走，跨层仅允许电梯或坡道，可在节点等候，但通行全程不得与封闭时段相交。在出发后 24 小时内求最早到达，平局依次按总路程和边 ID 序列择优。
- **楼层图呈现**：逐层标出路线与等待时间；起终点相同的并行通道会错开绘制，不可达时列出阻断原因。
- **核验记录固化**：每次查询固化路网、封闭时段版本及当时的查询结果，后续修改不改变旧结果，旧记录不会被重算。

## 启动方式

### Docker Compose 一键启动（推荐）

```bash
docker compose up --build
```

启动后访问 <http://localhost:8000>。

### 本地开发启动

```bash
pip install -r backend/requirements.txt
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

访问 <http://localhost:8000>。

## 使用流程

1. **导入路网**：在左侧面板点击「选择文件」，选择示例文件 `data/sample_network.json`（或自行编写的路网 JSON），点击「导入路网」。
2. **登记封闭时段**：在「封闭时段登记」中填写通道 ID、起止时间和时区，点击「登记封闭时段」。
3. **查询路线**：填写起点、终点、出发时刻、最小净宽和最大坡度，点击「查询路线」。
4. **查看结果**：右侧「核验结果」面板会逐层列出路线与等待时间；不可达时列出阻断原因。
5. **查看历史记录**：右侧「历史核验记录」列出所有核验记录，点击可查看当时的路网与结果。

## 路网 JSON 格式

```json
{
  "floors": [
    { "id": "f1", "name": "1F", "level": 0 }
  ],
  "nodes": [
    { "id": "n1", "floor_id": "f1", "x": 0, "y": 0, "kind": "entrance", "name": "主入口" }
  ],
  "edges": [
    {
      "id": "e1",
      "from": "n1",
      "to": "n2",
      "type": "corridor",
      "length": 10,
      "seconds": 12,
      "width": 1.8,
      "slope": 0
    }
  ]
}
```

### 字段说明

- **floors**：楼层列表。`id` 唯一标识，`name` 显示名称，`level` 楼层序号。
- **nodes**：节点列表。
  - `kind`：节点类型，可选 `entrance`（出入口）、`exit`（出口）、`room`（房间）、`junction`（节点）、`elevator`（电梯）。
  - `x` / `y`：楼层图坐标。
- **edges**：通道列表。
  - `type`：通道类型，可选 `corridor`（通道）、`stairs`（楼梯）、`elevator`（电梯）、`ramp`（坡道）。
  - `length`：长度（米）。
  - `seconds`：通行秒数。
  - `width`：净宽（米）。
  - `slope`：坡度（度）。

### 校验规则

- 节点 ID、通道 ID、楼层 ID 不可重复。
- 通道的起终点必须存在。
- 不允许悬空节点（未连接任何通道的节点）。
- 跨楼层通道的类型必须是 `elevator` 或 `ramp`；同楼层通道类型不可为 `elevator` 或 `ramp`。
- 长度、通行秒数、净宽不可为负；坡度在 0~90 之间。
- 多条通道可以使用相同的起终点（并行通道），只要通道 ID 不同即可；各自的秒数、净宽、坡度、类型与封闭时段互不影响。

### 示例路网中的并行通道

`data/sample_network.json` 内置两组同起终点的并行通道，用于验证路线不会被后导入的通道覆盖：

| 起终点 | 通道 ID | 通行秒数 | 净宽 | 说明 |
|--------|---------|----------|------|------|
| n3 ↔ n8（1F↔2F 电梯） | `e5` / `e6` | 20s | 1.2m | 慢速窄梯 |
| n3 ↔ n8（1F↔2F 电梯） | `e27` / `e28` | 8s | 1.5m | 快速宽梯 |
| n1 ↔ n2（主入口↔门厅） | `e1` / `e2` | 12s | 1.8m | 宽敞通道 |
| n1 ↔ n2（主入口↔门厅） | `e29` / `e30` | 30s | 1.0m | 窄慢通道 |

## API 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/network/import` | 导入路网 JSON |
| GET | `/api/network` | 获取当前路网 |
| GET | `/api/network/versions` | 列出路网版本 |
| GET | `/api/network/{version}` | 获取指定版本路网 |
| POST | `/api/closures` | 按通道 ID 登记封闭时段 |
| GET | `/api/closures` | 列出封闭时段 |
| DELETE | `/api/closures/{id}` | 删除封闭时段 |
| POST | `/api/route/query` | 查询轮椅路线（返回实际选中通道的步骤或阻断原因，并固化核验记录） |
| GET | `/api/records` | 列出核验记录 |
| GET | `/api/records/{id}` | 获取核验记录详情（含固化结果，不重算） |

### 路线查询示例

以下示例基于 `data/sample_network.json`。先导入路网：

```bash
curl -X POST http://localhost:8000/api/network/import \
  -H 'Content-Type: application/json' \
  --data @data/sample_network.json
```

#### 1. 可达：在并行电梯中选中实际更快的通道

请求：

```bash
curl -X POST http://localhost:8000/api/route/query \
  -H 'Content-Type: application/json' \
  -d '{
    "start": "n1",
    "end": "n11",
    "departure": "2026-10-01T08:00:00+08:00",
    "min_width": 1.2,
    "max_slope": 90
  }'
```

响应（节选）：跨层一步选中的是 8 秒的快速宽梯 `e27`，而不是 20 秒的 `e5`；总到达时间 56 秒，每个步骤都带实际通道 ID、该通道属性和节点等待秒数：

```json
{
  "reachable": true,
  "arrival": "2026-10-01T00:00:56+00:00",
  "total_seconds": 56.0,
  "total_length": 44.0,
  "wait_seconds": 0.0,
  "steps": [
    { "edge_id": "e1",  "from_node": "n1", "to_node": "n2", "type": "corridor", "length": 10.0, "seconds": 12.0, "width": 1.8, "slope": 0.0, "wait_seconds": 0.0, "cross_floor": false, "floor_id": "f1", "floor_name": "1F" },
    { "edge_id": "e3",  "from_node": "n2", "to_node": "n3", "type": "corridor", "length": 10.0, "seconds": 12.0, "width": 1.8, "slope": 0.0, "wait_seconds": 0.0, "cross_floor": false, "floor_id": "f1", "floor_name": "1F" },
    { "edge_id": "e27", "from_node": "n3", "to_node": "n8", "type": "elevator", "length": 4.0,  "seconds": 8.0,  "width": 1.5, "slope": 0.0, "wait_seconds": 0.0, "cross_floor": true,  "floor_id": "f1", "floor_name": "1F" },
    { "edge_id": "e7",  "from_node": "n8", "to_node": "n9", "type": "corridor", "length": 10.0, "seconds": 12.0, "width": 1.8, "slope": 0.0, "wait_seconds": 0.0, "cross_floor": false, "floor_id": "f2", "floor_name": "2F" },
    { "edge_id": "e15", "from_node": "n9", "to_node": "n11", "type": "corridor", "length": 10.0, "seconds": 12.0, "width": 1.5, "slope": 0.0, "wait_seconds": 0.0, "cross_floor": false, "floor_id": "f2", "floor_name": "2F" }
  ],
  "reasons": [],
  "blocking_closures": [],
  "record_id": 1,
  "network_version": 1,
  "closure_version": 0
}
```

把 `min_width` 提高到 `1.3` 时，`e27`（1.5m）仍可通行而 `e5`（1.2m）被宽度限制排除；这正是修复前只保留一条并行电梯时会误报不可达的情形。

#### 2. 封闭与等待：封闭一条并行通道，自动改走另一条

为快速梯登记封闭时段（仅封闭 `e27`，不影响并行的 `e5`）：

```bash
curl -X POST http://localhost:8000/api/closures \
  -H 'Content-Type: application/json' \
  -d '{
    "edge_id": "e27",
    "start": "2026-10-01T08:00:00+08:00",
    "end":   "2026-10-01T08:30:00+08:00",
    "tz": "Asia/Shanghai"
  }'
```

再次发起与示例 1 相同的查询：算法比较「在节点等到 08:30 再乘 `e27`」与「立即乘 `e5`」两个方案后选择后者，跨层步骤变为 `e5`，到达时间 00:01:08 UTC。若等待更短（例如封闭只持续几分钟），响应对应步骤的 `wait_seconds` 会标出在节点的等待时长。

#### 3. 不可达：返回阻断原因

当两条并行电梯和坡道在出发后 24 小时内全部封闭（或净宽要求高于全部通道）时，响应为：

```json
{
  "reachable": false,
  "reasons": ["路径被封闭时段阻断，24 小时内无法到达"],
  "blocking_closures": [
    { "edge_id": "e5",  "from_node": "n3", "to_node": "n8", "start": "2026-10-01T00:00:00+00:00", "end": "2026-10-02T00:30:00+00:00" },
    { "edge_id": "e27", "from_node": "n3", "to_node": "n8", "start": "2026-10-01T00:00:00+00:00", "end": "2026-10-02T00:30:00+00:00" }
  ],
  "record_id": 1,
  "network_version": 1,
  "closure_version": 3
}
```

`blocking_closures` 按通道 ID 列出每条被封闭的并行通道（示例中为自由流最短路径上的两部并行电梯；坡道属于另一条绕行路径，未列入该路径的诊断）。若是净宽/坡度/楼梯等静态限制导致的不可达，`reasons` 会给出「不存在满足净宽/坡度要求的可达路径」等原因。所有查询结果（可达时的 `steps`，或不可达时的 `reasons` 与 `blocking_closures`）都原样固化进核验记录，之后在浏览器或 `/api/records/{id}` 中查看的都是当时结论，路网或封闭时段的后续修改不会重算旧记录。

## 技术栈

- 后端：Python 3.12、FastAPI、SQLite、NetworkX
- 前端：原生 JavaScript、SVG
- 部署：Docker、Docker Compose
