# Landlayer Specification
# landlayer

**A Streamlined, Programmable Wayland Stacking Compositor**

| 属性                   | 定义               |
| -------------------- | ------------------- |
| Version              | 0.1                 |
| Status               | Initial Design      |
| Language             | Rust                |
| Compositor Framework | Smithay             |
| Configuration        | Embedded Lua        |
| IPC                  | Unix Domain Socket  |
| Window Model         | Floating / Stacking |
| Target               | Linux / Wayland     |

## 1. Project Vision

### 1.1 Definition

Landlayer 是一个以自由浮动为基础，以几何操作为核心，以可编程性为扩展方式的 Wayland Compositor。

它不追求丰富的桌面功能、复杂的窗口布局或视觉效果，而是致力于将有限的窗口管理功能做得完整、统一、精确。

### 1.2 Design Principles

1. Floating First：自由浮动是基本窗口组织方式，不强制采用平铺布局。
2. Geometry First：窗口几何操作是核心能力，而非附属功能。
3. Consistent Stacking：窗口层级具有明确、统一的语义。
4. Keyboard Driven：键盘是主要操作方式，支持连续操作。
5. Mouse Supported：鼠标操作完整，不因键盘优先而受到限制。
6. Small Core：核心功能保持精简，避免不必要的子系统。
7. Programmable：通过 Lua 和 IPC 提供灵活的扩展能力。
8. Reuse Infrastructure：充分利用 Smithay 和现有 Wayland 生态，不重复实现底层基础设施。

### 1.3 Non-goals

Landlayer 不打算实现：

- Tiling layout system
- Tabbed windows / Window grouping
- Desktop environment
- Panel / Dock / Launcher
- Wallpaper management
- Notification daemon
- Complex animations
- Complex window decorations
- General-purpose plugin framework

这些功能由外部程序或其他系统组件承担。

## 2. Window Management Model

### 2.1 Floating Windows

所有普通顶层窗口均采用自由浮动模式。

每个窗口独立拥有：

- Position
- Size
- Stacking layer
- Workspace membership
- Visibility state
- Focus state
- Window rules

窗口之间可以任意重叠，不存在自动平铺或强制排列关系。

### 2.2 Three-layer Stacking

Landlayer 采用三个互斥的窗口层级：

ABOVE

Always Above

NORMAL

Default Window Layer

BELOW

Always Below

定义：

```
enum WindowLayer {
    Below,
    Normal,
    Above,
}
```

规则：

- Normal 是新窗口的默认层级。
- 每一层内部具有独立的 stacking 顺序。
- Raise / Lower 只改变当前层内部顺序。
- 改变层级时，窗口进入目标层的顶部。
- Focus 不自动改变层级。
- 三层之间不允许发生违反层级顺序的覆盖。

系统级 surface、Session Lock、必要的 Popup 和输入法界面应有独立的协议优先级，不能简单套用普通窗口的三层模型。

### 2.3 Window States

以下状态彼此独立：

| 状态         | 含义                     |
| ---------- | ---------------------- |
| Layer      | Below / Normal / Above |
| Sticky     | 是否跨 Workspace 显示       |
| Maximized  | 最大化                    |
| Fullscreen | 全屏                     |
| Minimized  | 最小化                    |
| Focused    | 当前焦点                   |

不应把这些状态混合成一个复杂的窗口模式枚举。

### 2.4 Workspace Membership

每个普通窗口属于一个 Workspace。

窗口可以：

- Move to another Workspace
- Move to another Output
- Toggle Sticky
- Restore to previous Workspace

Sticky 窗口在所属 Output 的所有 Workspace 上可见。

Sticky 不意味着跨越所有 Output。

### 2.5 Window Lifecycle

必须支持：

- Create
- Map
- Unmap
- Focus
- Raise / Lower
- Minimize / Restore
- Close
- Destroy

窗口关闭、取消映射、客户端断开连接等情况均应正确处理。

## 3. Workspace and Output

### 3.1 Workspace Model

每个 Output 拥有自己的 Workspace 集合。

例如：

```
Output A
  Workspace 1
  Workspace 2
  Workspace 3

Output B
  Workspace 1
  Workspace 2
  Workspace 3
```

每个 Output 独立切换 Workspace。

Workspace 使用全局唯一 ID，而不是仅依靠序号识别。

### 3.2 Workspace Operations

必须支持：

- Switch Workspace
- Next / Previous Workspace
- Move Window to Workspace
- Move Window to Another Output
- Create / Remove Workspace
- Sticky / Unsticky

### 3.3 Output Operations

必须支持：

