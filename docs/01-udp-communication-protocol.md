# 粉末/液体分装系统 — 上下位机 UDP 通信协议草案

**版本**：V0.2  
**适用对象**：上位机（PC/工控机） ↔ 主控板  
**传输方式**：UDP  
**角色**：主控板 = UDP Server，上位机 = Client  
**默认端口**：主控板 `5000`，上位机本地 `5001`（可配置）  
**字节序**：小端（Little-Endian）  
**校验**：CRC16（Modbus 多项式），覆盖 Header + Payload

---

## 1. 系统通信拓扑

```
                         ┌─────────────────┐
                         │     上位机       │
                         │   (UDP Client)   │
                         └────────┬────────┘
                                  │ UDP (以太网)
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                          主控板 (MCU/SoC)                          │
│  流程引擎 │ 联锁 │ 任务队列 │ 设备调度 │ 状态聚合 │ 日志              │
└──┬──────┬──────┬──────┬──────┬──────┬──────────────────────────┘
   │      │      │      │      │      │
   │CAN1  │CAN2  │CAN2  │RS485 │RS232 │RS485(继电器)
   │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼
 分粉器  X/Y/Z  移液枪  开盖    称重   震荡电机
                  轴           夹爪   四路指示灯
```

### 1.1 总线职责划分

| 接口 | 连接对象 | 角色 | 说明 |
|------|----------|------|------|
| **UDP** | 上位机 | 主控板为 Server | 任务下发、状态上报、参数部署 |
| **CAN1** | 分粉器 | 主控板为主 | 固体取粉/吐粉 |
| **CAN2** | X/Y/Z 轴、移液枪 | 主控板为主 | 共总线，靠 CAN ID 区分节点 |
| **RS485-1** | 开盖夹爪 | 主控板为主 | 夹取、旋转、开关盖 |
| **RS485-2** | 继电器板 | 主控板为主 | 震荡电机 + 4 路指示灯 |
| **RS232** | 称重模块 | 主控板为主 | 净重读取、去皮、稳定判定 |

### 1.2 CAN2 节点规划（建议）

| CAN ID | 节点 | 类型 |
|--------|------|------|
| 0x100 | X 轴驱动器 | 运动 |
| 0x101 | Y 轴驱动器 | 运动 |
| 0x102 | Z1 轴（分粉器/通用 Z） | 运动 |
| 0x103 | Z2 轴（开盖模块 Z） | 运动 |
| 0x104 | Z3 轴（移液枪 Z / 辅助） | 运动 |
| 0x200 | 移液枪 | 液体执行器 |

---

## 2. UDP 报文结构

```
┌──────┬─────┬─────┬─────┬──────┬──────┬──────┬─────────┬───────┐
│ SOF  │ VER │ SRC │ DST │ SEQ  │ CMD  │ LEN  │  DATA   │ CRC16 │
│ 0xAA │ 0x02│ 1B  │ 1B  │ 2B   │ 2B   │ 2B   │  N      │ 2B    │
└──────┴─────┴─────┴─────┴──────┴──────┴──────┴─────────┴───────┘
```

| 字段 | 长度 | 说明 |
|------|------|------|
| SOF | 1 | 固定 `0xAA` |
| VER | 1 | 协议版本，当前 `0x02` |
| SRC | 1 | 源地址：`0x01`=上位机，`0x10`=主控板 |
| DST | 1 | 目的地址 |
| SEQ | 2 | 帧序号（uint16），应答原样带回 |
| CMD | 2 | 命令字（uint16），应答 CMD \| 0x8000 |
| LEN | 2 | DATA 长度（uint16） |
| DATA | N | 载荷，最大 1024 字节 |
| CRC16 | 2 | 从 VER 到 DATA 末尾 |

### 2.1 可靠性策略

| 机制 | 规则 |
|------|------|
| 请求-应答 | 写操作必须 ACK，超时 300ms，重试 3 次 |
| 周期上报 | 主控板每 200ms 发 `RPT_STATUS` |
| 心跳 | 上位机每 1s 发 `SYS_HEARTBEAT`；3s 无心跳 → `COMM_TIMEOUT` |
| 事件 | 步骤完成、报警即时 `RPT_EVENT` |
| 大包分包 | DATA > 1200 字节时按 FRAG 分包（可选） |

---

## 3. 命令字定义（上位机 ↔ 主控板）

### 3.1 系统类（0x00xx）

