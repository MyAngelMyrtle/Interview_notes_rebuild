# FreeRTOS 内核机制

## 1 列表

### 1.1 列表分类

| 大类 | 回答的问题 | 包含内容 |
| --- | --- | --- |
| State List（状态链表） | 任务现在处于什么状态？ | Ready、Delayed、Overflow Delayed、Suspended、Pending Ready、Termination |
| Event List（事件链表） | 任务正在等什么事件？ | Queue、Semaphore、Mutex 等对象内部的等待链表 |

它们分别对应 TCB 的两个列表项：

```c
xStateListItem   // 挂 State List：管理任务状态
xEventListItem   // 挂 Event List：管理等待事件
```

### 1.1.2 State List

| 分类 | 典型变量 | 作用 |
| --- | --- | --- |
| Ready List（就绪链表） | `pxReadyTasksLists[ configMAX_PRIORITIES ]` | 存放已就绪、可运行的任务 |
| Delayed / Overflow Delayed List（延时/溢出延时链表） | `xDelayedTaskList1`、`xDelayedTaskList2`；`pxDelayedTaskList`、`pxOverflowDelayedTaskList` | 两个实际容器；两个指针分别标识当前周期与跨 Tick 溢出的任务 |
| Suspended List（挂起链表） | `xSuspendedTaskList` | 存放没有确定唤醒时间的任务 |
| Pending Ready List（待就绪链表） | `xPendingReadyList` | 调度器暂停期间，临时存放已被唤醒的任务 |
| Termination List（待删除链表） | `xTasksWaitingTermination` | 存放等待 Idle Task 回收资源的已删除任务 |

### 1.1.3 Event List

事件等待链表通常属于 Queue、Semaphore、Mutex 等内核对象：

```c
xTasksWaitingToSend
xTasksWaitingToReceive
```

一个带超时的阻塞任务会同时挂在两个链表中：

```text
Task TCB
├── xStateListItem → Delayed List    // 管理超时
└── xEventListItem → Event List      // 管理事件唤醒
```

`xStateListItem` 与 `xEventListItem` 是两个独立节点，因此任务可以同时处于一个状态链表和一个事件链表中；但同一个 `ListItem_t` 在同一时刻只能属于一个 `List_t`。
# 4 任务创建

## 4.1 xTaskCreate（动态）

```c
BaseType_t xTaskCreate(
    TaskFunction_t pxTaskCode,          // 任务函数（函数指针）
    const char * const pcName,          // 任务名字（调试用）
    uint16_t usStackDepth,              // 栈大小（单位是"字"）
    void * const pvParameters,          // 传给任务函数的参数
    UBaseType_t uxPriority,             // 优先级
    TaskHandle_t * const pxCreatedTask  // 任务句柄存这里
);
```

TCB由内部`pvPortMalloc`自动从`ucHeap`分配，不需要传。

## 4.2 xTaskCreateStatic（静态）

```c
TaskHandle_t xTaskCreateStatic(
    TaskFunction_t pxTaskCode,
    const char * const pcName,
    uint32_t ulStackDepth,
    void * const pvParameters,
    UBaseType_t uxPriority,
    StackType_t * const puxStackBuffer,   // 栈内存
    StaticTask_t * const pxTaskBuffer     // TCB内存
);
```

比动态多两个参数：栈内存地址、TCB内存地址——因为静态创建不经过`ucHeap`，需要开发者提供固定内存。

> [!warning] 在32位MCU平台，1字 = 32位 = 4字节，在FreeRTOS里面创建任务用字

## 3. 关闭调度器与临界区的区别

| 对比项      | `vTaskSuspendAll()`           | `taskENTER_CRITICAL()` |
| -------- | ----------------------------- | ---------------------- |
| 管什么      | 调度器（`uxSchedulerSuspended`）   | 中断（BASEPRI）            |
| 关中断      | ❌                             | ✅                      |
| ISR 能跑   | ✅                             | ❌                      |
| 任务切换     | ❌                             | ❌                      |
| 丢节拍      | **不丢**（记 `xPendedTicks`，恢复时补） | **丢**                  |
| 防 ISR 竞争 | ❌ 不能                          | ✅ 能                    |
| 期间阻塞 API | 禁止                            | 禁止                     |
| 恢复开销     | 高（回放 pended ticks）            | 一条 `msr basepri, #0`   |

**SuspendAll 关闭任务调度，仍能计时，taskENTER_CRITICAL关中断，丢Tick**

### BASEPRI 屏蔽谁

```c
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5
#define configMAX_SYSCALL_INTERRUPT_PRIORITY ( 5 << 4 )    /* 0x50 → BASEPRI */
#define configKERNEL_INTERRUPT_PRIORITY      ( 15 << 4 )   /* 0xF0 → SysTick/PendSV */
```

