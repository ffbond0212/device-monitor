# 设备数据采集与远程监控系统 · 模块规划与接口契约 v1

> 配套文档：`Device_Monitor_Plan.md`（v2 大纲）。本文规定每个模块的**职责、接口签名、约束、验收标准**——实现由你自己写，这里是契约，不是答案。
> 每个模块四件套：**职责一句话 / 接口签名（C）/ 约束（红线）/ 验收标准（done 的定义）**。

---

## 〇、总则

### 代码分层（目录结构，Git 同步建）

```
Device_Monitor/
├── Core/                    # CubeMX 生成区（只读区，除 USER CODE 块外不动）
├── BSP/                     # 板级驱动：bsp_can / bsp_oled / net_esp（对 HAL 的薄封装）
├── Service/                 # 业务服务：svc_filter / proto_parse / svc_shared /
│                            #   svc_link_mon / svc_ctrl / svc_diag / svc_health /
│                            #   svc_mqtt / app_latency
├── App/                     # 应用层：app_frame_table（帧表/信号表配置）、任务创建入口
└── Bootloader/              # 独立工程（Week4 IAP 最小版）
```

**分层红线**：Service 不许直接调 HAL（必须经过 BSP）；中断里只准调 BSP 的 `*_FromISR` 类函数；CubeMX 生成区除 USER CODE 块外一行不改。

### 命名规范

- 类型 `Xxx_t`，函数 `模块_动词()`（如 `LinkMon_Feed`），宏 `模块_XXX`
- 每个模块一对文件：`svc_xxx.c` / `svc_xxx.h`，头文件里写清模块职责注释（三行以内）
- 所有跨任务共享的结构体，字段访问必须走 Service 提供的 Get/Update，禁止任务间直接摸

### 公共类型（app_frame_table.h，M01 定义，全员引用）

```c
typedef struct {
    uint32_t id;        // 11 位标准帧 ID（本项目不用扩展帧）
    uint8_t  dlc;       // 0~8
    uint8_t  data[8];
} CAN_Frame_t;

typedef struct { uint16_t rpm; int16_t temp; uint16_t volt; uint8_t status; } MotorData_t;
typedef struct { uint32_t dtc; uint8_t alarm; uint32_t uptime; } AlarmData_t;
```

---

## 一、模块总表

| # | 模块 | 层 | 周次 | 优先级 | 状态 |
|---|---|---|---|---|---|
| M01 | app_frame_table 帧表/信号表配置 | App | W1 | 核心 | 待写 |
| M02 | bsp_can CAN 收发驱动 | BSP | W1 | 核心 | 待写 |
| M03 | svc_filter 验收滤波 | Service | W2 | 核心 | 待写 |
| M04 | proto_parse 信号解包 | Service | W2 | 核心 | 待写 |
| M05 | svc_shared 共享数据区 | Service | W2 | 核心 | 待写 |
| M06 | svc_link_mon 掉线检测 | Service | W2 | 核心（v2 新增） | 待写 |
| M07 | svc_ctrl 控制闭环 | Service | W2 | 核心（v2 新增） | 待写 |
| M08 | bsp_oled OLED 驱动 | BSP | W2 | 核心 | 待写 |
| M09 | app_latency 端到端时延测量 | Service | W2 | 核心（简历数字） | 待写 |
| M10 | svc_diag 诊断服务 | Service | W3 | 核心 | 待写 |
| M11 | svc_health 健康监控+看门狗 | Service | W2~W3 | 核心（v2 新增） | 待写 |
| M12 | net_esp ESP8266 透传 | BSP | W4 | 增强 | 待写 |
| M13 | svc_mqtt MQTT 客户端 | Service | W4 | 增强 | 待写 |
| M14 | boot_iap Bootloader 最小版 | Boot | W4 | 核心（v2 升级） | 待写 |
| M15 | dev_sim 模拟设备侧工程（第二块板） | 独立工程 | W2~W3 | 核心 | 待写 |

---

## 二、Week 1 模块

### M01 · app_frame_table —— 帧表/信号表配置

**职责**：集中定义所有 CAN 帧 ID、DLC、方向、信号布局；全项目唯一真源，改协议只改这一个文件。

