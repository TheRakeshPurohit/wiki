# ECS 实体组件系统（以 Bevy 为例）

> 2026-08-12 创建

## 范式差异：ECS 相对 OO 是标签，不是树

ECS 与 OO 的分界线不在"有没有对象"，而在**数据与逻辑如何组织**。OO 用类定义把数据封装进对象、逻辑（方法）挂在对象上，身份由类决定；ECS 把数据拆成扁平的组件、逻辑（系统）与数据分离，身份由组件组合决定。结构上，这是一次**从树到标签**的迁移（完整论证见 [现代语言设计](modern-language-design.md)）：

- **OO（class 继承）＝ 树**：class 体系的结构特性在**边**——只跨级连接（父→子），同层不连接，一个类只能处在唯一继承链上，层层向上收敛到唯一根类（Java 的 Object）。class 是**命名**（标识主体"它是谁"），单一且唯一，身份来自它在这棵树上的命名位置。
- **ECS ＝ 标签**：entity 是纯标识（一句柄），component 是标签集合——**一个物体，很多标签**，每个标签描述一个属性/方面，可同时存在（`Player`、`Health`、`Transform`）、自由组合。组合数量是标签种类的**乘积**（一物多标签，即多维）。

这个区分的本质是 **标签 = 倒排索引**：把"实体"和"分类"解耦——实体用唯一标识定位，分类用标签集合表达，查询走倒排索引求交集。ECS 的 `component` 与 trait/tag/label 是同一类东西：都是"挂在标识上的标签"；区别只是 ECS 的组件比纯标签多承载了数据（`Health(3)`、`Transform`），是"标签 + 数据"。

| ECS 里的角色 | 结构中的位置 |
|:--|:--|
| entity | 纯标识（没有内容，只是一句柄） |
| component | 标签 + 数据（`Player` 是纯标签，`Health(3)` 兼顾数据） |

系统按组件筛选实体（`Query`），本质就是倒排索引求交集——一个实体可同时带 `Player`/`Health`/`Transform` 多组标签，正是"一物多标签"，而非树。标签的组合方式也对应**参数多态**（实现接口，如 Rust 的 `T: A + B`），而非 OO 的**包含多态**（子类型/里氏替换）——这是又一个与 OO 在范式上的分岔。


## 定义：三层模型

ECS 把游戏世界的所有状态拆成三个正交的层：

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

## 为什么：数据与逻辑分离的收益

**数据与逻辑分离**是 ECS 收益的结构前提，不是"数据驱动"。"数据驱动"太宽泛——OO 相对命令式/过程式同样是围绕数据组织的（对象就是数据 + 方法），所以它不能把 ECS 和 OO 分开。真正的分野在**逻辑与数据的关系**：

- **OO**：逻辑（方法）绑在数据（对象）上，数据拥有自己的操作
- **ECS**：逻辑（系统）从数据（组件）剥离，系统作用于数据，数据只是被操作的状态

这个分离带来四个结构性收益（不是锦上添花）：

1. **无状态同步**：状态躺在组件里，任何系统 `Query` 到它就读到最新值——不需要 getter/setter、不需要通知、不需要事件总线传值。"状态同步"在这里成为伪问题，正是这个根因
2. **组合远多于继承**：想要"会飞的、会攻击的、红色敌人"，给实体拼几组组件即可，不用建一堆子类。组件种类的笛卡尔积就是实体种类，类型爆炸被化解
3. **缓存局部性**：同类型组件连续存储，系统批量遍历时利用内存连续性
4. **显式依赖 = 可并行**：系统间的读写依赖由调度器显式声明，无共享状态的系统可安全并行

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

- [现代语言设计](modern-language-design.md)：ECS 的 `component` 与 `trait` 同属"标签（一物多标签 = 倒排索引）"结构；组件组合对应参数多态而非 OO 的包含多态
- [AI 辅助视频制作与游戏引擎](ai-creative-production-workflow.md)：fabricario 管线中 Bevy 作为交互式出口的定位