| CMD | 名称 | 方向 | 说明 |
|-----|------|------|------|
| 0x0001 | SYS_HEARTBEAT | 上→主 | 心跳 |
| 0x0002 | SYS_GET_INFO | 上→主 | 查询固件版本、能力 |
| 0x0003 | SYS_SET_MODE | 上→主 | 设置运行模式 |
| 0x0004 | SYS_RESET | 上→主 | 故障复位 |
| 0x0005 | SYS_HOME | 上→主 | 整机回零 |
| 0x0006 | SYS_ESTOP | 上→主 | 软急停 |

**SYS_SET_MODE DATA**

```
uint8 mode
  0 = AUTO     自动
  1 = MAINT    手动/维护
  2 = SIM      仿真
```

**SYS_GET_INFO 应答 DATA**

```
char fw_ver[16]
char hw_ver[16]
uint8 axis_count
uint8 recipe_capacity
uint32 feature_flags
```

**feature_flags 位定义**

| bit | 能力 |
|-----|------|
| 0 | 固体分装 |
| 1 | 液体分装 |
| 2 | 称重闭环 |
| 3 | 开盖旋转扫码 |
| 4 | 震荡混匀 |

---

### 3.2 部署类（0x01xx）

| CMD | 名称 | 方向 | 说明 |
|-----|------|------|------|
| 0x0101 | DEPLOY_RECIPE | 上→主 | 下发配方 |
| 0x0102 | DEPLOY_POSITIONS | 上→主 | 下发点位表 |
| 0x0103 | DEPLOY_PARAMS | 上→主 | 下发运动/开盖/分粉/移液/称重参数 |
| 0x0104 | DEPLOY_VERIFY | 上→主 | 回读校验（CRC） |
| 0x0105 | RECIPE_LIST | 上→主 | 查询本地配方列表 |

**DEPLOY_RECIPE DATA**

```
uint32 recipe_id
uint16 recipe_ver
uint8  step_count
StepDef steps[step_count]

--- StepDef ---
uint8  step_type
uint8  medium_type     // 0=通用, 1=固体, 2=液体
uint8  point_id
uint16 param_idx
uint32 param_value
```

**DEPLOY_POSITIONS DATA**

```
uint8 point_count
PointDef points[point_count]

--- PointDef ---
uint8  point_id
int32  x_um
int32  y_um
int32  z1_um
int32  z2_um
int32  z3_um
uint16 flags
```

**DEPLOY_PARAMS DATA**

```
MotionParams motion
LidParams    lid
DispParams   disp
PipetteParams pipette
WeighParams  weigh
GripperParams gripper

--- MotionParams ---
uint16 xy_speed
uint16 z_speed
uint16 accel
int32  soft_limit_min[5]
int32  soft_limit_max[5]

--- LidParams ---
uint16 clamp_speed
uint16 rotate_speed
uint16 torque_threshold
uint8  retry_max

--- DispParams ---
uint32 target_mg
uint16 take_timeout_ms
uint16 disp_timeout_ms
uint8  retry_max

--- PipetteParams ---
uint32 target_ul
uint16 aspirate_speed
uint16 dispense_speed
uint16 blowout_ul

--- WeighParams ---
uint32 target_mg
uint32 tolerance_mg
uint32 stable_time_ms
uint8  weigh_mode       // 0=开环, 1=称重闭环

--- GripperParams ---
uint16 clamp_force
uint16 rotate_speed
uint16 scan_angle_deg
uint16 open_lid_torque
```

---

### 3.3 任务类（0x02xx）

| CMD | 名称 | 方向 | 说明 |
|-----|------|------|------|
| 0x0201 | TASK_LOAD | 上→主 | 加载任务队列 |
| 0x0202 | TASK_START | 上→主 | 启动 |
| 0x0203 | TASK_PAUSE | 上→主 | 暂停 |
| 0x0204 | TASK_RESUME | 上→主 | 继续 |
| 0x0205 | TASK_ABORT | 上→主 | 中止 |
| 0x0206 | TASK_SKIP | 上→主 | 跳过当前瓶 |
| 0x0207 | TASK_CLEAR | 上→主 | 清空队列 |

**TASK_LOAD DATA**

```
uint16 task_count
TaskItem tasks[task_count]

--- TaskItem ---
uint32 task_id
uint32 recipe_id
uint16 recipe_ver
uint8  medium          // 1=固体, 2=液体
uint8  src_point_id
uint8  dst_point_id
uint8  weigh_point_id
uint32 target_amount   // mg 或 ul
char   barcode[32]
uint8  flags           // bit0=需扫码, bit1=需称重, bit2=需震荡
```

