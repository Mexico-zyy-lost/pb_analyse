# 粉末/液体分装系统 — 主控板流程状态机与联锁表

**版本**：V0.2  
**适用对象**：主控板固件 / 上位机流程编排  
**能力范围**：固体分装、液体分装、称重闭环、开盖旋转扫码、震荡混匀

---

## 1. 板台功能模块

| 模块 | 接口 | 功能 |
|------|------|------|
| XY 台 | CAN2 | 工位间移动 |
| Z 轴组（×3） | CAN2 | 分粉器升降、开盖模块升降、移液枪升降 |
| 开盖夹爪 | RS485-1 | 抓瓶、开关盖、空中旋转扫码 |
| 分粉器 | CAN1 | 固体取粉、吐粉 |
| 移液枪 | CAN2 | 吸液、吐液、吹出 |
| 称重模块 | RS232 | 固体/液体定量闭环 |
| 震荡电机 | RS485-2 继电器 | 样本混匀 |
| 四路指示灯 | RS485-2 继电器 | 运行/待机/报警/完成 |
| 摄像头 | 上位机 UDP | 夹爪旋转到位后触发扫码 |

---

## 2. 轴与机构映射

| axis_id | 名称 | 机构 |
|---------|------|------|
| 0 | X | 样本架横向 |
| 1 | Y | 样本架纵向 |
| 2 | Z1 | 分粉器升降 |
| 3 | Z2 | 开盖模块升降 |
| 4 | Z3 | 移液枪/辅助升降 |
| — | LID1/LID2 | 开盖夹爪（RS485 一体化控制） |

---

## 3. 工位定义

| point_id | 名称 | 用途 |
|----------|------|------|
| SRC | 源瓶位 | 固体粉源瓶 / 液体试剂瓶 |
| DST | 目标样本瓶位 | 分装目标 |
| WEIGH | 称重工位 | 固体定量闭环 |
| PIPETTE | 移液工位 | 枪头、吸液准备位 |
| IDLE | 待机位 | 安全停放 |

每个 point 包含：`x_um, y_um, z1_um, z2_um, z3_um` 及分层高度：`safe / work / fine`

---

## 4. 系统状态机

```
                    ┌─────────┐
         上电 ─────>│  POWER  │
                    └────┬────┘
                         │ 自检 OK
                         ▼
                    ┌─────────┐
              ┌────│  IDLE   │<──────────────┐
              │    └────┬────┘               │
              │         │ SYS_HOME           │ BATCH_DONE
              │         ▼                    │
              │    ┌─────────┐               │
              │    │ HOMING  │───────────────┤
              │    └────┬────┘               │
              │         │ HOME_DONE          │
              │         ▼                    │
              │    ┌─────────┐    PAUSE      ┌─────────┐
              │    │ RUNNING │──────────────>│ PAUSED  │
              │    └────┬────┘<──────────────│         │
              │         │                   └─────────┘
              │    严重故障/急停
              │         ▼
              │    ┌─────────┐
              └───>│ FAULT   │
                   └─────────┘
                         │ SYS_RESET + 条件满足
                         └────────> IDLE
```

### 4.1 状态说明

| 状态 | 说明 | 允许指令 |
|------|------|----------|
| IDLE | 空闲，已回零 | HOME, TASK_START, DEPLOY, MAINT |
| HOMING | 回零中 | 仅 ESTOP |
| RUNNING | 执行任务 | PAUSE, ABORT, SKIP |
| PAUSED | 暂停在安全点 | RESUME, ABORT |
| FAULT | 故障 | RESET, ABORT |
| MAINT | 维护 | JOG, SAVE_POINT |

### 4.2 指示灯自动控制

| 系统状态 | 指示灯 |
|----------|--------|
| IDLE | 黄灯亮 |
| RUNNING | 绿灯亮 |
| FAULT | 红灯亮 |
| BATCH_DONE | 蓝灯闪 3 次 |

---

## 5. 配方类型

| recipe_type | 介质 | 主要执行器 |
|-------------|------|------------|
| SOLID_DISPENSE | 固体 | 开盖夹爪 + 分粉器 |
| SOLID_WEIGH_CLOSE | 固体闭环 | 分粉器 + 称重台 |
| LIQUID_DISPENSE | 液体 | 移液枪（开盖可选） |
| CUSTOM | 混合 | 步骤表自由组合 |

---

## 6. 步骤类型定义