```c
#define FRAME_MOTOR_ID      0x100u
#define FRAME_ALARM_ID      0x200u
#define FRAME_CTRL_ID       0x300u
#define FRAME_CTRL_ACK_ID   0x301u
#define FRAME_DIAG_REQ_ID   0x500u
#define FRAME_DIAG_RESP_ID  0x501u

#define LINK_TIMEOUT_CYCLES 5u    // 掉线判定：连续 5 个周期未收到
#define CTRL_ACK_TIMEOUT_MS 200u  // 控制指令确认超时
```

**约束**：ID 用宏不用字面量；信号布局的字节偏移也必须宏化（`MOTOR_RPM_OFF 0`），解包模块只准引用这些宏。

**验收**：搜全工程代码，除了本文件没有任何地方出现 `0x100` 这类帧 ID 字面量。

### M02 · bsp_can —— CAN 收发驱动

**职责**：对 HAL_CAN 的薄封装——发送（查邮箱）、接收（中断→队列）、loopback 自测。

```c
void     CAN_Init(void);                              // 初始化 + 启动 + 使能 Rx FIFO0 中断
bool     CAN_SendFrame(const CAN_Frame_t *frame, uint32_t timeout_ms);
void     CAN_RxFifo0Callback(CAN_Frame_t *out);       // 由 HAL 回调里调用，帧拷出到队列
uint32_t CAN_GetErrorCounters(uint32_t *tec, uint32_t *rec);  // 给 health 用
```

**上手路径（板子没到也能写）**：先不接 TJA1050，CubeMX 里把 CAN 模式选 **Loopback**（或代码里 `hcan.Init.Mode = CAN_MODE_LOOPBACK`），自己发自己收——验的就是位时序配没配对、帧格式对不对、中断链路通不通。main 里每 500ms 发一帧固定测试帧，串口打印收到的 ID/DLC/数据，逐字节对上就算 M02 通了。**loopback 不是独立模块，是 M02 的自测手段，也是 W1 总闸门的第一项**。它的价值是"择祸"：总线不通的时候，你分不清是代码错、配置错还是接线错，loopback 一跑，代码/配置的锅先择干净，剩下才是线和 TJA1050 的事。

**约束**：
- 中断回调里**只做帧搬运**（拷结构体 + 投队列），不解包、不打印、不浮点
- 发送前必须查 `HAL_CAN_GetTxMailboxesFreeLevel`，邮箱满时按 `timeout_ms` 等待，超时返回 false——**不许死等**
- 队列用 FreeRTOS 队列（深度 16），中断里用 `xQueueSendFromISR`

**验收**：
- loopback 模式自发自收，串口打印 ID/DLC/数据与发送值逐字节一致
- 连续发 1000 帧无丢失、无死机；讲得出“为什么接收放中断、解包放任务”

---

## 三、Week 2 模块

### M03 · svc_filter —— 验收滤波

**职责**：把“只收 6 个业务 ID”翻译成 32 位标识符 + 屏蔽寄存器配置。

```c
void Filter_Init(void);   // 按 app_frame_table 的 6 个 ID 生成滤波器配置并生效
```

**约束**：标准帧用 32 位屏蔽位模式（Identifier 高 16 位 + Mask 高 16 位）；滤波后总线上其他 ID 的帧不产生接收中断。

**验收**：用另一节点（或分析仪）发一个业务外的 ID，串口/示波器确认**无任何中断反应**；讲得出“总线负载高时滤波怎么降低 MCU 中断频率”。

### M04 · proto_parse —— 信号解包（逻辑层，自己写）

**职责**：把 8 字节原始数据按信号表还原成物理值。**纯函数，无硬件依赖**。

```c
void Proto_ParseMotor(const uint8_t *data, MotorData_t *out);
void Proto_ParseAlarm(const uint8_t *data, AlarmData_t *out);
uint8_t Proto_Checksum(const uint8_t *data, uint8_t len);   // byte7 校验
```

**约束**：跨字节信号按 Intel 小端拼（`(b1<<8)|b0`）；温度是有符号 int16；每个函数入参做长度断言（DLC 不对直接返回不解析）。

