- [Управление состояниями без использования ObservableObject](#управление-состояниями-без-использования-observableobject)
- [Обёртка в EquatableView](#обёртка-в-equatableview)
- [Snapshot состояния в View](#snapshot-состояния-в-view)
- [Обертка ObservableObject в ObservableObject](#обертка-observableobject-в-observableobject)
- [Использование Publisher вместо ObservableObject](#использование-publisher-вместо-observableobject)

# Управление состояниями без использования ObservableObject

> Это перевод [этой](https://nalexn.github.io/swiftui-observableobject/) статьи

SwiftUI был разработан c поощрением создания приложений в стиле single source of
truth. Однако при использовании `@ObservedObject` снижается производительность
при большом количестве подписчиков. Это значит, что при разработке большого
приложения начнутся проблемы с централизованном состоянии.

# Обёртка в EquatableView

В SwiftUI существует
[`EquatableView`](https://developer.apple.com/documentation/swiftui/equatableview),
который сравнивает предыдущее View и обновленное и, если оно не поменялось,
предотвращает обновление.

```swift
class AppState: ObservableObject {
    @Published var value: Int = 0
}

struct CustomView: View, Equatable {
    @EnvironmentObject var appState: AppState

    var body: some View {
        Text("Value: \(appState.value)")
    }

    static func == (lhs: Self, rhs: Self) -> Bool {
        return lhs.appState.value == rhs.appState.value
    }
}
```

Оборачиваем:

```swift
CustomView().equatable().environmentObject(AppState())
```

Это позволит прописывать свои собственные стратегии по сравнению, вместо
сравнения `body`.

Однако, это не работает – приложение просто зависает, так как `AppState` это
[reference type](https://developer.apple.com/swift/blog/?id=10), его SwiftUI не
будет копировать, поэтому постоянно сравнивается один и тот же instance.

# Snapshot состояния в View

Можно попробовать просто сохранять внутри view snapshot:

```swift
struct CustomView: View, Equatable {
    @EnvironmentObject var appState: AppState
    private var prevValue: Int

    var body: some View {
        // Сохраняем предыдущее состояние
        self.prevValue = appState.value

        return Text("Value: \(appState.value)")
    }

    static func == (lhs: Self, rhs: Self) -> Bool {
        return lhs.prevValue == rhs.appState.value
    }
}
```

Однако, это банально не скомпилируется, так как `self` immutable.

# Обертка ObservableObject в ObservableObject

В Combine есть `removeDuplicates()`, которая убирает ненужные обновления, при
которых значения не меняются.

Однако, в SwiftUI `@ObservableObject` и `@EnvironmentObject` не имеют такой
функциональности. Поэтому придется обернуть `ObservableObject` во что-то другое.

Этой оболочкой может быть
[`Deduplicated`](https://gist.github.com/nalexn/ace9ddd07db5a6e150163712e20c6235),
его главная часть:

```swift
object.objectWillChange
    .delay(for: .nanoseconds(1), sheduler: RunLoop.main)
    .compactMap { _ in makeSnapshot() }
    .prepend(makeSnapshot())
    .removeDuplicates()
    .dropFirst()
    .sink { [weak self] _ in
        self?.objectWillChange.send()
    }
```

Тут циклично делается snapshot и убираются дупликаты. Теперь можно:

```swift
struct CustomView: View {
    @EnvironmentObject var appState: Deduplicated<AppState, Snapshot>

    var body: some View {
        Text("Value: \(appState.value)")
    }

    struct Snapshot {
        let value: Int
    }
}

let view = CustomView().environmentObject(
    appState.deduplicated {
        Snapshot(value: $0.value)
    }
)
```

Но есть проблемы с таким подходом:

- Обновления происходят асинхронно из-за `.delay`, это может привести к race
  condition при использовнии UIKit.

# Использование Publisher вместо ObservableObject

```swift
struct AppState {
    var value1: Int = 0
    var value2: Int = 0
    var value3: Int = 0
}

struct ContentView: View {
    @Environment(\.injected) private var injected: AppState.Injection

    @State private var state = ViewState()

    var body: some View {
        Text("Value: \(state.value)")
            .onReceive(stateUpdate) { self.state = $0 }
    }

    private var stateUpdate: AnyPublisher<ViewState, Never> {
        injected.appState.map { $0.viewState }
            .removeDuplicates().eraseToAnyPublisher()
    }

    struct ViewState: Equatable {
        var value: Int = 0
    }
}

extension AppState {
    var viewState: ContentView.ViewState {
        return .init(value: value1)
    }

    struct Injection: EnvironmentKey {
        let appState: CurrentValueSubject<AppState, Never>

        static var defaultValue: Self {
            return .init(appState: .init(AppState()))
        }
    }
}
```

У этого подхода больше плюсов:

- Легко масштабировать. Не нужно ничего менять при добавлении нового view
- Используем нативную dependency injection `@Environment`, вместо
  `@EnvironmentObject`
- Код чище

Если нам понадобится `Binding`, наприпет в `.sheet`:

```swift
extension Binding where Value: Equatable {
    func dispatched(
        to state: CurrentValueSubject<AppState, Never>,
        _ keyPath: WritableKeyPath<AppState, Value>
    ) -> Self {
        return .init { () -> Value in
            self.wrappedValue
        } set: { newValue in
            self.wrappedValue = newValue
            state.value[keyPath: keyPath] = newValue
        }
    }
}

var body: some View {
    ...
    .sheet(isPresented: viewState.showSheet
        .dispatched(to: appState, \.routing.showSheet)
    ) { ... }
}
```
