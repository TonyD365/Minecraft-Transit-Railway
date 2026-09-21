# 信号桌与联锁系统 — 设计方案

> 状态：设计阶段，尚未实现
> 目标分支：本 fork 的 `master`。
> 资源归属：方块与物品的模型、贴图为**占位资源**，由上游自行设计；
> **信号桌的控制界面是本项目的最终成品，不是占位**，详见 §0.1。

---

## 0. 范围

在 MTR 现有的「轨道 + 信号色」机制之上，增加一套**贴近现实的联锁系统（interlocking）**：

1. **信号仪表板（Signal Dashboard）** — 手持物品，仅 OP 可用。负责管理：新建信号桌、划定控制区域、
   指定每个桌子可以控制哪些信号机与道岔、设置该桌子的玩家黑白名单。
2. **信号桌（Signal Desk）** — 方块 + 座椅实体。负责操作：坐下后弹出港铁 OCC 风格的控制界面，
   排进路、取消进路、单动道岔。
3. **联锁引擎（interlocking engine）** — 运行在服务端。负责进路锁闭、冲突检查、自动扳道岔、
   接近锁闭、进路解锁，以及在进路不正确时把列车拦停。

界面是**依据公开资料重制**的港铁 MTR OCC / NX 信号工作站视觉风格。
本项目为 GPL 授权，不会复制任何第三方代码进来。

### 0.1 资源归属：哪些是占位，哪些是成品

向上游提交 PR 时，两部分的定位**完全不同**：

| 部分 | 定位 | 谁负责 |
|---|---|---|
| 信号桌方块的模型与贴图 | **占位（placeholder）** | 上游自行设计 |
| 仪表板物品的贴图 | **占位** | 上游自行设计 |
| 方块状态、配方、掉落表 | **占位** | 上游自行调整 |
| **信号桌的控制界面（§7）** | **最终成品** | 本项目负责，上游直接使用 |
| 联锁引擎、数据模型、网络包（§2–§5） | **最终成品** | 本项目负责 |

由此得出一条**硬性技术约束**：

> **控制界面不得依赖任何贴图文件。**

因为如果界面用了 `.png` 资源，而那些资源被归类为「上游自己做」，界面就会在上游那边变成残缺状态。
因此 `SignalDeskScreen` 与 `SignalDeskDiagram` 必须**纯代码绘制（procedurally drawn）**：

* 轨道、道岔、信号灯点、进路高亮 —— 全部用矩形与线段绘制原语（drawing primitive）画出
* 文字一律使用 Minecraft 原版字体（vanilla font），不引入自定义字体图集
* 颜色全部写成代码中的常量（§7.2），不从资源包读取
* 唯一允许的外部依赖是**语言文件**（`en_us` / `zh_cn`），因为那是翻译而不是美术

这样，界面**自带完整品相、零外部资源依赖**，上游合并后即刻可用；
而方块长什么样，则完全交给他们决定。

---

## 1. 引擎约束（已对照代码验证）

以下都是阅读 `libs/Transport-Simulation-Core-0.0.1.jar` 与 mod 源码后确认的事实。
整份方案的形态都是被这几条约束决定的。

### 1.1 模拟核心是预编译的黑盒

`Transport-Simulation-Core-0.0.1.jar` 掌管列车运行与区段预约（block reservation）。
我们**改不了它**，只能调用它公开的 API。

### 1.2 列车的停车点由 rail 的信号色决定，而不是由信号机方块决定

摘自 `org.mtr.core.data.Rail`：

```java
public IntAVLTreeSet getSignalColors();
public void applyModification(SignalModification);
public void blockRail(LongArrayList colors);
private static void reserveRail(...);
boolean isBlocked(long, Rail$BlockReservation);
```

这些**全部是 rail 级别的**。核心库里根本没有「世界中的信号机方块」这个概念。

而 `BlockSignalBase` 只做两件事：

```java
public int getActualAspect(boolean occupied, boolean isBackSide)  // 读取状态 → 显示灯色
public void checkForRedstoneUpdate(...)                           // 接红石 → 触发封锁
```

**结论：** 列车停在**「rail 上信号色区段（colour span）的边界」**。
我们想让车停在哪里，就把色段边界画在哪里。物理信号机方块只是**标示器（annunciator）**。
**道岔旁边不需要摆任何信号机。**