**验收**：给一组构造的字节流（如 `64 00 E8 03 20 1F 01 XX`），输出 100rpm/100.0℃/80.0V；**讲得出 Intel 和 Motorola 的区别**——这题面试必问。

### M05 · svc_shared —— 共享数据区

**职责**：共享数据区的唯一守门人，所有读写走互斥量。

```c
void Shared_Update(const MotorData_t *d, const AlarmData_t *a);
void Shared_Get(MotorData_t *d, AlarmData_t *a);
void Shared_GetSnapshot(MotorData_t *d, AlarmData_t *a);   // 一次性拷两份，防撕裂
```

**约束**：内部持有一个互斥量，Take/Get 成对出现；`Shared_GetSnapshot` 解决“转速是新帧、告警是旧帧”的撕裂问题。

**验收**：解析任务 100Hz 更新 + 显示任务 5Hz 读取，长时间运行数据无跳变；讲得出为什么要互斥量、为什么这里不用无锁。

### M06 · svc_link_mon —— 设备掉线检测

**职责**：每个业务 ID 维护“最后接收时刻”，超时判定掉线/恢复。

```c
void LinkMon_Feed(uint32_t frame_id);        // 收到帧时调用（can_parse 任务里）
void LinkMon_Tick1s(void);                   // health 任务每秒调用，刷新状态
bool LinkMon_IsOnline(uint32_t frame_id);
typedef void (*LinkMon_EventCb)(uint32_t frame_id, bool online);
void LinkMon_RegisterCallback(LinkMon_EventCb cb);   // 掉线/恢复事件回调（显示+上报用）
```

**约束**：阈值 = 连续 `LINK_TIMEOUT_CYCLES` 个周期未收到；**状态迁移必须防抖动**（离线后第一次收到不算恢复，要连续 2 次）；`Feed` 里不准做耗时操作。

**验收**：拔掉总线，**3 秒内** OLED 显示设备离线；插回 2 秒内显示恢复；讲得出“阈值怎么定的、为什么防抖动”。

### M07 · svc_ctrl —— 控制闭环（0x300/0x301）

**职责**：下行指令的发送、确认匹配、超时告警。

```c
bool Ctrl_SendCommand(uint8_t cmd);                  // 发 0x300，记录 pending 序号
void Ctrl_OnAck(const CAN_Frame_t *ack);             // 收到 0x301 调用（can_parse 里）
bool Ctrl_IsPending(void);                            // 有待确认指令？
```

**约束**：每条指令带自增序号，**序号不匹配的 ack 一律丢弃**（防旧确认冒充实到新指令）；200ms 未确认报“指令超时”事件；pending 期间不允许发第二条（返回 false）。

**验收**：发启动指令→设备执行→0x301 确认→链路显示成功；发指令后拔线→200ms 后超时告警；讲得出“远程控制为什么不能开环”。

### M08 · bsp_oled —— SSD1306 I2C 驱动

**职责**：I2C 接口 OLED 的初始化与基本绘制（字符/数字/简单条图）。

```c
void OLED_Init(void);
void OLED_Clear(void);
void OLED_ShowString(uint8_t x, uint8_t y, const char *str);
void OLED_ShowNum(uint8_t x, uint8_t y, uint32_t num, uint8_t len);
void OLED_DrawBar(uint8_t x, uint8_t y, uint8_t w, uint8_t h, uint8_t pct);
```

**约束**：I2C 读写必须过互斥量（display 任务独占，其他任务不得直接调绘制函数）；刷新用局部刷新，不全屏清；绘制函数内部不准有 >10ms 的忙等。

**验收**：一屏显示转速大字 + 温度/电压条 + **掉线状态图标**；连续刷新 1 小时无花屏、无 I2C 卡死。

### M09 · app_latency —— 端到端时延测量

**职责**：用 GPIO 翻转标记“设备发送”和“网关解析完成”两个时刻，供示波器测量。

```c
void Latency_MarkTx(void);      // 模拟设备侧：发 0x100 前翻转 GPIO-A
void Latency_MarkParsed(void);  // 网关侧：can_parse 解包完成后翻转 GPIO-B
```