- Multiple Outputs
- Output Detection
- Output Hotplug
- Output Removal
- Output Configuration Changes
- Resolution Changes
- Scale Changes

Output 被移除时，其窗口必须迁移到仍然存在的 Output。

### 3.4 Coordinate System

窗口几何使用逻辑坐标（logical coordinates）。

每个 Workspace 定义自己的工作区域：

```
struct WorkspaceGeometry {
    x: f64,
    y: f64,
    width: f64,
    height: f64,
}
```

几何操作、吸附和空白区域计算均以 Workspace 工作区域为参照，而不是直接使用物理屏幕像素。

工作区域应考虑 Layer Shell 等外部 surface 对可用空间的占用。

## 4. Geometry Engine

Landlayer 的核心特色

Geometry Engine 应作为独立的 Rust 模块设计，不依赖 Smithay 的具体实现。

### 4.1 Basic Geometry

窗口表示为矩形：

\\[ R=(x,y,w,h) \\]

其中：

- \\(x,y\\)：左上角坐标。
- \\(w,h\\)：宽度和高度。

基本结构：

```
struct Rect {
    x: f64,
    y: f64,
    width: f64,
    height: f64,
}
```

Workspace 也用 Rect 表示。

所有几何操作最终都转化为 Rect 的变化。

### 4.2 Edge Operations

四条边均支持独立操作：

| Edge   | Move | Resize / Expand / Shrink |
| ------ | ---- | ------------------------ |
| Left   | 左右移动 | 调整左边界                    |
| Right  | 左右移动 | 调整右边界                    |
| Top    | 上下移动 | 调整上边界                    |
| Bottom | 上下移动 | 调整下边界                    |

其中 Move 改变窗口位置而不改变尺寸；Resize 改变指定边缘而保持相对的另一边缘不动。

### 4.3 Corner Operations

四个角均支持二维操作：

- Top-left
- Top-right
- Bottom-left
- Bottom-right

每个角既可以向 Workspace 对应角移动，也可以背离对应角移动。

例如：

| 操作               | 窗口位置 | 窗口尺寸  |
| ---------------- | ---- | ----- |
| Top-left 向左上     | 左上移动 | 宽、高增加 |
| Top-left 向右下     | 右下移动 | 宽、高减少 |
| Bottom-right 向右下 | 右下移动 | 宽、高增加 |
| Bottom-right 向左上 | 左上移动 | 宽、高减少 |

其他两个角同理。

所有操作支持独立的 X/Y 增量，也支持比例增量。

### 4.4 Geometry Constraints

必须处理：

- Minimum Size
- Maximum Size
- Aspect Ratio
- Workspace Boundaries
- Client Size Constraints

客户端的尺寸约束应受到尊重，避免窗口出现不合法的尺寸。

同时应区分：

- Requested geometry：用户希望得到的几何。
- Actual geometry：客户端实际接受并呈现的几何。

Wayland 窗口的 configure/ack 生命周期需要纳入实现，不能假定发送 configure 后窗口就立即改变了尺寸。

### 4.5 Workspace-relative Geometry

支持相对于 Workspace 尺寸的操作。

例如：

```
Move left  by 5% workspace width
Move down  by 5% workspace height

Resize left  by 5% workspace width
Resize bottom by 5% workspace height
```

也支持绝对逻辑像素增量。

这样用户可以通过 Lua 定义不同精度和速度的操作。

### 4.6 Standard Geometry Operations

提供以下基本操作：

- Move
- Resize
- Expand
- Shrink
- Center
- Maximize
- Fill Workspace
- Restore Previous Geometry

Maximize、Fullscreen 和 Fill Workspace 必须有明确区别：

| 操作             | 行为                     |
| -------------- | ---------------------- |
| Maximize       | 填充可用工作区域，保留窗口装饰        |
| Fullscreen     | 占据整个 Output，不显示普通窗口装饰  |
| Fill Workspace | 按 Workspace 工作区域设置窗口几何 |

其中 Maximize 和 Fill Workspace 在第一版可以使用相同的几何计算，但保留不同的语义和 API。

## 5. Edge Snap and Free Area Filling

### 5.1 Edge Snap

Edge Snap 是 Geometry Engine 的一等功能。

支持的吸附目标：

1. Workspace 四条边。
2. 其他普通窗口的四条边。
3. 可用工作区域的边界。
4. 其他由系统定义的几何参考线。

吸附应当支持：

- Mouse dragging。
- Keyboard movement。
- Keyboard resizing。

### 5.2 Snap Behavior

基本原则：