| step_type | 值 | 主控板内部动作 |
|-----------|-----|----------------|
| INIT | 0x01 | 绑定 task_id、src/dst、target |
| MOVE_TO | 0x02 | XY 移动到 point |
| Z_DOWN | 0x03 | 指定 Z 下降至工作高度 |
| Z_UP | 0x04 | 所有 Z 升至安全高度 |
| GRIPPER_GRAB | 0x05 | RS485 夹紧瓶身 |
| GRIPPER_RELEASE | 0x06 | RS485 松开 |
| GRIPPER_ROTATE | 0x07 | RS485 旋转到扫码角 |
| SCAN_BARCODE | 0x08 | 通知上位机扫码并等待结果 |
| OPEN_LID | 0x09 | RS485 开盖 |
| CLOSE_LID | 0x0A | RS485 关盖 |
| DISPENSE_TAKE | 0x0B | CAN1 取粉 |
| DISPENSE_PUT | 0x0C | CAN1 吐粉 |
| WEIGH_LOOP | 0x0D | RS232 称重闭环 |
| PIPETTE_ASPIRATE | 0x0E | CAN2 吸液 |
| PIPETTE_DISPENSE | 0x0F | CAN2 吐液 |
| PIPETTE_BLOWOUT | 0x10 | CAN2 吹出残留 |
| SHAKER | 0x11 | 继电器震荡定时 |
| WAIT | 0x12 | 延时或等 IO |
| END | 0xFF | 本瓶结束 |

---

## 7. 固体分装标准流程 SOLID_STD_V1

| 步 | step_type | 设备 | 说明 |
|----|-----------|------|------|
| 0 | INIT | — | 加载任务、绿灯亮 |
| 1 | MOVE_TO | XY+Z2 | 到源瓶位 |
| 2 | GRIPPER_GRAB | RS485 | 抓取瓶身 |
| 3 | Z_UP | Z2 | 抬至空中扫码高度 |
| 4 | GRIPPER_ROTATE | RS485 | 旋转到扫码角 |
| 5 | SCAN_BARCODE | UDP | 扫码校验 |
| 6 | OPEN_LID | RS485 | 开盖 |
| 7 | MOVE_TO | XY+Z1 | 分粉器对准（或 WEIGH 位） |
| 8 | Z_DOWN | Z1 | 分粉器下降 |
| 9 | DISPENSE_TAKE | CAN1 | 取粉 |
| 10 | MOVE_TO | XY | 到目标样本瓶或 WEIGH |
| 11a | WEIGH_LOOP | RS232+CAN1 | 闭环：吐粉→称重→直到达标 |
| 11b | DISPENSE_PUT | CAN1 | 开环：直接吐粉 |
| 12 | Z_UP | Z1/Z2 | 抬升 |
| 13 | CLOSE_LID | RS485 | 关盖（可选） |
| 14 | GRIPPER_RELEASE | RS485 | 放瓶 |
| 15 | SHAKER | 继电器 | 震荡混匀（可选） |
| 16 | END | — | 蓝灯闪、BOTTLE_DONE |

---

## 8. 液体分装标准流程 LIQUID_STD_V1

| 步 | step_type | 设备 | 说明 |
|----|-----------|------|------|
| 0 | INIT | — | 加载任务 |
| 1 | MOVE_TO | XY+Z3 | 到源试剂瓶 |
| 2 | GRIPPER_GRAB | RS485 | 抓瓶（若需开盖） |
| 3 | GRIPPER_ROTATE + SCAN | RS485+UDP | 扫码（可选） |
| 4 | OPEN_LID | RS485 | 开盖 |
| 5 | Z_DOWN | Z3 | 移液枪下降至吸液高度 |
| 6 | PIPETTE_ASPIRATE | CAN2 | 吸液 target_ul |
| 7 | Z_UP | Z3 | 抬起 |
| 8 | MOVE_TO | XY+Z3 | 到目标样本瓶 |
| 9 | Z_DOWN | Z3 | 下降 |
| 10 | PIPETTE_DISPENSE | CAN2 | 吐液 |
| 11 | PIPETTE_BLOWOUT | CAN2 | 吹出残留 |
| 12 | Z_UP | Z3 | 抬起 |
| 13 | CLOSE_LID + RELEASE | RS485 | 关盖放瓶（可选） |
| 14 | SHAKER | 继电器 | 震荡（可选） |
| 15 | END | — | 完成 |

---

## 9. 关键子流程

### 9.1 旋转扫码（开盖前）

```
前置：夹爪已抓瓶，Z2 在空中安全高度

1. CMD_ROTATE(scan_angle) → 到位
2. 上报 RPT_EVENT(SCAN_READY)
3. 发 SCAN_REQUEST 给上位机
4. 等待 SCAN_RESULT（超时 → ALM_SCAN_FAIL）
5. 比对 task.barcode：
   - 匹配 → 继续 OPEN_LID
   - 不匹配 → ALM_BARCODE_MISMATCH
```

### 9.2 OPEN_LID（开盖）