### 1.3 手动封锁只维持 1 秒

```java
private static final int MANUAL_BLOCK_DURATION = 1000;   // 毫秒
private long manualBlockCooldown;
```

`blockRail()` **只保持 1000 毫秒**。要让道岔在整条进路存续期间一直锁住，
服务端必须**每 tick 重新下发一次封锁** —— 也就是**心跳（heartbeat，保活机制）**。
这是整个方案最大的技术风险，因此放在第 2 阶段最先验证。

### 1.4 这个 mod 目前没有任何玩家权限系统

整个 `org/mtr/mod` 里找不到一处 `hasPermissionLevel` 检查。我们是在造**第一套**权限机制。

这带来一个**引导漏洞（bootstrapping hole）**：如果仪表板谁都能合成，
被拉黑的玩家只要做一个仪表板，就能把自己加回白名单，名单等于白设。
因此**仪表板必须在服务端做 OP 校验**，而不能只是把配方藏起来。

---

## 2. 数据模型

### 2.1 `SignalDesk` — 信号桌

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | `long` | 唯一编号 |
| `name` | `String` | 显示在界面顶栏 |
| `areaMin`, `areaMax` | `BlockPos` | 该桌子控制的长方体区域 |
| `controlledSignalIds` | `LongOpenHashSet` | 允许操作的信号机 |
| `controlledPointIds` | `LongOpenHashSet` | 允许操作的道岔组 |
| `access` | `SignalDeskAccess` | 玩家黑白名单 |
| `protectionMode` | `ProtectionMode` | `AREA_ENTRY` 或 `AT_POINTS` |
| `deskBlockPos` | `BlockPos` | 实体桌子的位置 |

### 2.2 `SignalDeskAccess` — 黑白名单

```java
enum Mode { OPEN, ALLOWLIST, BLOCKLIST }    // 开放 / 白名单 / 黑名单
Mode mode;                                  // 默认 OPEN
ObjectOpenHashSet<UUID> players;            // 存 UUID，绝不存名字（玩家会改名）
boolean opBypass;                           // OP 是否无视名单，默认 true
```

**坐下时校验一次，之后每个操作包再校验一次** —— 玩家有可能在已经坐着的时候被移出名单。

### 2.3 `Signal` — 逻辑信号机

挂靠在**世界中已有的 MTR 信号机方块**上，扫描桌子区域自动发现，**不需要新增方块类型**。

| 字段 | 含义 |
|---|---|
| `id` | 由 `BlockPos` 推导 |
| `pos`, `facing` | 位置与朝向 |
| `guardedRailIds` | 它正前方防护的 rail |
| `signalColors` | 它持有的信号色（读自现有 block entity） |
| `aspect` | 当前显示的灯色，见 §4 |
| `heldRouteId` | 当前由它开放的进路，没有则为 `null` |

信号机摆在**进路边界**（咽喉区入口、站台端），**不摆在道岔旁**。

### 2.4 `PointGroup` — 道岔组

自动发现：rail 图中**连接数大于 2 的节点**即为分歧点（junction）。

| 字段 | 含义 |
|---|---|
| `id` | 由节点 `Position` 推导 |
| `nodePos` | 分歧点位置 |
| `normalRailId` | **定位（normal）** —— 直向 / 正线一侧 |
| `reverseRailId` | **反位（reverse）** —— 侧向 / 分歧一侧 |
| `currentPosition` | `NORMAL` / `REVERSE` |
| `lockedByRouteId` | 非空 = 已锁闭，不能手动扳动 |
| `isFlankFor` | 仅作为侧防（flank protection）被锁的进路 |

所谓「扳道岔」，实质是**选择哪一条腿不被封锁**，另一条腿由心跳持续封锁。

### 2.5 `SignalRoute` — 进路

| 字段 | 含义 |
|---|---|
| `id` | 唯一编号 |
| `entranceSignalId` | **N（入口）** —— 列车从这里出发 |
| `exitSignalId` | **X（出口）** —— 进路到此为止 |
| `railIds` | 进路经过的全部 rail，**按顺序存**（为 v2 分段解锁预留） |
| `requiredPoints` | `Map<道岔id, NORMAL/REVERSE>` |
| `overlapRailIds` | 出口信号机之后的**安全超距（overlap）**，见 §5.4 |
| `flankPointIds` | 侧防道岔，见 §5.5 |
| `state` | 见 §3 |
| `approachLockTimer` | 取消进路后的延时解锁倒计时（毫秒） |

