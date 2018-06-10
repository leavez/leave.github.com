---
layout:     post
title:      使用泛型构造一个聪明的 JSON 解析库
category:   []
tags:       [技术]
published:  True
date:       2018-6-10

---

## "Don't Repeat Yourself"

在 Codable 出现之前，Swift 中把一个 JSON 字典转换成对象的时候，需要手动转换类型：

```swift
self.address = dictionary["address"] as? String
```

即使使用了 `SwiftyJSON` 这样的库，只是方便了书写方式，仍然需要手动指定类型。

其实 `self.address` 已经在声明的时候指明类型了，提取 JSON 的时候指定类型，在信息上讲完全是冗余的。利用泛型，我们完全可以省略这一步骤：

```swift
extension Dictionary {
    func mapValue<T>(key: Key) -> T? {
        return self[key] as? T
    }  
}

// let address: String
self.address = dictionary.mapValue("address")  //  ?? "defaultValue"
```

这里 `mapValue` 中的泛型 T 是由返回值类型决定的，它随着赋值对象类型的不同而不同。这样，我们就把手动的 cast 省略掉了，更专注于映射关系。

## 更复杂的类型

上面的实现中，取出的 raw value 只做了简单的 cast，对于复杂一些的类型（比如 custom object） 就不适用了。让我们继续改进。

对于 custom object，我们可以指定它符合一个 protocol，这个 protocol 中有初始化方法。

```swift
protocol JSONInitializable {
    init?(dictionary: [String: Any]) 
}
```

这样我们可以扩展上面的代码

```swift
extension Dictionary {
    func mapValue<T>(key: Key) -> T? {
        return self[key] as? T
    } 
    
    func mapValue<T: JSONInitializable>(key: Key) -> T? {
        if let value = self[key] as? [String: Any] {
            return T.init(dictionary: value)
        }
        return nil
    } 
}
```

对于符合 JSONInitializable 的类型，编译器会自动使用第二个方法。

虽然只有几行代码，我们的解析库已经初见雏形。完整的使用方法如下：

```swift
class Person: JSONInitializable {
    let name: String
	let contect: Contact?
    
    required init?(dictionary: [String: Any]) {
     	name = dictionary.mapValue("phone") // ?? "" 
        contact = dictionary.mapValue("contact") 
    }
}

struct Contact: JSONInitializable {
    let phone: String
    let email: String
    let address: String
    
    init?(dictionary: [String: Any]) {
     	phone = dictionary.mapValue("phone")   
        //...
    }
}
```



## "Build the Whole World"

上面的实现仍不能方便的支持所有情况，比如容器类的类型（Array，Dictionary）。这个问题很好处理，我们只需要对容器类型本身实现 `JSONInitializable` 协议，他们无穷的组合方式，即可被我们收入囊中。

但在此之前，我们先优化一下 JSON 的数据结构。由于 JSON 种不只有字典，还有数组，所以使用 `[String:Any]` 来代表 JSON 是不能使用与 Array 的。为了简单，我们使用 `Any` 代表 JSON （这样可以与 JSONSerialization  解析出的 JSON 数据结构无缝衔接）。但由于 `Any` 过于宽泛，并且无法扩展，我们加一小层包装：

```swift
struct JSON {
    private value: Any
    
    subscript(key: String) -> Any? {
        return value[key]
    }
    var rootValue: Any {
        return value
    }
}

// JSONInitializable 对应的更改
protocol JSONInitializable {
    init?(JSON: JSON) 
}

// mapValue 方法声明在 JSON 数据结构里
// 方法实现没有实质改变
extension JSON {
    func mapValue<T>(key: Key) -> T? { ... }
    func mapValue<T: JSONInitializable>(key: Key) -> T? { ... }
}
```

这样，我们就可以对 `Array` 进行扩展了：

```swift
extension Array: JSONInitializable where Element: JSONInitializable {

    public init?(JSON: JSON) {
        guard let values = JSON.rootValue as? [Any]) {
            return nil
        }
        self = values.map {
            return Element.init(JSON: JSON(value:$0))
        }
    }
}
```

同理可以对其余容器型类型进行扩展。值得注意的是，Optional 也是一种容器型类型。

至此，无论是 `[Int]`，还是 `[[Int]]` ，还是 `[String:[Person]]?`  ... 无穷无尽的组合都可以试用。

我们的 JSON 解析库也已完成。它虽然只有很少的代码，但功能却很强大：

- 只需关注映射关系，不需要手动指定类型
- 不只兼容基本类型，也适用于自定义类型，和其各种组合

## 更多

上面的代码为了演示，只追求代码简单，仍有很多方面可以丰富：

- Error 处理：在缺少 key，或者数据格式不符的情况，上面只做了简单的返回 nil。这里可以把 initializer 设为 `throws`，错误的时候 throw error，让上层决定怎么处理。
- nested key：可以方便地扩展 JSON 的 subscript 方法，支持 key path。
- 默认的自定义转换：可以通过扩展基本类型支持 JSONInitializable，使得兼容从其他数据类型格式初始化，比如 String to Int，Double to Date 等。
- 省略映射关系？：虽然 swift 类通过反射也可以获得 property 的名称，但 let property 编译器会要求在初始化中手动赋值，所以仍不能实现自动映射。`var` 和 nullable type ？ 不，那不是 swift。

对以上内容的一个生产级别的实现存在这里：[Github/Mappable](https://github.com/leavez/Mappable)，欢迎大家使用。

### 关于 Codable

相比于 Swift 4 中的 Codable，这个库还是显得原始一些，因为前者把映射关系也省略了，直接使用 property 的内容。但 Codable 在 Swift 4 中还是有个一个巨大的局限：不支持子类的初始化。当父类已经实现 Codable 的情况下，子类的自动映射完全失效，必须手动处理，实现非常麻烦。虽然 Swift 倡导使用 struct 和组合的方式来声明 model，但在开发中，类继承这种传统的 OOP 方式仍有很多使用的空间。这也给这种原始的库留下了生存空间。

但手动指定映射关系一定是不好的吗？我觉得不是。在 ObjC 时代的大中型项目中使用过一个自动映射的库，那个时候我仍然坚持手动显式地指明映射关系。这样写代码更清晰，可以清楚地知道哪些是网络的数据，哪些是本地数据。而且强制要求手动指定映射关系，可以给（写代码随意的）程序员一个机会，使得改进 property 的命名，而不是完全和 JSON 里的 key 一样，而后者常常不适用于客户端。

