# Managing Theme Colors in SwiftUI

I always feel compelled to provide a feature to change the theme color in my apps. In SwiftUI, this is quite simple, but there are a few key points to keep in mind.

## Setting Theme Colors for a View

You can use the `tint` modifier to set a theme color for a View and its subviews, like this:

```swift
.tint(.red)
```

Generally, you only need to apply this modifier to the top-level View. Components like Buttons, which have built-in theme colors, will change according to the theme color you set.

## Accessing the Set Theme Color within a View

If you want to access the theme color you set, there are two ways to do so.

One way is through `Color.accentColor`, which can directly fetch this value:

```swift
.foregroundStyle(Color.accentColor)
```

> ⚠️ Through testing, `accentColor` can sometimes be unreliable for unknown reasons. There is a workaround for this later on.

Another way is using `.tint` with modifiers that support `ShapeStyle`:

```swift
.foregroundStyle(.tint)
.background(.tint)
.fill(.tint)
.storke(.tint)
```

## Setting Theme Colors for System Components

Some system components do not follow the `tint` color setting, such as Alert and FileImporter. In these cases, you can use `UIView.appearance` to set the theme color, like this:

```swift
UIView.appearance(whenContainedInInstancesOf: [UIAlertController.self]).tintColor = UIColor(tintColor)
UIView.appearance(whenContainedInInstancesOf: [UIDocumentPickerViewController.self]).tintColor = UIColor(tintColor)
```

## Bringing It All Together

Below is a `ViewModifier` I wrote to set a unified theme color for all Views in the app. It also includes settings related to `colorScheme`.

Notably, I added a `tintColor` environment value to help resolve the issue where `Color.accentColor` sometimes does not work. You can directly access the set theme color via this value.

```swift
private struct TintColor: EnvironmentKey {
    static let defaultValue: Color = .blue
}

extension EnvironmentValues {
    var tintColor: Color {
        get { self[TintColor.self] }
        set { self[TintColor.self] = newValue }
    }
}

struct ThemeModifier: ViewModifier {
    @Default(.customColorScheme) var customColorScheme
    @Default(.customTintColor) var customTintColor
    @Environment(\.colorScheme) private var colorScheme

    var tintColor: Color {
        if let themeColor = ThemeColor(rawValue: customTintColor) {
            return themeColor.color
        } else if let uiColor = UIColor(hexString: customTintColor) {
            return Color(uiColor: uiColor)
        }
        return .indigo
    }

    var preferredColorScheme: ColorScheme {
        customColorScheme.preferredColorScheme ?? colorScheme
    }

    func body(content: Content) -> some View {
        content
            .preferredColorScheme(customColorScheme.preferredColorScheme)
            .tint(tintColor)
            .environment(\.tintColor, tintColor)
            .onAppear {
                updateColorScheme(colorScheme: preferredColorScheme)
                updateTintColor(tintColor: tintColor)
            }
            .onChange(of: preferredColorScheme) {
                updateColorScheme(colorScheme: $0)
            }
            .onChange(of: tintColor) {
                updateTintColor(tintColor: $0)
            }
    }

    func updateColorScheme(colorScheme: ColorScheme) {
        if colorScheme == .dark {
            UIView.appearance(whenContainedInInstancesOf: [UIAlertController.self]).overrideUserInterfaceStyle = .dark
            UIView.appearance(whenContainedInInstancesOf: [UIDocumentPickerViewController.self]).overrideUserInterfaceStyle = .dark
        } else {
            UIView.appearance(whenContainedInInstancesOf: [UIAlertController.self]).overrideUserInterfaceStyle = .light
            UIView.appearance(whenContainedInInstancesOf: [UIDocumentPickerViewController.self]).overrideUserInterfaceStyle = .light
        }
    }

    func updateTintColor(tintColor: Color) {
        UIView.appearance(whenContainedInInstancesOf: [UIAlertController.self]).tintColor = UIColor(tintColor)
        UIView.appearance(whenContainedInInstancesOf: [UIDocumentPickerViewController.self]).tintColor = UIColor(tintColor)
    }
}

extension View {
    func useTheme() -> some View {
        modifier(ThemeModifier())
    }
}

@main
struct DemoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .useTheme()
        }
    }
}
```

> In the code, I used [Defaults](https://github.com/sindresorhus/Defaults) to simplify the operation of UserDefaults.