---

## 3. 进路状态机（state machine）

```
                     拒绝（附带原因）
                   ┌──────────────┐
                   │              │
  IDLE ──排进路──> REQUESTED ──通过──> LOCKED ──列车接近──> APPROACH_LOCKED
    ^                                    │                        │
    │                                    │ 取消（附近无车）        │ 列车越过信号机
    │                                    │ = 立即生效              v
    │                                    v                    OCCUPIED
    └───────────── RELEASING <───────────┴──── 列车出清进路 ───────┘
```

| 状态 | 中文 | 含义 |
|---|---|---|
| `IDLE` | 空闲 | 未排进路，入口信号机显示停车 |
| `REQUESTED` | 请求中 | 正在跑检查（只持续一个 tick） |
| `LOCKED` | 已锁闭 | 道岔已扳妥并锁死，入口信号机已开放 |
| `APPROACH_LOCKED` | 接近锁闭 | 已有列车凭进行信号接近 |
| `OCCUPIED` | 占用 | 列车已进入进路内 |
| `RELEASING` | 解锁中 | 列车出清，锁正在逐步解除 |

---

## 4. 信号显示（aspect）

MTR 的 `getActualAspect` 返回 `0..3`，映射如下：

| 值 | 显示 | 中文 | 出现条件 |
|---|---|---|---|
| `1` | **Danger**（红） | 停车 | 未排进路，或进路已被占用 |
| `3` | **Caution**（黄） | 注意 | 已排进路，但出口之后的区段被占用 |
| `2` | **Preliminary caution**（双黄） | 预告注意 | 前方两个区段被占用（仅四显示信号机） |
| `0` | **Clear**（绿） | 进行 | 已排进路且前方空闲 |

信号机**只能由联锁开放，玩家不能直接点亮它**。
玩家申请的是**一条进路**，灯色只是这条进路的**结果**。

---

## 5. 联锁逻辑

运行在服务端，每 tick 执行 `SignalInterlocking.tick()`。

### 5.1 排进路（NX 入口-出口法）

玩家先点入口信号机 N，再点出口信号机 X。

```
1. 寻路 PATH       找 N → X 的 rail 路径。找不到 → 拒绝「无此进路」
2. 权限 AUTHORITY  路径上每个信号机与道岔都必须在本桌子的授权集合里
                   → 否则拒绝「非本桌管辖」
3. 冲突 CONFLICT   railIds 与 overlapRailIds 中任何一条不得属于另一条非空闲进路
                   → 否则拒绝「进路冲突」
4. 道岔 POINTS     所需道岔必须未被锁，或已锁在相同位置
                   → 否则拒绝「道岔已锁闭」
5. 空闲 CLEAR      railIds 上当前不得有列车 → 否则拒绝「区段占用」
6. 超距 OVERLAP    X 之后的安全超距必须空闲 → 否则拒绝「超距不可用」
7. 侧防 FLANK      侧防道岔必须能扳到防护位 → 否则拒绝「侧防不可用」

  → 七项全过：
8. 扳岔 CALL       把所有所需道岔（含侧防道岔）扳到位
9. 锁闭 LOCK       给每个道岔写入 lockedByRouteId；状态置为 LOCKED
10. 封锁 BLOCK     启动心跳，持续封锁所有不属于本进路的分歧腿
11. 开放信号        解除本进路 rail 的封锁；N 按 §4 显示进行信号
```

拒绝**不会静默** —— 原因写入桌面消息栏（§7.6），同时该进路在图上**闪红约 2 秒**。

### 5.2 取消进路

玩家**右键入口信号机**。

按现实做法分两种情况：

* **附近无车接近** → **立即取消**。信号转红，道岔解锁，进路回到 `IDLE`。
* **已有列车凭进行信号接近**（`APPROACH_LOCKED`）→ 触发**接近锁闭（approach locking）**。
  信号**立刻转红**，但道岔**继续锁闭**一段**延时解锁**时间（默认 120 秒，可配置）。
  这是现实中防止「司机已经看到绿灯，你却把他脚下的道岔扳走」的保护措施。
  桌面上会显示**实时倒计时**。