- 当目标边缘进入指定吸附距离时，自动吸附。
- 离开吸附范围后，可以自由移动。
- 不改变窗口的 Stacking layer。
- 不强制改变窗口的 Workspace membership。
- 不自动将窗口转化为 Tiling window。

吸附距离可以配置：

```
geometry = {
    snap_distance = 12,
}
```

建议使用逻辑像素作为默认单位。

### 5.3 Snap Priority

当存在多个候选吸附目标时，应采用确定性的优先级规则。

建议优先级：

1. Workspace 边界。
2. 其他窗口边界。
3. 其他参考线。

同一优先级内选择距离最近的目标。

对于多个窗口同时形成的候选目标，应避免吸附结果在相邻帧之间来回跳动。需要适当的 hysteresis（吸附滞回）机制。

### 5.4 Fill Free Area

这是区别于普通 Edge Snap 的更高级几何功能。

目标是寻找 Workspace 中可以容纳窗口的空白矩形，并将窗口调整到该区域。

例如：

```
+-------------------------------+
|                 |             |
|     Window A    |             |
|                 |  Free Area  |
|-----------------|             |
|                 |             |
|     Window B    |             |
|                 |             |
+-----------------+-------------+
```

如果新窗口或当前窗口可以填充右侧空白区域，就将它调整到相应位置和尺寸。

需要明确：

- 只考虑指定 Workspace 中可见的普通窗口。
- Below / Normal / Above 三层窗口的参与规则需要区分。
- Sticky 窗口应在当前 Workspace 中参与空间占用计算。
- 不移动其他窗口。
- 不强制改变其他窗口尺寸。
- 不保证找到全局最优解。

建议第一版采用基于矩形分割的确定性算法，生成候选空白矩形，再根据面积、尺寸适配度和位置选择结果。

不需要引入复杂的通用装箱算法。

### 5.5 Relationship Between Snap and Fill

两者应共享几何计算基础，但属于不同操作：

- Snap：调整窗口，使其边缘与目标边缘对齐。
- Fill Free Area：寻找空白矩形，并使窗口填充该区域。

不要把它们混为一个操作。

## 6. Input and Interaction

### 6.1 Keyboard-first

键盘是 Landlayer 的主要窗口操作方式。

必须支持：

- Arbitrary key bindings
- Modifier combinations
- Key repeat
- Continuous movement
- Continuous resizing
- Workspace switching
- Window cycling

### 6.2 Key Repeat

Compositor 自身的快捷键处理必须支持按键重复。

可配置：

- Repeat delay
- Repeat rate
- Per-binding repeat enable/disable

Repeat 主要用于：

- Move
- Resize
- Expand / Shrink
- Workspace-relative geometry operations

Toggle 类操作默认不重复执行。

### 6.3 Acceleration

可以支持按住快捷键一段时间后加速。

但第一版不必引入复杂的速度曲线。

建议先提供：

- Normal step
- Fast step
- Configurable repeat rate

加速机制作为后续增强。

### 6.4 Mouse Interaction

必须支持：

- Click to focus
- Raise on click
- Mouse move
- Mouse resize
- Drag move
- Drag resize
- Edge snapping during drag

Focus follows mouse 可以作为可选配置。

## 7. Window Decorations and Theme

### 7.1 Decoration Philosophy

装饰保持极简。

不实现复杂的视觉效果和动画系统。

### 7.2 Built-in Theme

支持：

- Active window color
- Inactive window color
- Border width
- Border color
- Optional titlebar
- Title font
- Menu background
- Menu foreground
- Selection color

标题栏可以关闭。

窗口边框应保留独立配置，以便在无标题栏时仍然具有清晰的焦点提示。

### 7.3 Inner Menu

内置简单菜单。

至少包括：

- Root Menu
- Window Menu
- Workspace Menu

Window Menu 包括：

- Raise / Lower
- Always Above / Normal / Always Below
- Sticky / Unsticky
- Maximize / Restore
- Fullscreen
- Move to Workspace
- Close

菜单内容允许 Lua 定义，菜单绘制和交互由 Rust 核心负责。

## 8. Configuration: Embedded Lua

### 8.1 Design

采用嵌入式 Lua，而不是自定义配置语言。

Lua 负责：

- Configuration
- Key bindings
- Mouse bindings
- Window rules
- Menus
- Command composition
- Custom geometry operations

Rust 负责所有核心状态与底层实现。

### 8.2 API Principles

Lua 不应直接修改内部 Rust 数据结构。

所有操作通过稳定的公开 API 完成。

例如：

```
bind("Super+h", function()
    window.move(-20, 0)
end)

bind("Super+Shift+h", function()
    window.resize_edge("left", -20)
end)

bind("Super+Control+h", function()
    window.move_corner("top_left", -20, 0)
end)

bind("Super+1", function()
    workspace.switch(1)
end)
```

