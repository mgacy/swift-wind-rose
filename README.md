# WindRose

WindRose provides types for Tailwind utility classes. While it was originally developed for use with [Plot](https://github.com/JohnSundell/Plot), it should be usable with any HTML DSL library.

## Requirements

Swift 5.10 toolchain with Swift Package Manager, macOS 13.0.

## Installation

Add the following dependency clauses to your Package.swift:

```swift
dependencies: [
    .package(url: "https://github.com/mgacy/swift-wind-rose.git", from: "0.5.0")
],
targets: [
    .target(
        name: "MyTarget",
        dependencies: [
            ...
            .product(name: "WindRose", package: "swift-wind-rose")
        ])
]
```


## Design

The core of the library is a simple `UtilityClass` struct that is generic over a phantom `Property` type. It has a `name` and a collection of `Modifier`s representing Tailwind's variant modifiers.

```swift
public struct Modifier {
    public let name: String
}

public struct UtilityClass<Property> {
    let name: String
    let modifiers: [Modifier]

    var className: String {
        modifiers.isEmpty ? name : "\(modifiers.map(\.name).joined(separator: ":")):\(name)"
    }
}
```

WindRose defines a type for every group of Tailwind classes which it uses as a same-type constraint for declarations of each group's classes. The [display](https://tailwindcss.com/docs/display) classes, for example, are defined as follows:

```swift
public enum Display {}

public extension UtilityClass where Property == Display {
    /// CSS: `display: block;`
    static let block = UtilityClass(name: "block")

    /// CSS: `display: inline-block;`
    static let inlineBlock = UtilityClass(name: "inline-block")
    
    ...
}
```

Various protocols like [`ColorClass`](Sources/WindRoseCore/Protocols/ColorClass.swift) are used to generalize utility class declarations, though there remains significant room for improvement. WindRose does not yet support [custom Tailwind configurations](https://tailwindcss.com/docs/configuration).


### Modifiers

Tailwind's [variant modifiers](https://tailwindcss.com/docs/hover-focus-and-other-states) are supported via corresponding `UtilityClass` methods that add a `Modifier` to a given instance: 

```swift
public struct Modifier {
    public let name: String
}

extension Modifier {
    /// CSS: `&:hover`
    static let hover = Modifier("hover")
    
    ...
}

extension UtilityClass {
    static func modifying(_ utilityClass: Self, with modifier: Modifier) -> Self {
        UtilityClass(
            utilityClass.name,
            modifiers: utilityClass.modifiers.prepending(modifier))
    }
}

public extension UtilityClass {
    static func hover(_ utilityClass: UtilityClass) -> Self {
        modifying(utilityClass, with: .hover)
    }
    
    ...
}
```

As in Tailwind, modifiers can be stacked to target more specific situations. For example, to change the background color in dark mode, at the medium breakpoint, on hover:

```swift
let stacked: UtilityClass<BackgroundColor> = .dark(.md(.hover(.fuchsia600)))
stacked.classname // "dark:md:hover:bg-fuchsia-600"
```


## Usage

WindRose is designed to be used with a one of the [many](https://github.com/BinaryBirds/swift-html) [HTML](https://github.com/tayloraswift/swift-dom) [DSLs](https://github.com/pointfreeco/swift-html) available for Swift and is not particularly useful outside of additional, library-specific support. The [Plot](https://github.com/JohnSundell/Plot) library, for example, defines a `Component` protocol that can be used as follows:

```swift
struct Banner: Component {
    var title: String
    var imageURL: URLRepresentable

    var body: Component {
        Div {
            H2(title)
            Image(imageURL)
        }
        .class("banner")
    }
}
```

Given additional extensions on that protocol to provide support for each property

```swift
public extension Component {
    func `class`<Property>(_ utilityClasses: [UtilityClass<Property>]) -> Component {
        attribute(
            named: "class",
            value: className(utilityClasses),
            replaceExisting: false)
    }
}

public extension Component {
    func backgroundAttachment(_ utilityClasses: UtilityClass<BackgroundAttachment>...) -> Component {
        self.class(utilityClasses)
    }

    func backgroundClip(_ utilityClasses: UtilityClass<BackgroundClip>...) -> Component {
        self.class(utilityClasses)
    }

    func backgroundColor(_ utilityClasses: UtilityClass<BackgroundColor>...) -> Component {
        self.class(utilityClasses)
    }
    
    ...
}
```

WindRose enables expressive use of Tailwind's utility classes:

```swift
struct ChatNotification: Component {
    var body: Component {
        Div {
            Div {
                Image(url: "/img/logo.svg", description: "ChitChat Logo")
            }
            
            Div {
                Text("ChitChat")
                    .fontSize(.xl)
                    .fontWeight(.medium)
                    .textColor(.black)

                Paragraph("You have a new message")
                    .textColor(.slate500)
            }
        }
        .backgroundColor(.white)
        .display(.flex)
        .itemAlignment(.center)
        .margin(.horizontal(.auto))
    }
}
```