```
前置：已扫码通过（若启用），Z2 在开盖高度

1. CMD_OPEN_LID
2. 监测开盖完成信号
3. 置位 lid_opened = true
4. 上报 STEP_DONE

失败：retry ≤ lid.retry_max → 重试；否则 ALM_OPEN_LID_FAIL
```

### 9.3 DISPENSE_TAKE / PUT（固体）

**取粉前置联锁**：`lid_opened == true`，Z1 在取粉高度，XY 在源瓶位，分粉器 READY

**吐粉前置联锁**：`take_done == true`，Z1 在吐粉高度，XY 在样本瓶位

### 9.4 WEIGH_LOOP（称重闭环）

```
1. TARE（首次进入）
2. loop:
     分粉器微量吐粉
     wait stable_time_ms
     if |weight - target| <= tolerance → DONE
     if retry > max → ALM_WEIGH_OVERRANGE
3. 上报 WEIGH_IN_TOLERANCE
```

### 9.5 PIPETTE 液体操作

**吸液前置**：`lid_opened == true`（若需开盖），Z3 在吸液高度，移液枪 READY

**吐液前置**：`aspirate_done == true`，Z3 在吐液高度

### 9.6 Z 安全策略（全局）

| 规则 | 说明 |
|------|------|
| Z-SAFE-01 | XY 移动前，Z1/Z2/Z3 均须 >= Z_SAFE_HEIGHT |
| Z-SAFE-02 | Z 下降前，XY 须到位（±容差） |
| Z-SAFE-03 | 急停后禁止自动下降，须 RESET + HOME |

---

## 10. 联锁表

| ID | 联锁条件 | 禁止动作 | 报警 |
|----|----------|----------|------|
| IL-01 | 未回零 | TASK_START | ALM_NOT_HOMED |
| IL-02 | 急停有效 | 所有运动 | ALM_ESTOP |
| IL-03 | 门盖未关 | XY/Z 运动 | ALM_INTERLOCK |
| IL-04 | lid_opened==false | DISPENSE_TAKE / PIPETTE_ASPIRATE | ALM_INTERLOCK |
| IL-05 | take_done==false | DISPENSE_PUT | ALM_INTERLOCK |
| IL-06 | aspirate_done==false | PIPETTE_DISPENSE | ALM_INTERLOCK |
| IL-07 | 分粉器 FAULT | TAKE/PUT | 继承故障 |
| IL-08 | 移液枪 FAULT | 移液步骤 | ALM_PIPETTE_FAIL |
| IL-09 | 夹爪未抓瓶 | ROTATE / OPEN_LID | ALM_GRIPPER_FAIL |
| IL-10 | 扫码失败且强制校验 | OPEN_LID | ALM_BARCODE_MISMATCH |
| IL-11 | 称重不稳定 | 闭环判定 | ALM_WEIGH_UNSTABLE |
| IL-12 | Z 不在安全高度 | XY 移动 | 内部阻塞 |
| IL-13 | CAN2 总线忙 | 并发多轴命令 | 内部串行队列 |
| IL-14 | 运行中 | DEPLOY_* | BUSY |
| IL-15 | 心跳丢失 3s | 新任务 | PAUSE + ALM_COMM_TIMEOUT |

---

## 11. 瓶任务状态机

```
BOTTLE_IDLE
    │ TASK_LOAD + START
    ▼
BOTTLE_INIT
    │ step 0 done
    ▼
BOTTLE_RUNNING ──步骤循环──> step 1..N
    │
    ├─ 成功 ──> BOTTLE_DONE ──> 下一瓶 / BATCH_DONE
    │
    └─ 失败 ──> BOTTLE_FAULT
                    │
                    ├─ 自动重试（未超限）──> BOTTLE_RUNNING
                    ├─ 上位机 SKIP ────────> 下一瓶
                    └─ 上位机 ABORT ───────> 任务结束
```

---

## 12. 暂停 / 恢复策略

| 项目 | 规则 |
|------|------|
| 暂停触发 | TASK_PAUSE 或门盖打开 |
| 暂停点 | 当前步骤完成后执行，不中断开盖/旋转中间态 |
| 暂停位置 | 优先 Z_UP 到安全高度后保持 |
| 恢复 | TASK_RESUME，从下一 step 或当前 step 重试 |
| 禁止暂停 | ESTOP、LIMIT、伺服故障 |

---

## 13. 重试策略

| 步骤 | 默认重试 | 超限后 |
|------|----------|--------|
| OPEN_LID | 2 | ALM_OPEN_LID_FAIL |
| DISPENSE_TAKE | 1 | ALM_TAKE_TIMEOUT |
| DISPENSE_PUT | 1 | ALM_DISP_TIMEOUT |
| PIPETTE_ASPIRATE | 1 | ALM_PIPETTE_FAIL |
| SCAN_BARCODE | 1 | ALM_SCAN_FAIL |
| MOVE_TO | 0 | ALM_AXIS_FAULT |

