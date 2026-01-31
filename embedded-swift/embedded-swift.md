# Embedded Swift 开发规范

当编写 Embedded Swift 代码时，请使用此规范进行代码审查和问题排查。

## 快速检查清单

在提交代码前，确保以下问题都已避免：

- [ ] 没有使用 `any Protocol` 类型（协议存在类型，包括 `[Protocol]` 数组）
- [ ] 没有使用 `String.count`（使用 `utf8.count` 代替）
- [ ] 没有直接使用 C 预处理器宏
- [ ] 没有重复定义 C 函数
- [ ] 子类没有重复定义父类属性
- [ ] `===` 运算符只用于类类型
- [ ] 没有使用 `as?` 运行时类型检查（使用虚函数多态代替）
- [ ] 没有使用 `weak` 引用（闭包中直接捕获 `self`）

---

## 详细规范

### 1. Protocol Existential Types（协议存在类型）

**严重程度**: 编译错误 / 运行时不工作

**问题描述**: Embedded Swift 不支持协议存在类型。这包括：
- 显式的 `any Protocol` 语法
- 隐式的协议类型数组 `[Protocol]`（等价于 `[any Protocol]`）
- 协议类型参数 `func foo(_ p: Protocol)`（等价于 `func foo(_ p: any Protocol)`）

**错误示例**:
```swift
protocol Updateable {
    func update()
}

class AnimationManager {
    // ❌ [Updateable] 实际上是 [any Updateable]，协议存在类型数组
    var values: [Updateable] = []

    func add(_ value: Updateable) {  // ❌ 协议存在类型参数
        values.append(value)
    }

    func updateAll() {
        for value in values {
            value.update()  // 运行时可能不工作
        }
    }
}
```

**解决方案**: 将 Protocol 重构为 Class 基类

```swift
// ✅ 使用基类代替协议
open class Updater {
    open func update() {
        // 默认实现或留空
    }
}

class AnimationManager {
    var values: [Updater] = []  // ✅ 类数组，不是协议存在类型

    func add(_ value: Updater) {  // ✅ 类类型参数
        values.append(value)
    }

    func updateAll() {
        for value in values {
            value.update()  // ✅ 虚函数调用，正常工作
        }
    }
}

// 泛型类可以继承基类
class AnimationValue<T>: Updater {
    override func update() {
        // 具体实现
    }
}
```

**注意事项**:
- 使用 `open class` 允许跨模块继承
- 使用 `open func` 允许子类重写方法
- 所有原本遵循协议的类型改为继承基类
- 泛型类也可以继承非泛型基类

---

### 2. String.count 字符计数

**严重程度**: 链接错误

**问题描述**: `String.count` 返回字符串的 Unicode 图形簇数量，需要调用 `_swift_stdlib_getGraphemeBreakProperty` 和 `_swift_stdlib_isInCB_Consonant` 等标准库函数，这些在 Embedded Swift 中不可用。

**错误示例**:
```swift
let text = "Hello"
let charCount = text.count  // ❌ 链接错误: undefined reference to '_swift_stdlib_getGraphemeBreakProperty'
let width = Double(text.count) * 6.0  // ❌ 同上
```

**解决方案**: 使用 `utf8.count`

```swift
let text = "Hello"
let byteCount = text.utf8.count  // ✅ 正确，返回 5
let width = Double(text.utf8.count) * 6.0  // ✅ 正确
```

**注意事项**:
- `utf8.count` 返回 UTF-8 编码的字节数
- 对于纯 ASCII 字符串，`utf8.count` 等于字符数
- 对于包含中文、emoji 等多字节字符的字符串，结果会不同
- 如果需要处理非 ASCII 字符串，需要额外的逻辑

**ASCII vs UTF-8 对比**:
```swift
"Hello".utf8.count      // 5（ASCII，每字符1字节）
"你好".utf8.count        // 6（中文，每字符3字节）
"👍".utf8.count          // 4（emoji，4字节）
```

---

### 3. C 预处理器宏

**严重程度**: 编译错误

**问题描述**: Swift 无法直接访问 C 头文件中通过 `#define` 定义的宏常量。

**错误示例**:
```c
// u8x8.h
#define U8X8_MSG_GPIO_CS    (64 + 9)   // = 73
#define U8X8_MSG_GPIO_DC    (64 + 10)  // = 74
#define U8X8_MSG_GPIO_RESET (64 + 11)  // = 75
```

```swift
// Swift 代码
func handleGPIO(_ msg: Int32) {
    switch msg {
    case U8X8_MSG_GPIO_CS:  // ❌ 找不到符号
        // ...
    }
}
```

**解决方案**: 手动计算宏值并定义为 Swift 常量