| 数值 | 临界区内 | 能否调 API |
| --- | --- | --- |
| 0~4 | ✅ 能打断 | ❌ 禁止 |
| 5~15 | ❌ 被屏蔽（含 5） | ✅ |
| NMI / HardFault | ❌ 永不屏蔽 | — |

**数值越小优先级越高**，准确说法是"数值 ≥ 5 被屏蔽"。BASEPRI 只挡一档（高优先级可穿透）；PRIMASK（`cpsid i`）全挡，CM0 无 BASEPRI 只能用后者。

临界区靠嵌套计数，最外层退出才开中断：

```c
vPortEnterCritical(): portDISABLE_INTERRUPTS(); uxCriticalNesting++;
vPortExitCritical():  uxCriticalNesting--; if( !uxCriticalNesting ) portENABLE_INTERRUPTS();
```

坑：分组必须全给抢占优先级；阈值要写 `5 << 4` 而非 `5`；写 0 等于关闭屏蔽。

## 怎么选

```text
与 ISR 共享数据？
├─ 是 → taskENTER_CRITICAL
└─ 否 → 操作极短？ → 都可以，Critical 更简单
         操作较长/多对象/要遍历？ → vTaskSuspendAll
```

### 坑

1. SuspendAll 期间禁止阻塞调用（有 `configASSERT`保护，否则可能出现任务被标记为阻塞但继续执行）
2. SuspendAll **不能**保护与 ISR 共享的数据 —— ISR 照跑、照改就绪链表
3. Critical 必须短，超过一个节拍就丢 tick
4. ISR 里两者都不能用，需 `portSET_INTERRUPT_MASK_FROM_ISR()`

## 代价

| 场景              | 关中断时机   | 结果        |
| --------------- | ------- | --------- |
| Critical（N 拍）   | 分散在 N 拍 | 丢节拍，延迟分散  |
| SuspendAll（N 拍） | 集中在恢复时  | 不丢节拍，延迟集中 |
## 2 SysTick

```c
void xPortSysTickHandler( void )
{
    // 屏蔽中断
    vPortRaiseBASEPRI();
    {
        /* RTOS 系统节拍计数加 1，并且找到最高优先级的任务。 */
        if( xTaskIncrementTick() != pdFALSE )
        {
            /* 进行上下文切换。将 PendSV 中断设置为挂起状态。 */
            portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
        }
    }
    /* 清零 BASEPRI，解除本 ISR 临时设置的中断屏蔽； */
    vPortClearBASEPRIFromISR();
}
```

### 2.1 硬件原理

24 位硬件递减计数器，中断只是它的可选输出：计数到 0 → 自动重装 → 置 COUNTFLAG → 挂起异常。

| 特性   | 意义                     |
| ---- | ---------------------- |
| 独立计数 | 不依赖 CPU 指令，睡眠也照走       |
| 自动重装 | 周期严格等于 `LOAD+1` 个时钟    |
| 硬件触发 | 全程无软件参与，**计时不受任务负载影响** |

`LOAD = 时钟频率 / 节拍频率 - 1`（EGG: 72 MHz + 1 kHz → 71999）。
## 5.PendSV，SVC

### 5.1 三个专用异常

| 异常 | 作用 | 触发 | 优先级 |
| --- | --- | --- | --- |
| SVC (11) | 启动第一个任务 | `svc 0`，仅一次 | 中 |
| PendSV (14) | **上下文切换** | 写 ICSR | 最低 |
| SysTick (15) | 时基 | 计数器到 0 | 最低 |

### 5.2 PendSV 是什么

可纯软件触发的异常：置位 `ICSR` bit28（PENDSVSET），清除 bit27。写后需 `DSB` + `ISB`。

```c
portNVIC_SHPR3_REG |= portNVIC_PENDSV_PRI;   /* 15 << 4 = 0xF0，最低 */
```

### 5.3 为什么要优先级最低

1. **不能在 ISR 中间切**：中断运行在 MSP，任务运行在 PSP，允许任务抢占 handler 会让栈与异常返回路径混乱
2. **合并切换请求**：PendSV有多个唤醒源，但 pending 只有一位，从而最终只执行一次 PendSV（惰性切换，多个请求只执行一次）
3. **不抬高中断延迟**：PendSV 优先级最低，避免多个切换拉长中断响应时间。

同优先级防的是抢占。从Systick 到 PendSV，返回时使用尾链，不直接出栈，这一步避免了进入PendSV时反复压栈当前任务的PSP

### 5.4 切换时序