已进入 `OCCUPIED` 的进路**完全不能取消**，只能由 OP 执行**紧急解锁**（§6.4）。

### 5.3 进路解锁

**v1 —— 整进路解锁（route release）。** 列车出清进路最后一条 rail 时，所有锁一次性解除。

**v2 —— 分段解锁（sectional release）。** 列车走过一段就解一段，后续列车可以更早进入咽喉区。
v1 不做，但 `railIds` 特意存成**有序列表**，将来加上去不需要做数据迁移（data migration）。

### 5.4 安全超距（overlap）

出口信号机之后的一小段轨道，随进路一起预留，保证列车**冒进（overrun）**少许时仍有受保护的轨道。
默认取 X 之后的第一条 rail，可按信号机单独配置。与进路同时锁闭、同时解锁。

### 5.5 侧防（flank protection）

**不在进路上**、但有可能让其他列车侧向冲入本进路的道岔，
会被强制扳到**背离进路**的位置并锁死。
专门用来防那个经典事故：另一列车从侧线溜出来，撞进你的进路。

### 5.6 心跳封锁

```java
// 每个服务端 tick，遍历所有处于 LOCKED / APPROACH_LOCKED / OCCUPIED 的进路
for (long railId : route.blockedLegRailIds) {
    rail.blockRail(route.blockColors);   // 1000 毫秒后过期，所以必须重发
}
```

如果服务器卡顿超过一秒，锁会短暂失效，下一 tick 自动补上。
但**失效方向是危险的** —— 封锁一旦失效，rail 就变成可通行。
因此我们额外加一道保险：**只要心跳漏拍，就立刻把入口信号机压回停车（Danger）**，
而不是单纯依赖封锁本身。这叫**故障导向安全（fail-safe）**。

### 5.7 进路排错时拦停列车 —— 这正是需求的核心

把原始需求精确重述一遍：

> 进路排去了 B 站台，但这趟车该走 A 站台。
> 通往 A 的那条腿被心跳封锁着，列车预约不到，于是**停在被封锁色段的边界**。
> 信号员取消进路，重排 N → A。封锁解除，信号开放，列车继续走。

**具体停在哪里**，由 `protectionMode` 决定：

| 模式 | 停车位置 | 说明 |
|---|---|---|
| `AREA_ENTRY`（默认） | 整个咽喉区的入口信号机 | 贴近现实。一架信号机防护区内所有道岔，道岔旁无需信号机。 |
| `AT_POINTS` | 紧贴道岔之前 | 调试时更直观。同样不需要在那里摆信号机方块。 |

两者是**同一套代码**，区别只是**把色段边界画在哪里**。

---

## 6. 操作方式

### 6.1 信号桌界面（`SignalDeskScreen`）

| 操作 | 效果 | 说明 |
|---|---|---|
| 左键信号机 N，再左键信号机 X | 排进路 N → X，道岔自动扳到位 | 标准 NX 操作 |
| **右键入口信号机** | 取消进路（立即，或进入接近锁闭倒计时） | 信号转红 |
| 左键道岔组 | 在定位 / 反位间切换 —— 若已被进路锁闭则拒绝 | 单动道岔 |
| 左键信号机后按 Esc / 右键空白处 | 放弃这次未完成的入口选择 | 撤销选择 |
| 悬停任意对象 | 提示框：编号、状态、所属进路、锁闭原因 | |
| 滚轮 / 拖拽 | 缩放与平移轨道示意图 | |

已选中的入口信号机会**高亮**，让操作员随时知道有一次 NX 操作只做了一半。

### 6.2 信号仪表板界面（`SignalDashboardScreen`，仅 OP）

* 信号桌的列出 / 新建 / 删除 / 改名
* 在 `WidgetMap` 上**拖拽矩形**划定控制区域
* 两个勾选列表：区域内的信号机、区域内的道岔组 —— 打勾即授权
* 权限面板：模式下拉框（`OPEN` / `ALLOWLIST` / `BLOCKLIST`）、玩家列表、增删按钮、`opBypass` 勾选框
* `protectionMode` 下拉框
* **校验警告（validation）**，直接显示在界面里：
  * 区域内某个道岔组，任何一个接近方向上都没有信号机防护
  * 两个桌子的授权集合重叠（可能会互相抢同一组道岔）
  * 某信号机被授权，但它的 rail 在区域之外