```swift
// GPIO 消息常量（从 u8x8.h 计算: U8X8_MSG_GPIO(x) = 64 + x）
private enum GPIOMessage {
    static let CS: Int32 = 73     // 64 + 9
    static let DC: Int32 = 74     // 64 + 10
    static let RESET: Int32 = 75  // 64 + 11
}

func handleGPIO(_ msg: Int32) {
    switch msg {
    case GPIOMessage.CS:  // ✅ 正确
        // ...
    }
}
```

**最佳实践**:
- 在 Swift 常量定义旁添加注释说明计算来源
- 使用 `enum` 或 `struct` 组织相关常量
- 如果宏值可能变化，考虑在 C 桥接代码中提供函数返回这些值

---

### 4. C 代码重复符号定义

**严重程度**: 链接错误

**问题描述**: 在多个 C 源文件中定义相同的函数会导致链接器报 "multiple definition" 错误。

**错误示例**:
```c
// rtos_utils.c
void delay_ms(uint32_t ms) {
    vTaskDelay(pdMS_TO_TICKS(ms));
}

// spi.c
void delay_ms(uint32_t ms) {  // ❌ 重复定义
    vTaskDelay(pdMS_TO_TICKS(ms));
}
```

**解决方案**: 只在一个源文件中定义，其他文件通过头文件声明

```c
// rtos_utils.h
#ifndef RTOS_UTILS_H
#define RTOS_UTILS_H

#include <stdint.h>

void delay_ms(uint32_t ms);  // 声明

#endif

// rtos_utils.c
#include "rtos_utils.h"

void delay_ms(uint32_t ms) {  // ✅ 唯一定义
    vTaskDelay(pdMS_TO_TICKS(ms));
}

// spi.c
#include "rtos_utils.h"  // ✅ 使用声明，不重复定义
```

---

### 5. 子类重复定义属性

**严重程度**: 编译错误（属性歧义）

**问题描述**: Swift 子类不能重新声明与父类相同名称的存储属性。

**错误示例**:
```swift
open class View {
    open var frame: Rect

    public init(frame: Rect) {
        self.frame = frame
    }
}

class ModalView: View {
    var frame: Rect  // ❌ 'frame' 使用歧义
    var cornerRadius: Double = 3

    init(frame: Rect, cornerRadius: Double) {
        self.frame = frame  // ❌
        self.cornerRadius = cornerRadius
    }
}
```

**解决方案**: 直接使用继承的属性

```swift
class ModalView: View {
    var cornerRadius: Double = 3

    init(frame: Rect, cornerRadius: Double) {
        self.cornerRadius = cornerRadius
        super.init(frame: frame)  // ✅ 使用父类初始化器
    }

    // 可以直接访问 self.frame，它来自父类
}
```

---

### 6. === 运算符类型要求

**严重程度**: 编译错误

**问题描述**: `===`（恒等运算符）只能用于引用类型（类），不能用于协议类型。

**错误示例**:
```swift
protocol Drawable { }

class Container {
    var items: [Drawable] = []

    func remove(_ item: Drawable) {
        items.removeAll { $0 === item }  // ❌ 编译错误
    }
}
```

**解决方案**: 转换为 AnyObject 后比较

```swift
class Container {
    var items: [Drawable] = []

    func remove(_ item: Drawable) {
        let targetObject = item as AnyObject
        items.removeAll { ($0 as AnyObject) === targetObject }  // ✅ 正确
    }
}
```

**或者使用基类方案（推荐）**:
```swift
open class Drawable { }

class Container {
    var items: [Drawable] = []

    func remove(_ item: Drawable) {
        items.removeAll { $0 === item }  // ✅ 类类型可以直接使用 ===
    }
}
```

---

### 7. as? 运行时类型检查

**严重程度**: 编译器崩溃 / 链接错误

**问题描述**: Embedded Swift 不支持 `as?` 运行时类型检查。对类使用 `as?` 会导致编译器崩溃（"While deserializing SIL vtable" 错误），对协议使用 `as?` 会导致链接错误（缺少 `swift_conformsToProtocol` 运行时函数）。

**错误示例**:
```swift
open class View {
    open func draw() {}
}

class ModalView: View {
    func dismiss(completion: @escaping () -> Void) {
        // 动画 dismiss
        completion()
    }
}

class Router {
    var modal: View?

    func dismissModal() {
        // ❌ 编译器崩溃: "While deserializing SIL vtable for 'ModalView'"
        if let modalView = modal as? ModalView {
            modalView.dismiss {
                self.modal = nil
            }
        }
    }
}
```

**使用协议也不行**:
```swift
protocol Dismissable {
    func dismiss(completion: @escaping () -> Void)
}

class Router {
    var modal: View?

    func dismissModal() {
        // ❌ 链接错误: undefined reference to 'swift_conformsToProtocol'
        if let dismissable = modal as? Dismissable {
            dismissable.dismiss { self.modal = nil }
        }
    }
}
```