```text
① A 运行（Thread / PSP）
② SysTick：硬件压 R0-R3/R12/LR/PC/xPSR 到 A 的栈；返回真 → 置 PENDSVSET
③ SysTick 退出 → 尾链进入 PendSV
④ PendSV：保存 r4-r11 → 栈顶写回 TCB → vTaskSwitchContext 选 B
⑤ 恢复 B 的 r4-r11 → msr psp → bx r14
⑥ 硬件弹出 B 的 8 个寄存器 → B 继续运行
```

> [!note] 压栈归属
> 由**第一个打断任务的异常**完成（这里是 SysTick），不是 PendSV；压到 PSP（任务栈）。不切换也照压、退出时弹回 —— 这就是中断延迟里的 ~12 周期。尾链进入 PendSV 时**复用同一份帧**。

**这个细节其实还挺重要，很多八股说当前任务的PSP压栈在PendSV完成，其实不是，这取决于哪个异常最先从 Thread 进入 Handler**
### 5.5 汇编

```asm
mrs  r0, psp                /* 取任务栈顶 */
stmdb r0!, {r4-r11}         /* 软件保存 r4-r11 */
str  r0, [r2]               /* 栈顶写回 TCB[0] */
stmdb sp!, {r3, r14}        /* 保存 TCB 指针与 EXC_RETURN */
msr  basepri, %0
bl   vTaskSwitchContext     /* 选下一个任务 */
msr  basepri, #0
ldmia sp!, {r3, r14}
ldr  r1, [r3]
ldr  r0, [r1]               /* 新任务栈顶 */
ldmia r0!, {r4-r11}
msr  psp, r0
bx   r14                    /* 异常返回 */
```

### 5.6 硬件压栈的 8 个寄存器 + 软件压栈的 8 个寄存器

**这里不举例带FPU的情况，想了解自己去搜**

| 保存者 | 寄存器 | 原因 |
| --- | --- | --- |
| 硬件 | R0-R3、R12、LR、PC、xPSR | 异常进入自动压栈 |
| 软件 | R4-R11 | callee-saved，硬件不压 |

```text
高地址  [xPSR PC LR R12 R3 R2 R1 R0]   ← 硬件帧（SysTick 进入时）
         [R4~R11                   ]   ← 软件追加，SP 指这里
```

尾链时两者**共用同一份硬件帧**，不会压两次。若 PendSV 由 `taskYIELD()` 直接从任务触发，压栈就发生在 PendSV 进入时 —— 规则始终是"第一个打断任务的异常负责压栈"。切换开销约 100~200 周期（无 FPU），开 FPU 接近翻倍。

### 5.7 SVC：只做一次"进门"

调度器启动的最后一步：

```asm
xPortStartScheduler() → prvPortStartFirstTask():
    msr msp, <复位保存的 MSP>
    cpsie i            /* 开中断 */
    svc 0              /* ★ 触发 SVC */
```

SVC handler 做的事：

```asm
ldr  r0, [r1]              /* 第一个任务的栈顶 */
ldmia r0!, {r4-r11}        /* 恢复 r4-r11 */
msr  psp, r0               /* ★ PSP 指向任务栈 */
msr  basepri, #0           /* 开中断 */
orr  r14, #0xd             /* EXC_RETURN = 0xFFFFFFFD */
bx   r14                   /* 异常返回 → Thread + PSP */
```

**本质：把"进入第一个任务"伪造成一次异常返回**，栈上的 8 个寄存器帧由 `pxPortInitialiseStack` 提前摆好。

**为什么必须用 SVC**：不能直接 `bx` 到任务函数 —— 那样还在 Handler 模式、用 MSP。只有"异常返回"能让硬件同时完成三件事：切到 Thread 模式、SP 换成 PSP、弹出预置的 8 个寄存器。此后 SVC 不再执行。

| EXC_RETURN | 返回 | 栈 | FP |
| --- | --- | --- | --- |
| 0xFFFFFFF1 | Handler | MSP | 无 |
| 0xFFFFFFF9 | Thread | MSP | 无 |
| 0xFFFFFFFD | Thread | PSP | 无 |
| …E1 / E9 / ED | 同上 | 同上 | 有 |

### 5.8 一次切换的完整链路

```text
① 请求：某处写 ICSR（PENDSVSET）        ← 只登记，不切换
② 受理：PendSV handler 执行（5.5 的汇编）
③ 决策：bl vTaskSwitchContext() → 选下一个任务
④ 恢复：新任务 r4-r11、PSP → bx r14
```

### 5.9 请求方：谁写 ICSR

| 场景 | 入口 |
| --- | --- |
| 任务主动让出 | `taskYIELD()` |
| API 内部触发 | `portYIELD_WITHIN_API()` |
| ISR 唤醒更高优先级任务 | `portYIELD_FROM_ISR()` |
| SysTick 节拍到期 | `xPortSysTickHandler`（条件见 5.11） |
| 调度器恢复 | `xTaskResumeAll()` 处理 `xYieldPending` |