### 6.3 坐下

右键桌子 → 服务端校验 `SignalDeskAccess` → 生成 `EntitySignalDeskSeat` → 玩家骑上去 →
视角对准屏幕 → 打开 GUI。
站起、死亡、掉线都会移除座椅实体。**之后每个操作包都会重新校验权限。**

### 6.4 紧急解锁（emergency release）

仅 OP 可用，弹确认框，操作连同玩家名字写入消息日志。
无视状态强行解除一条进路上的所有锁。
这是进路卡死或状态不同步时的**逃生出口（escape hatch）**。

---

## 7. 界面 —— 港铁 OCC 风格

### 7.0 两种显示状态

信号桌的屏幕有**两种**显示方式，不是只有坐下才看得见：

| 状态 | 显示方式 | 目的 |
|---|---|---|
| **无人使用** | 屏幕在**世界中实时渲染**在方块表面 | 控制室有「活着」的感觉；路过与围观的玩家能看到实时状态 |
| **有人坐下** | 打开**可操作的全屏界面** | 真正下达指令 |

世界中渲染这一点，本 mod 已有成熟先例 —— **PIDS 就是在方块表面实时绘制的**，
因此走的是现成路子，不需要新机制。两种状态**共用同一套绘制代码**
（`SignalDeskDiagram`），区别只在于：世界中渲染的版本**只读、不接受输入**，
且按距离降低刷新率以节省性能。

### 7.1 绘制方式

用 `ScreenExtension` 重制，缩放平移复用现成的 `WidgetMap`。
视觉语言参照港铁 MTR OCC / NX 信号工作站。

### 7.2 配色

| 元素 | 颜色 |
|---|---|
| 背景 | 近黑 `#0A0A0A` |
| 轨道：空闲且未排进路 | 深灰 `#555555` |
| 轨道：进路已排 | 白 `#FFFFFF` |
| 轨道：被列车占用 | 红 `#FF3030` |
| 轨道：已锁闭但尚未占用 | 琥珀 `#FFB000` |
| 道岔：反位 | 该侧腿画粗 |
| 信号机：停车 | 红点 |
| 信号机：注意 | 黄点 |
| 信号机：进行 | 绿点 |
| 已选中的入口信号机 | 闪烁白环 |

### 7.3 轨道示意图

**示意式（schematic），不是地理式** —— rail 被拉直成水平线段，道岔处用短 45° 斜线连接，
这正是真实控制盘的画法。
道岔组画成从正线分出的一条短支线，**已选中的腿画实线，未选中的腿画暗**。

### 7.4 列车标示（train describer）

被占用的区段上带一个小方框，写着该车的车次 / 交路编号，**随列车逐区段移动**。
这是让界面「像真的」的关键细节。

### 7.5 顶栏

桌子名称 · 控制区域 · 游戏内时间 · 连接状态。

### 7.6 消息日志

右下角，滚动显示，最新在上。
每一次排进路、取消、拒绝（含原因）、扳道岔、紧急解锁，都带**时间戳**和**操作玩家**。

### 7.7 底部按钮栏

`排进路` · `取消进路` · `单动道岔` · `紧急解锁`（OP） · `缩放至全图`
—— 对应真实工作站上的模式按钮，照顾更习惯按按钮而不是点序列的玩家。

---

## 8. 安全模型

| 层级 | 校验内容 | 在哪里执行 |
|---|---|---|
| 仪表板物品 | OP（`hasPermissionLevel(2)`） | 服务端，包处理器内 |
| 坐上信号桌 | `SignalDeskAccess` | 服务端，`onUse` 内 |
| 任何桌面操作 | `SignalDeskAccess` **重新校验** + 桌子的信号/道岔授权集合 | 服务端，每一个操作包内 |
| 紧急解锁 | OP | 服务端 |

**绝不信任客户端传来的任何东西。**
客户端界面只是一个**渲染器 + 输入设备**，所有决策都在服务端做出，然后广播回来。

---

## 9. 文件规划

