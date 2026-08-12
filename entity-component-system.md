# ECS 实体组件系统（以 Bevy 为例）

> 2026-08-12 创建

## 定义：三层模型

ECS（Entity Component System，实体组件系统）是游戏领域的核心数据组织范式，把游戏世界的所有状态拆成三个正交的层：

- **实体（Entity）**：一个纯标识（一个句柄），本身无内容
- **组件（Component）**：挂在实体上的数据块（可以是一个空标记，也可以是一组数值）
- **系统（System）**：作用在组件之上的逻辑，按组件类型筛选实体、读写数据

关键：**实体是什么，完全由它身上的组件组合决定**，不来自任何类定义。

## 组件 vs 实体：数据 vs 组合

实体与组件是两个半面：

- **组件是纯粹的数据，不带行为**
- **实体是组件的一次组合**（一个句柄 + 一组组件）

```rust
#[derive(Component)]
struct Health(u32);   // 数据

#[derive(Component)]
struct Player;        // 空标记，用来标识身份

// 一个实体 = 句柄 + 组件集合
commands.spawn((Player, Health(3), Speed(3.0), Transform::from_xyz(0.0, 0.0, 0.0), Sprite::default()));
```

与面向对象的本质差异：OOP 里对象类型由类定义固定，ECS 里实体可以在**运行时通过增删组件改变身份**——同一个句柄，把 `Health` 移除、加一个 `Dead` 标记，它就从"活的"变成"尸体"，无需换对象。

## 为什么：数据驱动的好处

**数据即状态，逻辑即系统。** 这套分层带来四个结构性收益（不是锦上添花）：

1. **无状态同步**：状态躺在组件里，任何系统 `Query` 到它就读到最新值——不需要 getter/setter、不需要通知、不需要事件总线传值。"状态同步"在这里成为伪问题，正是这个根因
2. **组合远多于继承**：想要"会飞的、会攻击的、红色敌人"，给实体拼几组组件即可，不用建一堆子类。组件种类的笛卡尔积就是实体种类，类型爆炸被化解
3. **缓存局部性**：同类型组件连续存储，系统批量遍历时利用内存连续性
4. **显式依赖 = 可并行**：系统间的读写依赖由调度器显式声明，无共享状态的系统可安全并行

## 拓扑定位：网不是树

承接 [现代语言设计](modern-language-design.md) §"网形的本质，标签 = 倒排索引"：**ECS 是那条命题最干净的例证**——entity 是"唯一标识"那极的极致（一个连内容都没有的句柄），component 是"分类集"那极的极致：

| ECS 里的角色 | 拓扑中的位置 |
|:--|:--|
| entity | 纯标识（没有内容，只是一句柄） |
| component | 分类 + 数据（`Player` 标记是分类，`Health(3)` 兼顾数据） |

系统按组件筛选实体（`Query`），本质就是标签倒排索引求交集。一个实体可同时属于 `Player`/`Health`/`Transform` 多组组件，组件间无依赖、无收敛根——正是现代语言设计文档说的"多根灌木草地"，而非单根树。这里的 `component` 与文档里的 `trait`/`tag`/`label` 是**同一极**的东西：都是"挂在标识上的分类集"，只是 ECS 的组件比纯标签多承载了数据。

**但这不等于"ECS 纯网"。** Bevy 内部也有树：`Parent`/`Child` 场景图（用于变换传播、渲染顺序、级联销毁）。区别两根轴：

- **数据访问拓扑**（Query 按组件筛选）＝ 网
- **数据关系拓扑**（Parent/Child 层级）＝ 树

渲染、物理、音频只是**装在 ECS 之上的领域消费者**，不改变其网形本质。这与"树和网各司其职"的姿态一致：网管复用与逻辑，树管寻址与序。

## Bevy 实践（0.19）

以下 API 全部经 Bevy 0.19.0 实测编译通过。

### 生命周期

生成、销毁、改类型都经 `Commands`（缓冲延迟执行，避免迭代中修改崩溃）：

```rust
commands.spawn((Player, Health(3), ...));   // 生成
commands.entity(e).despawn();               // 销毁（延迟）
commands.entity(e).insert(Dead);            // 运行时改身份
commands.entity(e).remove::<Health>();
```

### 状态共享与消息

- **本机状态**：`Query` 直读组件，天然一致，无需同步
- **跨系统一次性消息**：Bevy 0.19 用 `Message` 取代旧 `Event`

```rust
#[derive(Message)]
struct HitEvent { target: Entity, damage: u32 }

fn a(mut w: MessageWriter<HitEvent>) { w.write(HitEvent { target: e, damage: 1 }); }
fn b(mut r: MessageReader<HitEvent>) { for ev in r.read() { /* 消费 */ } }
```

### 反馈即组件

把"闪白""飘字"做成组件，受实体生命周期管理，可叠加、可被系统统一处理：

```rust
#[derive(Component)]
struct FlashColor { timer: Timer }

// 命中时插入，一个系统读它、调亮度、结束后移除
commands.entity(e).insert(FlashColor { timer: Timer::from_seconds(0.15, TimerMode::Once) });
```

## 交叉引用

- [现代语言设计](modern-language-design.md)：ECS 的 `component` 与 `trait` 同属"网形（标签 = 倒排索引）"拓扑
- [AI 辅助视频制作与游戏引擎](ai-creative-production-workflow.md)：fabricario 管线中 Bevy 作为交互式出口的定位