这张表**只负责"请求"**，不会立刻切换到PendSV中

### 5.10 为什么"请求"与"执行"要分两段

假设让 SysTick 直接切：

1. 切换代码写进 ISR → ISR 变长，其他中断延迟增大
2. 中断嵌套时切换点不确定 → 可能在中断链中间切栈
3. 多个唤醒源无法合并 → 每来一个请求就切一次
4. 没有统一切换点 → 调试钩子、运行时统计都难做

所以设计成：**请求随手可发（写一位，几纳秒），执行只在唯一时刻（PendSV，最低优先级，所有 ISR 之后）**。

### 5.11 SysTick → PendSV 的触发条件

`xTaskIncrementTick()` 返回真只有三种情况：

```c
/* 1. 到期任务优先级严格高于当前任务 */
if( pxTCB->uxPriority > pxCurrentTCB->uxPriority ) xSwitchRequired = pdTRUE;
/* 2. 当前优先级就绪任务 > 1（时间片轮转） */
if( listCURRENT_LIST_LENGTH( &pxReadyTasksLists[ pxCurrentTCB->uxPriority ] ) > 1U ) ...
/* 3. 存在欠账 */
if( xYieldPending != pdFALSE ) ...
```

| 场景 | 返回 | PendSV |
| --- | --- | --- |
| 无任务到期 | FALSE | ❌ |
| 低优先级任务到期 | FALSE | ❌ |
| 更高优先级任务到期 | TRUE | ✅ |
| 同优先级且开时间片 | TRUE | ✅ |
| 调度器挂起 | FALSE | ❌（记 `xPendedTicks`） |
| 协作式（`PREEMPTION=0`） | FALSE | ❌ |

用 `>` 而非 `>=`：同优先级交给时间片统一处理。置位只是"登记"，`vTaskSwitchContext` 每次都重新选，不绑定特定任务。

### 5.12 面试速答

| 问题               | 答案                                  |
| ---------------- | ----------------------------------- |
| PendSV 是什么       | 可软件挂起的异常，专用于切换，优先级最低                |
| 为什么不在 SysTick 里切 | 延迟到所有 ISR 之后、合并请求、避免中断中切栈           |
| 硬件保存哪些           | R0-R3、R12、LR、PC、xPSR                |
| 为何软件还要存 r4-r11   | callee-saved，硬件不压                   |
| `0xFFFFFFFD`     | Thread + PSP + 无 FP                 |
| 尾链               | 退出时不回任务，直接进下一个 pending 异常           |
| SysTick 何时请求切换   | 更高优先级到期 / 同优先级可轮转 / `xYieldPending` |
| `xYieldPending`  | 记录"该切但当时不能切"的欠账                     |
*callee-saved是啥？*

*简单解释一下，通用寄存器分为两组，R0-R3、R12、LR、标志位是**caller-saved**（调用者保存）**R4-R11**、SP属于**callee-saved**（被调用者保存），caller-saved被调用函数可以随意破坏；调用者若想保留，**需要自己提前存**，而callee-saved被调用函数若要用，**必须先保存、返回前恢复**，保证调用前后值不变。为什么这么干呢？因为 **R0-R3、R12**用于传参、返回值、临时计算 —— 每个函数都在用，若由被调用者保存，几乎每个函数都要多存 5 个寄存器，代价巨大。所以约定「谁要跨调用保留，谁自己存」- **R4-R11**被编译器用来放**长期变量**（循环计数器、跨调用的局部变量）。这类变量一旦调用发生就得保留，交给被调用者保存最划算 —— 它只在**真的用到**时才压栈。对于普通中断来说，如果编译器生成的序言使用了**R4-R11**，会自己完成push/pop，但是**在沟槽的任务切换中，没有被调用者能恢复callee-saved**（切换后不会返回原处），所以只能显式地切换。*

**这一大坨可以简单了解，我觉得正常人不会问这个**

## 6 xPendedTicks

调度器挂起时 SysTick 仍中断，但只记账：

```c
if( uxSchedulerSuspended == pdFALSE ) { /* 正常处理 */ }
else                                  { ++xPendedTicks; }
```

`xTaskResumeAll()` 逐拍回放：

```c
do { if( xTaskIncrementTick() != pdFALSE ) xYieldPending = pdTRUE; } while( --xPendedCounts );
xPendedTicks = 0;
```

| 方面 | 一次性 +N |
| --- | --- |
| 到期扫描 | ✅ 正确（值驱动） |
| `xNextTaskUnblockTime` | ✅ 正确 |
| 32 位回绕交换链表 | ❌ 必错（只在 `==0` 时触发） |
| tick hook 次数 | ❌ 少 N-1 次 |

逐拍不是为了扫描，而是让**回绕检测与每拍副作用**自动正确；代价是回放期间处于临界区。