```
data/
  SignalDeskData.java          信号桌记录 + NBT/JSON 序列化
  SignalDeskAccess.java        黑白名单
  SignalRoute.java             进路记录 + 状态
  PointGroup.java              道岔组记录
  SignalRef.java               逻辑信号机，包装现有 block entity
  SignalDeskRegistry.java      全部信号桌的存档持久化
  SignalInterlocking.java      联锁引擎：tick、排路、取消、解锁、心跳
  SignalPathfinder.java        rail 图寻路 N → X，顺带收集沿途道岔

block/
  BlockSignalDesk.java         方块 + block entity      【占位模型】

item/
  ItemSignalDashboard.java     OP 限定手持物品          【占位贴图】

entity/
  EntitySignalDeskSeat.java    隐形座椅

screen/
  SignalDashboardScreen.java   管理界面                【成品，纯代码绘制】
  SignalDeskScreen.java        操作界面（OCC 风格）     【成品，纯代码绘制】
  SignalDeskDiagram.java       轨道示意图渲染器         【成品，纯代码绘制】

packet/
  PacketUpdateSignalDesk.java     仪表板 → 服务端（配置）
  PacketSignalDeskAction.java     信号桌 → 服务端（排路/取消/扳岔/紧急解锁）
  PacketSignalDeskState.java      服务端 → 客户端（实时状态广播）

resources/
  blockstates、models、textures、loot table、配方      【全部占位，上游自行设计】
  语言文件：en_us、zh_cn                               【成品】
```

每个占位资源都写上 `TODO: placeholder asset — to be replaced upstream`，
让美术和逻辑可以**分开审阅**，也更容易被上游接受。
注意界面相关的三个文件**不带这个标记** —— 它们是成品，不需要上游替换（见 §0.1）。

---

## 10. 实现顺序

| 阶段 | 产出 | 验证什么 |
|---|---|---|
| 1 | 数据模型、桌子方块、仪表板物品、座椅实体、占位资源 | 能放下、能坐上去、能干净地站起来 |
| 2 | **心跳封锁** —— 让一条 rail 无限期保持封锁 | 1000 毫秒过期被攻克。**风险最高，所以最先做。** |
| 3 | 仪表板界面：OP 校验、划区域、信号/道岔授权、黑白名单 | 配置重启后仍在 |
| 4 | 信号桌界面（静态）：轨道图、信号机、道岔、占用染色 | OCC 界面出来了且看得懂 |
| 5 | 寻路 + 排进路 + 七项联锁检查 + 自动扳道岔 | 排进路能真的扳岔并开放信号 |
| 6 | 取消进路、接近锁闭倒计时、进路解锁、**排错时拦停列车** | **需求的核心行为** |
| 7 | 安全超距、侧防、紧急解锁、消息日志、服务端全面校验 | 安全机制补齐 |
| 8 | 编译、进游戏实测、提交 `master` | 交付 |

第 2 阶段**故意排在第二位**：万一心跳撑不住，第 5–7 阶段都得换机制，
在界面写出来之前就发现这件事，代价小得多。

---

## 11. v1 的已知局限

* **只做整进路解锁，不做分段解锁** —— 同一时间咽喉区只容一列车。数据结构已为 v2 预留。
* **没有自动排进路（ARS, Automatic Route Setting）** —— 每条进路都要手排。
  按时刻表自动排路是很自然的后续扩展。
* **安全超距固定为一条 rail**，不是随速度变化的**浮动超距（swinging overlap）**。
* **道岔是靠 rail 图拓扑推断出来的**，复杂布置（交分道岔、交叉渡线、梯线）可能需要在仪表板里手动修正。
* **核心库是黑盒** —— 若将来核心版本改变 `blockRail` 的语义或那个 1000 毫秒常量，心跳机制必须重新评估。

---

## 参考资料

* 入口-出口（NX）进路控制 — <https://www.jmri.org/help/en/html/tools/EntryExit.shtml>
* 信号控制方式的演变 — <https://www.railengineer.co.uk/evolution-of-signalling-control/>
* 安全超距与侧防 — <https://www.railwaysignallingconcepts.in/overlap-flank-protection-railway-signalling/>
* 进路锁闭电路 — <https://www.railwaysignallingconcepts.in/route-locking-circuit-railway-signalling/>
* 进路解锁电路 — <https://www.railwaysignallingconcepts.in/railway-route-release-circuits/>
* SACEM 系统 — <https://en.wikipedia.org/wiki/SACEM_(railway_system)>
* 港铁 CBTC 信号升级 — <https://www.itsinternational.com/news/hong-kongs-mtr-upgrades-signalling-cbtc>