---

### 3.4 维护类（0x03xx）

| CMD | 名称 | 方向 | 说明 |
|-----|------|------|------|
| 0x0301 | MAINT_JOG | 上→主 | 点动 |
| 0x0302 | MAINT_STOP_JOG | 上→主 | 停止点动 |
| 0x0303 | MAINT_SAVE_POINT | 上→主 | 保存当前坐标到点位 ID |
| 0x0304 | MAINT_RUN_STEP | 上→主 | 单步执行（调试） |

**MAINT_JOG DATA**

```
uint8 axis_id   // 0=X,1=Y,2=Z1,3=Z2,4=Z3
int8  direction
uint16 speed
```

---

### 3.5 扫码与指示类（0x04xx）

| CMD | 名称 | 方向 | 说明 |
|-----|------|------|------|
| 0x0401 | SCAN_REQUEST | 主→上 | 请求上位机摄像头扫码 |
| 0x0402 | SCAN_RESULT | 上→主 | 返回条码/结果 |
| 0x0403 | LAMP_SET | 上→主 | 设置四路指示灯 |
| 0x0404 | SHAKER_CTRL | 上→主 | 震荡电机启停/定时 |

**SCAN_REQUEST DATA**

```
uint32 task_id
uint8  bottle_index
uint8  camera_id
uint16 timeout_ms
```

**SCAN_RESULT DATA**

```
uint8  result          // 0=OK, 1=失败, 2=超时
char   barcode[32]
```

**LAMP_SET DATA**

```
uint8 mask             // bit0~3 对应四路灯
uint8 values
```

---

### 3.6 上报类（0x08xx，主控板→上位机）

| CMD | 名称 | 说明 |
|-----|------|------|
| 0x0801 | RPT_STATUS | 周期状态 |
| 0x0802 | RPT_EVENT | 事件 |
| 0x0803 | RPT_ALARM | 报警 |
| 0x0804 | RPT_LOG | 运行日志（可选） |

**RPT_STATUS DATA（200ms）**

```
uint8  sys_mode
uint8  sys_state
uint8  step_index
uint8  step_type
uint8  medium
uint32 task_id
uint8  bottle_index
uint16 progress_pct
int32  axis_pos[5]         // X,Y,Z1,Z2,Z3
uint8  gripper_state
uint8  dispenser_state
uint8  pipette_state
int32  weight_mg
uint8  weight_stable
uint8  lamp_states
uint8  shaker_on
uint16 active_alarms
```

**RPT_EVENT DATA**

```
uint8  event_type
uint32 task_id
uint8  bottle_index
uint8  step_index
uint32 timestamp_ms
uint8  extra[8]
```

| event_type | 含义 |
|------------|------|
| 0x01 | STEP_STARTED |
| 0x02 | STEP_DONE |
| 0x03 | BOTTLE_DONE |
| 0x04 | BATCH_DONE |
| 0x05 | TASK_PAUSED |
| 0x06 | TASK_ABORTED |
| 0x10 | SCAN_READY |
| 0x11 | WEIGH_STABLE |
| 0x12 | WEIGH_IN_TOLERANCE |
| 0x13 | SHAKER_DONE |

---

## 4. 结果码与报警码

### 4.1 通用结果码（ACK 首字节）

| 码 | 含义 |
|----|------|
| 0x00 | OK |
| 0x01 | BUSY |
| 0x02 | INVALID_PARAM |
| 0x03 | NOT_HOMED |
| 0x04 | FAULT_ACTIVE |
| 0x05 | MODE_ERROR |
| 0x06 | CRC_ERROR |
| 0x07 | STORAGE_FULL |

### 4.2 报警码

| 码 | 名称 | 级别 | 说明 |
|----|------|------|------|
| 0x1001 | ALM_ESTOP | 严重 | 急停触发 |
| 0x1002 | ALM_LIMIT | 严重 | 硬限位 |
| 0x1003 | ALM_COMM_TIMEOUT | 警告 | 上位机心跳丢失 |
| 0x1101 | ALM_OPEN_LID_FAIL | 错误 | 开盖失败 |
| 0x1102 | ALM_TAKE_TIMEOUT | 错误 | 取粉超时 |
| 0x1103 | ALM_DISP_TIMEOUT | 错误 | 吐粉超时 |
| 0x1104 | ALM_AXIS_FAULT | 严重 | 伺服/步进故障 |
| 0x1105 | ALM_SCAN_FAIL | 错误 | 扫码失败 |
| 0x1106 | ALM_WEIGH_UNSTABLE | 错误 | 称重不稳定 |
| 0x1107 | ALM_WEIGH_OVERRANGE | 错误 | 超重/欠重 |
| 0x1108 | ALM_PIPETTE_FAIL | 错误 | 移液枪故障 |
| 0x1109 | ALM_GRIPPER_FAIL | 错误 | 夹爪故障 |
| 0x1110 | ALM_BARCODE_MISMATCH | 错误 | 条码与任务不符 |
| 0x1201 | ALM_NOT_HOMED | 警告 | 未回零 |