---

## 14. 设备调度原则

```
同一时刻：
  - CAN2：运动命令串行队列；移液与 XY 移动分时执行
  - CAN1：仅分粉器，独立
  - RS485-1：夹爪命令串行，旋转/开盖不可打断
  - RS232：称重轮询 50~100ms
  - RS485-2：继电器可异步，震荡计时由主控板维护
  - UDP：非实时，接收指令与上报

优先级：急停 > 限位 > 流程步骤 > 心跳 > 周期上报
```

---

## 15. 上位机与主控板职责

| 功能 | 上位机 | 主控板 |
|------|:------:|:------:|
| 固体/液体配方编排 | ✓ | 执行 |
| 任务队列 | ✓ | 执行 |
| UDP 通信 | ✓ | ✓ |
| XY/Z 运动 | | ✓ CAN2 |
| 移液枪 | | ✓ CAN2 |
| 分粉器 | | ✓ CAN1 |
| 开盖夹爪/旋转 | | ✓ RS485 |
| 称重读数/闭环 | 显示 | ✓ RS232 |
| 摄像头扫码 | ✓ | 触发/校验 |
| 指示灯/震荡 | 可手动 | ✓ 自动 |
| 急停/联锁 | 软停 | ✓ 硬件 |

---

## 16. 配置表示例

### 16.1 固体称重分装 + 扫码

```json
{
  "recipe_id": 2001,
  "name": "固体称重分装+扫码",
  "medium": "solid",
  "steps": [
    { "step": "INIT" },
    { "step": "MOVE_TO", "point": "SRC" },
    { "step": "GRIPPER_GRAB" },
    { "step": "Z_UP", "axis": "Z2", "height": "scan_safe" },
    { "step": "GRIPPER_ROTATE", "angle": 90 },
    { "step": "SCAN_BARCODE", "required": true },
    { "step": "OPEN_LID" },
    { "step": "MOVE_TO", "point": "WEIGH" },
    { "step": "DISPENSE_TAKE" },
    { "step": "WEIGH_LOOP", "target_mg": 50, "tolerance_mg": 2 },
    { "step": "MOVE_TO", "point": "DST" },
    { "step": "DISPENSE_PUT" },
    { "step": "GRIPPER_RELEASE" },
    { "step": "SHAKER", "duration_s": 30 },
    { "step": "END" }
  ]
}
```

### 16.2 液体移液分装

```json
{
  "recipe_id": 3001,
  "name": "液体移液分装",
  "medium": "liquid",
  "steps": [
    { "step": "INIT" },
    { "step": "MOVE_TO", "point": "SRC" },
    { "step": "OPEN_LID" },
    { "step": "Z_DOWN", "axis": "Z3", "height": "aspirate_work" },
    { "step": "PIPETTE_ASPIRATE", "target_ul": 100 },
    { "step": "Z_UP", "axis": "Z3" },
    { "step": "MOVE_TO", "point": "DST" },
    { "step": "Z_DOWN", "axis": "Z3", "height": "dispense_work" },
    { "step": "PIPETTE_DISPENSE" },
    { "step": "PIPETTE_BLOWOUT" },
    { "step": "Z_UP", "axis": "Z3" },
    { "step": "END" }
  ]
}
```

---

## 17. 验收测试用例

| 编号 | 场景 | 预期 |
|------|------|------|
| TC-01 | 固体开环单瓶 | 全流程 OK |
| TC-02 | 固体称重闭环 | 重量在允差内 |
| TC-03 | 液体移液 100μL | 精度满足规格 |
| TC-04 | 夹爪空中旋转扫码 | SCAN_READY → 条码匹配 |
| TC-05 | 条码不匹配 | ALM_BARCODE_MISMATCH，不继续开盖 |
| TC-06 | 震荡 30s | 继电器 ON/OFF，SHAKER_DONE |
| TC-07 | 四路灯状态 | IDLE/RUN/FAULT/DONE 正确 |
| TC-08 | CAN2 移液+XY 交替 | 无总线冲突 |
| TC-09 | UDP 断线 3s | PAUSE，恢复后可 RESUME |
| TC-10 | 固体+液体任务交替 | 队列顺序执行，介质切换正确 |
| TC-11 | 开盖失败 | 重试 2 次后 FAULT，可 SKIP |
| TC-12 | 未开盖取粉 | IL-04 联锁，拒绝 TAKE |
| TC-13 | 急停 | 立即停，RESET+HOME 后可再跑 |