**约束**：翻转用 BSRR 寄存器直接操作，不准用 HAL_GPIO_TogglePin（耗时不可控）；GPIO 选择避开已用引脚（记录进 app_frame_table 注释）。

**验收**：示波器测两上升沿间隔，**实测值写进简历**（预期 1~3ms 量级）；讲得出这个数由哪几段构成（总线传输+中断+队列+解包）。

---

## 四、Week 3 模块

### M10 · svc_diag —— 诊断服务（参照 ISO-14229）

**职责**：处理 0x500 诊断请求，按服务分发，产生 0x501 响应；管理会话状态与 S3 超时。

```c
void Diag_Init(void);                              // 注册 DID 表（设备型号/固件版本）
void Diag_OnRequest(const CAN_Frame_t *req);       // can_rx→队列→diag_srv 任务调用
void Diag_Tick1s(void);                            // health/定时器调用，S3 超时检查
bool Diag_IsSessionActive(void);                   // 是否扩展会话
```

**约束**：
- 状态机：默认会话 ↔ 扩展会话（0x10 服务 0x03 切换）；**扩展会话 5s 无活动自动回默认**（S3server）
- 单帧响应 ≤8 字节；超出走 ISO-TP（roadmap，本版只画流程图）
- 否定响应 0x7F + NRC 必须覆盖：服务不支持(0x11)、DID 不存在(0x31)、会话未激活(0x7F 类)
- 诊断请求只认 0x500，其他 ID 直接忽略（滤波已保证，双保险）

**验收**：
- 分析仪发 0x22 读设备型号 → 0x501 回正确数据
- 发不存在的 DID → 回 0x7F + 正确 NRC
- 0x10 切扩展会话后静默 6 秒 → 再发 0x22 回“会话未激活”
- **能画出一次完整诊断的时序图**（请求→响应，含 S3 超时）

### M11 · svc_health —— 健康监控 + 看门狗

**职责**：系统运行状态的总出口——栈/堆/总线三件套/任务存活/IWDG 喂狗。

```c
void Health_Init(void);                            // 使能 IWDG + 创建统计量
void Health_TaskBeat(TaskHandle_t task);           // 各任务周期调用，更新存活标记
void Health_Tick1s(void);                          // health 任务：汇总 + 喂狗 + 刷新 OLED/串口
uint32_t Health_GetFreeHeap(void);
void Health_GetBusStats(BusStats_t *out);          // 丢帧数/负载率/错误帧计数
```

**约束**：
- **总线三件套**：队列丢弃计数（队列满时 +1）、总线负载率（每秒帧数）、错误帧/BusOff 计数（读 CAN 错误计数器）
- 任务存活：每个注册任务更新自己的心跳字，health 检测到心跳不动 → 串口+OLED 报出具体任务名
- IWDG 超时设 2s 左右；喂狗只放在 health，**不准别处喂**
- 优先级翻转演示实验挂在这：注释保留“问题版 + 修复版”，README 写实验过程，**坑不留主分支**

**验收**：故意 kill 一个任务（调试器挂起或写死循环），2s 内看门狗复位，串口复位原因打印；拔线观察错误帧计数变化；讲得出“二级防护”分别防什么。

---

## 五、Week 4 模块

### M12 · net_esp —— ESP8266 TCP 透传

**职责**：AT 指令封装，只负责把字节流送到 TCP 连接（透传模式），**不碰 MQTT 协议**。

```c
bool ESP_Init(const char *ssid, const char *pwd);        // 连 WiFi
bool ESP_TcpConnect(const char *ip, uint16_t port);      // 连 Broker
bool ESP_Send(const uint8_t *data, uint16_t len);
void ESP_OnRecv(uint8_t *data, uint16_t len);            // 串口收到透传数据回调
```

**约束**：AT 交互带超时与重试（每条指令 2s 超时，最多 3 次）；串口接收用中断 + 行解析；连接断开主动上报事件（`+IPD` 断开或心跳失败）。

**验收**：上电自动连 WiFi + Broker；拔网线 → 检测到断开 → 插回自动重连；讲得出“AT 方案和自写固件方案的取舍”（这就是面试区分点）。