---

## 5. 设备底层协议（主控板侧）

### 5.1 CAN1 — 分粉器

| CAN ID | 方向 | 功能 |
|--------|------|------|
| 0x300 | 主→分 | 命令帧 |
| 0x301 | 分→主 | 状态帧 |
| 0x302 | 分→主 | 应答帧 |

| cmd | 说明 |
|-----|------|
| 0x01 | INIT |
| 0x10 | TAKE（data: target_mg uint32） |
| 0x11 | DISPENSE |
| 0x20 | STOP |
| 0x30 | RESET |

| state | 说明 |
|-------|------|
| 0 | IDLE |
| 1 | BUSY |
| 2 | TAKE_DONE |
| 3 | DISPENSE_DONE |
| 4 | FAULT |

### 5.2 CAN2 — X/Y/Z 轴 + 移液枪

运动轴支持：HOME、MOVE_ABS、MOVE_REL、STOP、GET_POS、GET_STATUS。

移液枪（CAN ID 0x200）：

| cmd | 说明 |
|-----|------|
| 0x01 | INIT |
| 0x10 | ASPIRATE（volume_ul, speed） |
| 0x11 | DISPENSE |
| 0x12 | BLOWOUT |
| 0x20 | TIP_EJECT |
| 0x30 | GET_STATUS |

### 5.3 RS485-1 — 开盖夹爪（从站 0x01）

| 命令 | 功能 |
|------|------|
| CMD_GRAB | 夹紧瓶身 |
| CMD_RELEASE | 松开 |
| CMD_OPEN_LID | 开盖 |
| CMD_CLOSE_LID | 关盖 |
| CMD_ROTATE | 旋转到指定角度 |
| CMD_HOME | 回零 |

### 5.4 RS232 — 称重模块

| 操作 | 说明 |
|------|------|
| TARE | 去皮 |
| READ | 读当前重量（mg） |
| IS_STABLE | 稳定标志 |
| ZERO | 清零 |

### 5.5 RS485-2 — 继电器板（从站 0x02）

| 通道 | 功能 |
|------|------|
| CH0 | 震荡电机 |
| CH1 | 指示灯-运行（绿） |
| CH2 | 指示灯-报警（红） |
| CH3 | 指示灯-待机（黄） |
| CH4 | 指示灯-完成（蓝） |

---

## 6. 通信时序示例

### 6.1 上线部署

```
上位机                         主控板
  |── SYS_GET_INFO ────────────>|
  |<── INFO_ACK ────────────────|
  |── DEPLOY_POSITIONS ────────>|
  |<── ACK(OK) ─────────────────|
  |── DEPLOY_RECIPE ───────────>|
  |<── ACK(OK) ─────────────────|
  |── DEPLOY_VERIFY ───────────>|
  |<── CRC_MATCH ───────────────|
  |── TASK_LOAD ───────────────>|
  |<── ACK(OK) ─────────────────|
  |── SYS_HOME ────────────────>|
  |<── EVENT: HOME_DONE ─────────|
  |── TASK_START ──────────────>|
  |<── RPT_STATUS (循环) ───────|
```

### 6.2 旋转扫码

```
主控板                          上位机
  |── RPT_EVENT(SCAN_READY) ───>|
  |── SCAN_REQUEST ────────────>|
  |<── SCAN_RESULT(barcode) ────|
  |  校验 barcode vs task        |
  |── 继续 OPEN_LID 或报警 ──────|
```

---

## 7. 实现约束

1. 写命令必须等 ACK，超时 300ms，重试 3 次
2. 运行中禁止 `DEPLOY_*`，除非 `sys_state == IDLE` 且维护模式
3. `TASK_START` 前置条件：已回零、无严重报警
4. 所有坐标单位统一为微米（μm）
5. 日志字段：`task_id + bottle_index + step_index + alarm_code + timestamp`
6. CAN2 运动与移液命令由主控板串行调度，避免总线冲突