**解决方案**: 使用虚函数多态（编译时多态）代替运行时类型检查

```swift
open class View {
    open func draw() {}

    // 在基类中定义可覆盖的方法和属性
    open var canDismiss: Bool { false }

    open func dismiss(completion: @escaping () -> Void) {
        completion()  // 默认立即完成
    }
}

class ModalView: View {
    override var canDismiss: Bool { true }

    override func dismiss(completion: @escaping () -> Void) {
        // 动画 dismiss
        completion()
    }
}

class Router {
    var modal: View?

    func dismissModal() {
        guard let modal = modal else { return }

        // ✅ 使用属性检查 + 虚函数调用，无需运行时类型检查
        if modal.canDismiss {
            modal.dismiss {
                self.modal = nil
            }
        } else {
            self.modal = nil
        }
    }
}
```

**核心原理**:
- `as?` 需要运行时类型信息（RTTI），Embedded Swift 不支持
- 虚函数调用在编译时通过 vtable 解析，无需运行时类型检查
- 通过 `canXxx` 属性模拟类型能力检查

**适用场景**:
- 需要根据子类类型执行不同行为时
- 需要检查对象是否支持某个操作时
- 多态分派场景

---

### 8. weak 引用

**严重程度**: 编译错误

**问题描述**: Embedded Swift 不支持 `weak` 属性，因为它需要运行时支持来追踪弱引用。

**错误示例**:
```swift
class MyPage: Page {
    let menu: TileMenu

    init() {
        self.menu = TileMenu()
        super.init(frame: .zero)

        // ❌ 编译错误: attribute 'weak' cannot be used in embedded Swift
        menu.onSelect = { [weak self] index in
            self?.handleSelection(index: index)
        }
    }
}
```

**解决方案**: 直接捕获 `self`，不使用 `weak`

```swift
class MyPage: Page {
    let menu: TileMenu

    init() {
        self.menu = TileMenu()
        super.init(frame: .zero)

        // ✅ 正确：直接捕获 self
        menu.onSelect = { index in
            self.handleSelection(index: index)
        }
    }
}
```

**注意事项**:
- 在 Embedded Swift 中无法使用 `weak` 或 `unowned` 引用
- 需要手动管理生命周期，确保闭包不会在对象销毁后被调用
- 对于嵌入式环境，对象通常是长生命周期的，循环引用问题较少

---

## 其他 Embedded Swift 限制

### 不支持的特性

- **反射（Reflection）**: `Mirror` 类型不可用
- **运行时类型检查**: `as?` 类型转换会导致编译器崩溃或链接错误（见规范第7条）
- **协议一致性检查**: `swift_conformsToProtocol` 运行时函数不可用
- **弱引用**: `weak` 和 `unowned` 属性不可用（见规范第8条）
- **Codable**: `JSONEncoder`/`JSONDecoder` 不可用
- **正则表达式**: `Regex` 类型不可用
- **并发**: `async/await` 和 `Actor` 不可用
- **异常处理**: `throw/catch` 支持有限

### 推荐做法

1. **使用简单数据类型**: 优先使用 `Int32`、`Double`、`Bool` 等基本类型
2. **避免复杂泛型**: 保持泛型简单，避免复杂的类型约束
3. **静态分派**: 使用 `final class` 或 `struct` 提高性能
4. **手动内存管理意识**: 虽然有 ARC，但要注意循环引用

---

## 调试技巧

### 链接错误排查

当遇到 `undefined reference to '_swift_stdlib_xxx'` 错误时：

1. 搜索代码中使用的 String 操作
2. 检查是否使用了 `String.count`、字符迭代等高级功能
3. 替换为 `utf8` 视图的等效操作

### 编译错误排查

当遇到 `cannot use a value of protocol type in embedded Swift` 错误时：

1. 找到使用协议类型的位置
2. 将协议重构为基类
3. 更新所有遵循协议的类型为继承基类

### 编译器崩溃排查

当遇到 `While deserializing SIL vtable for 'ClassName'` 错误时：

1. 搜索代码中 `as? ClassName` 的用法
2. 将运行时类型检查改为虚函数调用（见规范第7条）
3. 在基类中添加 `canXxx` 属性和对应方法
4. 子类覆盖这些方法提供具体实现

### 协议一致性链接错误排查

当遇到 `undefined reference to 'swift_conformsToProtocol'` 错误时：

1. 搜索代码中 `as? ProtocolName` 的用法
2. 同样需要改为虚函数调用方式
3. 避免任何形式的 `as?` 运行时类型检查