### M13 · svc_mqtt —— 自写 MQTT 客户端

**职责**：MQTT 3.1.1 报文组装与解析（QoS 0），跑在 ESP 透传之上。

```c
bool MQTT_Connect(const char *client_id);
bool MQTT_Publish(const char *topic, const uint8_t *payload, uint16_t len);
bool MQTT_Subscribe(const char *topic);
void MQTT_Tick1s(void);      // keepalive PINGREQ + 断线重连状态机
```

**约束**：剩余长度字段的变长编码自己实现；只做 QoS 0（PUBACK 收到即止）；keepalive 按 60s 周期发 PINGREQ，2 个周期无响应判定断线；断线重连状态机（空闲→连接→订阅→在线→掉线）。

**验收**：上电上线，看板实时曲线跟随设备数据；拔网线 2 分钟内自动重连恢复；**讲得出 CONNECT/PUBLISH/SUBSCRIBE 三种报文的字节级结构**——这才是“自写 MQTT”的含金量。

### M14 · boot_iap —— Bootloader 最小版

**职责**：独立工程。上电检查 `app_valid` 标志，有效则跳转 App，无效驻留。

```c
typedef void (*pFunction)(void);
void Boot_JumpToApp(uint32_t app_addr);   // 关中断→重定位向量表→设MSP→跳转Reset_Handler
bool Boot_CheckAppValid(void);            // 读 Flash 标志位
void Boot_MarkInvalid(void);              // （预留）升级失败标记，完整版用
```

**约束**：跳转前必须关中断、关外设时钟；App 工程链接地址改到 `0x08004000`（偏移 16KB，够放 Bootloader）；`app_valid` 标志写 Flash 前先擦后写。

**验收**：Bootloader 烧一次后，之后只烧 App 也能启动；把标志位写成无效值 → 驻留 Bootloader 串口提示；讲得出“双分区怎么回滚”（完整版四步流程，roadmap）。

---

## 六、Week 2~3 并行 · 模拟设备侧（M15，独立工程）

**职责**：第二块 C8T6 的固件——模拟下位设备，让台架跑起来。独立 CubeMX 工程，复用 M01 的帧表。

```c
void DevSim_Init(void);
void DevSim_Tick100ms(void);   // 发 0x100（可设转速/温度值，按键或串口改）
void DevSim_Tick1s(void);      // 发 0x200
void DevSim_OnCtrl(const CAN_Frame_t *cmd);  // 收 0x300 → 执行 → 回 0x301
void DevSim_SelfTestDiag(void);              // 开机发一次 0x22 诊断请求
```

**约束**：裸机或简易循环即可，不用 FreeRTOS（降低台架复杂度）；CAN 发送轮询查邮箱；模拟值要能改（串口命令），方便演示不同工况。

**验收**：上电自发 0x100/0x200；收到 0x300 正确执行并回 0x301；收到网关 0x22 请求能正确响应（设备侧诊断）；**整条台架一次点亮**。

---

## 七、模块依赖关系（开工顺序）

```
M01 帧表 ──► M02 bsp_can ──► M03 滤波 ──► M04 解包 ──► M05 共享区 ──► M08 OLED ──► M06 掉线
                                   │                                    │
                                   └──► M15 模拟设备（并行）───► M07 控制闭环 ──► M09 时延
M05 ──► M10 诊断 ──► M11 健康（贯穿）
M11 稳定后 ──► M12/M13 MQTT（增强）──► M14 Bootloader
```

**写码顺序铁律**：每写完一个模块，烧录验证通过再开下一个——**不准五个模块一起写完再烧**，那调试地狱我见过太多学生栽进去。

---

## 八、每周验收总闸门

| 周 | 总闸门（全过才进下周） |
|---|---|
| W1 | loopback 自发自收 + 串口打印一致 + 位时序两组能手推 |
| W2 | 台架两节点互通 + 解包值正确 + OLED 实时显示 + 拔线掉线演示 |
| W3 | 诊断三服务正确（含 S3 超时）+ 控制闭环 + 总线三件套有数 |
| W4 | OTA 最小版跳转成功 + README 完整 + 80% 进度启动投递 |