这里仅展示 API 方向，具体函数签名留待 API 设计阶段确定。

### 8.3 Window Rules

支持按照应用标识和窗口属性定义规则：

```
rule {
    app_id = "pavucontrol",
    floating = true,
    width = 600,
    height = 400,
    center = true,
}

rule {
    app_id = "mpv",
    layer = "above",
}
```

### 8.4 Runtime

要求：

- 配置文件加载。
- 配置错误报告。
- 配置重载。
- API 版本管理。
- Lua 错误隔离。
- 不允许脚本绕过核心状态管理。

建议初期采用单一 Lua Runtime，不急于支持多脚本隔离或复杂插件生命周期。

## 9. IPC

### 9.1 Transport

使用 Unix Domain Socket。

### 9.2 Protocol

建议使用简单的 JSON 命令协议，必要时采用 JSON-RPC 风格的 request/response。

### 9.3 Command API

IPC 与 Lua 尽量共享同一套 Command API。

例如：

```
{"cmd":"window.move","dx":20,"dy":0}
```

```
{"cmd":"window.set_layer","layer":"above"}
```

```
{"cmd":"workspace.switch","id":3}
```

```
{"cmd":"window.toggle_sticky"}
```

### 9.4 Event Subscription

至少提供：

- window_created
- window_closed
- window_focused
- window_layer_changed
- window_geometry_changed
- workspace_changed
- output_changed

事件应包含必要的对象 ID 和状态信息。

### 9.5 Security

IPC Socket 应具有合理的文件权限，只允许当前用户访问。

不应允许不可信客户端任意访问 compositor 内部对象或执行不受限制的操作。

## 10. Wayland Protocol and System Integration

Landlayer 应优先使用 Smithay 已有的协议和基础设施实现。

### Required

- Wayland Core
- XDG Shell
- XDG Decoration
- Layer Shell
- Data Device / Clipboard
- Primary Selection
- Seat / Keyboard / Pointer
- Output Management
- Session Lock
- Screencopy
- XWayland

### Important

- Input Method
- Text Input
- Relative Pointer
- Pointer Constraints
- Tablet support
- Fractional Scaling
- Presentation Time

其中，Required 是达到日常可用版本必须具备的能力；Important 可以根据实现成熟度分阶段支持。

协议的具体支持方式、版本及扩展应在 Smithay 选型和实现阶段逐项确认，不能仅因为库中存在某个协议模块就假定完整支持。

## 11. Technical Architecture

### 11.1 Technology Stack

| Component            | Technology         |
| -------------------- | ------------------ |
| Language             | Rust               |
| Compositor Framework | Smithay            |
| Lua Runtime          | mlua 或同类成熟方案       |
| IPC                  | Unix Domain Socket |
| Serialization        | serde              |
| Geometry             | Native Rust        |
| Testing              | cargo test         |
| Build                | Cargo              |

Lua 运行时的具体选择暂不冻结，需要比较维护情况、Lua 版本、FFI 安全性和异步集成方式。

### 11.2 Architectural Layers

User Configuration

Lua / Key Bindings / Rules / Menus

Command and Event API

Shared API for Lua and IPC

Window Management Core

Window / Workspace / Output / Stacking / Focus

Geometry Engine

Rect / Edge / Corner / Snap / Free Area

Smithay Integration

Wayland / Input / Rendering / Backend / Protocols

需要强调：Geometry Engine 与 Window Management Core 实际上是并列协作的逻辑模块，而不是严格的单向依赖。上图只是展示功能层次。

### 11.3 Module Structure

建议初期采用单一 Cargo package，内部按模块划分；待模块边界稳定后，再考虑拆分 workspace crates。

```
landlayer/
├── Cargo.toml
├── Cargo.lock
├── src/
│   ├── main.rs
│   ├── compositor/
│   ├── backend/
│   ├── protocol/
│   ├── window/
│   ├── workspace/
│   ├── output/
│   ├── stacking/
│   ├── geometry/
│   ├── input/
│   ├── decoration/
│   ├── menu/
│   ├── config/
│   ├── ipc/
│   └── rules/
├── tests/
│   ├── geometry/
│   ├── stacking/
│   └── integration/
├── examples/
│   └── config.lua
└── docs/
    ├── DESIGN.md
    ├── GEOMETRY.md
    └── API.md
```

### 11.4 Core Independence

这是架构中非常重要的约束：

- Geometry 不依赖 Smithay。
- Stacking 不依赖具体渲染实现。
- Workspace 不依赖物理 Output 坐标。
- Lua 不直接操作内部状态。
- IPC 与 Lua 共享 Command API。
- 所有窗口状态变更通过统一的核心操作完成。

这样未来升级 Smithay 或替换底层实现时，不需要重写整个窗口管理系统。

## 12. Reliability and Testing

Landlayer 不应只以能否启动作为完成标准。

### 12.1 Unit Tests

重点测试：

- Geometry calculations
- Edge and Corner operations
- Snap calculations
- Free Area detection
- Stacking order
- Workspace membership
- Window state transitions
- Geometry constraints

### 12.2 Integration Tests

测试：

- 多窗口创建与关闭。
- Focus 和 Stacking。
- 多 Workspace 切换。
- 多 Output 操作。
- 窗口跨屏移动。
- Sticky 行为。
- Maximize / Fullscreen。
- 客户端异常退出。
- Output 热插拔。

### 12.3 Operational Reliability

必须考虑：

- 崩溃恢复。
- 无效 Lua 配置。
- 客户端断开。
- 输出设备变化。
- Session Lock。
- 输入设备变化。
- 长时间运行的稳定性。

## 13. Development Priorities

建议采用以下开发阶段：

| 阶段 | 内容                                  | 交付目标          |
| -- | ----------------------------------- | ------------- |
| P0 | Rust / Smithay 技术验证                 | 最小 Compositor |
| P1 | Window Lifecycle / Focus / Stacking | 基本浮动窗口        |
| P2 | Geometry Engine                     | 四边、四角操作       |
| P3 | Edge Snap / Fill Free Area          | 核心特色完成        |
| P4 | Workspace / Multi-output            | 完整窗口管理        |
| P5 | Theme / Menu / Mouse                | 完整交互          |
| P6 | Lua / IPC / Rules                   | 可编程系统         |
| P7 | XWayland / 协议兼容 / 稳定性               | 日常使用版本        |

这里的阶段不是严格的瀑布式开发。协议基础设施、输出管理和客户端兼容性需要从早期就逐步纳入，不能等到最后才开始处理。

## 14. Design Decisions and Open Questions

为了避免后续设计反复，明确区分已经确定的决策与尚待讨论的事项。

### 已确定

Design decisions

- Rust + Smithay
- Floating / Stacking model
- Three-layer stacking
- No Tab / Window Grouping
- Independent Workspace per Output
- Keyboard-first interaction
- Key Repeat
- Four-edge operations
- Four-corner operations
- Workspace-relative geometry
- Edge Snap
- Fill Free Area
- Minimal built-in Theme
- Minimal built-in Menu
- Embedded Lua
- IPC
- No complex animation system

### 尚待确定

Open questions

1. Geometry operation semantics：Expand / Shrink 与 Move / Resize 的最终 API 如何组织。
2. Snap algorithm：多目标冲突、吸附滞回、键盘操作吸附的具体行为。
3. Free Area algorithm：候选矩形生成、优先级和选择策略。
4. Stacking details：Popup、Transient、Modal window 与三层普通窗口之间的具体关系。
5. Workspace lifecycle：动态创建、删除、编号与持久 ID 的关系。
6. Fullscreen policy：同一 Output 是否允许多个 Fullscreen 窗口，以及其对其他窗口的遮挡规则。
7. Focus policy：Click-to-focus、Focus-follows-mouse、Raise-on-focus 的默认组合。
8. Lua runtime：具体版本、执行模型、错误隔离。
9. IPC protocol：JSON 命令格式、事件订阅与版本兼容策略。
10. Window rules：规则匹配顺序，以及运行时规则修改的行为。

这些问题不妨在相应模块开始实现之前再冻结，但不必现在过早决定。

# 最后：Landlayer 的设计边界

最后再强调一下整个设计中最重要的三件事。

第一，三层 Stacking 是窗口组织的基础。

Below / Normal / Above 不只是两个额外的窗口属性，而是整个窗口空间的三个基本层级。每一层内部再独立管理窗口顺序。

第二，Geometry 是 Landlayer 真正值得投入设计精力的地方。

四边、四角、Workspace 相对操作、Edge Snap、Fill Free Area，应该形成一套统一的几何系统，而不是一组互不相关的快捷键功能。

第三，Lua 和 IPC 不应侵入核心。

Rust 核心提供稳定、明确的状态和操作接口。Lua 与外部程序通过同一套接口使用这些能力。

这三点决定了 Landlayer 的整体架构。

**这份文档是 v0.1 的设计基线，后续的修改都以明确的设计决策和版本记录来进行。 **
