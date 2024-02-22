- [Callbacks, часть 1: Delegate, NotificationCenter и KVO](#callbacks-часть-1-delegate-notificationcenter-и-kvo)
  - [Delegate](#delegate)
    - [Преимущества использования делегата](#преимущества-использования-делегата)
    - [Недостатки использования делегата](#недостатки-использования-делегата)
  - [`NotificationCenter`](#notificationcenter)
    - [Преимущества NotificationCenter](#преимущества-notificationcenter)
    - [Недостатки NotificationCenter](#недостатки-notificationcenter)
  - [Key Value Odserving (KVO)](#key-value-odserving-kvo)
    - [Преимущества KVO](#преимущества-kvo)
    - [Недостатки KVO](#недостатки-kvo)

# Callbacks, часть 1: Delegate, NotificationCenter и KVO

> Это перевод
> [этой](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/)
> статьи

В этой статье рассматриваются callback-техники в
[Cocoa](<https://en.wikipedia.org/wiki/Cocoa_(API)>)

## Delegate

Delegate – это общий случай
[слабого связывания](https://en.wikipedia.org/wiki/Loose_coupling) между
сущностями в Cocoa, этот метод может быть использован и в других языках и
платформах, так как это
[делегирование](https://ru.wikipedia.org/wiki/Шаблон_делегирования) (паттерн
проектирования).

Идея проста - вместо взаимодействия с объектом через его реальный API мы можем
вместо этого опустить детали о том, к какому точному типу мы обращаемся, и
вместо этого использовать `protocol` с подмножеством методов, которые нам нужны.
Мы определяем протокол сами и перечисляем только те методы, которые мы
собираемся вызывать. Реальный объект затем должен реализовать этот протокол, и,
вероятно, другие методы - но нам это даже знать не нужно.

Рассмотрим пример, есть пиццерия, она делает пиццу, ей не важно кто будет
покупателем, важно только то, что он умеет только _брать пиццу, когда она
готова_:

```swift
protocol PizzaTaker { func take(pizza: Pizza) }

class DodoPizza {
    var pizzaTaker: PizzaTaker?

    func makePizza() {
        ...
    }

    private func onPizzaIsReady(pizza: Pizza) {
        pizzaTaker?.take(pizza: pizza)
    }
}

class Customer: PizzaTaker {
    var name: String

    func take(pizza: Pizza) {
        eat(pizza)
    }
}
```

### Преимущества использования делегата

- Слабая связывание
- IDE (как XCode) может проверять корректность этих связываний
- Можно не только уведомлять, но и запрашивать данные (non-void func)
- Производительность, вызов метода делегата – прямой вызов функции

### Недостатки использования делегата

- Единственный получатель – в примере пиццу мог получить только один
  `PizzaTaker`
- Распределение тесно связанной бизнес логики
  ([низкая согласованность](<https://en.wikipedia.org/wiki/Cohesion_(computer_science)>)).
  Поскольку все методы обратного вызова (callback) должны быть реализованы как
  отдельные функции внутри получателя, мы не можем использовать анонимные
  функции и получать _action_ и _обратный вызов для action_, вложенные рядом
  друг с другом. Поэтому иногда трудно рассуждать о том, при каких условиях
  вызывается обратный вызов.
- Громоздкая инфраструктура

## `NotificationCenter`

`NotificationCenter` – класс в Cocoa который предоставляет функциональность
[издателя-подписчика](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)
и имеет дело с уведомлениями. Уведомления, которые он рассылает, являются
объектами класса `Notification`, которые могут нести необязательную полезную
нагрузку, но чаще используются просто как уведомления. `NotificationCenter` сам
по себе является шиной данных, он не отправляет уведомления сам по себе, только
когда кто-то просит его об отправке. Основная особенность этого шаблона
заключается в том, что отправитель и получатели (их может быть много) не
общаются напрямую, как в шаблоне делегирования. Вместо этого они оба общаются с
`NotificationCenter`.

`NotificationCenter` также предоставляет глобальную точку доступа (access point)
`defaultCenter`, однако этот класс не реализован как чистый синглтон - вам нужно
создать свой собственный экземпляр `NotificationCenter`.

Реализация такого кейса с пиццерией:

```swift
extension NSNotification.Name {
    static let PizzaReadiness = NSNotification.Name(rawValue: "pizza_is_ready")
}

class DodoPizza {
    func makePizza() { }

    private func onPizzaIsReady(pizza: Pizza) {
        NotificationCenter.default.post(
            name: NSNotification.Name.PizzaReadiness,
            object["pizza_object" : pizza]
        )
    }
}

class Customer {
    func startListeningWhenPizzaIsReady() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(pizzaIsReady(notification:))
        )
    }

    func stopListeningWhenPizzaIsReady() {
        NotificationCenter.default.removeObserver(
            self,
            name: NSNotification.Name.PizzaReadiness,
            object: nil
        )
    }

    dynamic func pizzaIsReady(notification: Notification) {
        if let pizza = notification.userInfo?["pizza_object"] as? Pizza {
        // Got the pizza!
        }
    }
}
```

### Преимущества NotificationCenter

- Несколько получателей. `NotificationCenter` прозрачно доставляет уведомление
  всем подписчикам. Также отправитель не может знать, сколько у него
  подписчиков.
- Слабое связывание. Eдинственное, что связывает отправителя и получателей
  вместе - это название уведомления, которое они используют для своей связи/
- Глобальная точка доступа. Когда вам не нужны
  [внедрения зависимостей](https://en.wikipedia.org/wiki/Dependency_injection) в
  ваш код, singleton `defaultCenter` позволит с легкостью соединить два
  случайных объекта вместе

### Недостатки NotificationCenter

- Глобальная точка доступа. Да, ещё и недостаток. Любая глобально доступная
  часть нарушает тестируемость вашего кода, и даже singleton как шаблон
  проектирования многие люди рассматривают как
  [anti-pattern](https://stackoverflow.com/questions/137975/what-are-drawbacks-or-disadvantages-of-singleton-pattern)
- Невозможность подключиться с помощью отладчика
- Неконтролируемый поток управления. Если вы пытаетесь понять бизнес-логику
  программы и увидеть место, куда отправляется уведомление, единственный способ
  продолжить исследование - найти всех получателей вручную с помощью текстового
  поиска во всем проекте, потому что они могут быть где угодно
- Передача данных очень подвержена ошибкам из-за упаковки и распаковки со
  словарем (`NSDictionary`). Даже при использовании из типобезопасного Swift
  компилятор не может проверить типы, а также структуру `Dictionary`
- Получатели должны отказаться от подписки, когда они собираются освободить
  место, иначе приложение может выйти из строя (это требование было удалено
  только в iOS 8)
- Отправитель не может отправить _void_ результат в отличие от делегата и
  закрытия
- Сторонние библиотеки могут полагаться на те же уведомления, что и ваш код, и
  мешать друг другу. Отличным примером является
  [NSManagedObjectContextDidSaveNotification из **CoreData**](https://stackoverflow.com/questions/17568667/ios-googleanalytic-continuously-produce-an-exception/27739971#27739971)
- Нет никакого контроля над тем, кто имеет право отправлять конкретное
  уведомление

## Key Value Odserving (KVO)

Есть
[специальный пост](https://nalexn.github.io/kvo-guide-for-key-value-observing/),
который автор написал о KVO в Swift 5 и Objective-C.

**KVO** - это традиционный
[паттерн наблюдатель](https://en.wikipedia.org/wiki/Observer_pattern),
встроенный в любой `NSObject` из коробки. С помощью KVO наблюдатели могут
получать уведомления о любых изменениях `@property` значений. Он использует
_среду выполнения Objective-C_ для автоматической отправки уведомлений, и из-за
этого для класса Swift вам нужно будет выбрать _динамизм Objective-C_,
унаследовав `NSObject` и пометив переменную, которую вы собираетесь наблюдать,
модификатором `dynamic`. Наблюдатели также должны быть потомками `NSObject`,
поскольку это обеспечивается KVO API.

Простой пример:

```swift
class ObservedClass: NSObject {
    @objc dynamic var value: CGFloat = 0
}

class Observer {
    var kvoToken: NSKeyValueObservation?

    func observe(object: ObservedClass) {
        kvoToken = object.observe(\.value, options: .new) { (object, change) in
            guard let value = change.new else { return }
            print("New value: \(value)")
        }
    }

    deinit {
        kvoToken?.invalidate()
    }
}
```

### Преимущества KVO

- Уже готово из коробки в `NSObject` в Cocoa
- Несколько наблюдателей, подписчики неограничены
- Нет необходимости изменять исходный код наблюдаемого класса
- Вы можете наблюдать объекты любого класса (в том числе из системных
  фреймворков, к источникам которых у нас нет доступа, поэтому мы не можем их
  изменять)
- Очень низкая связанность
  ([connascence of name](https://en.wikipedia.org/wiki/Connascence#Static_Connascences)),
  наблюдаемая сторона даже не знает, что за ней наблюдают, знания наблюдающей
  стороны ограничены именем `@property`
- Уведомление может быть настроено таким образом, чтобы доставлять не только
  самое последнее значение наблюдаемого свойства `@property`, но и предыдущее
  значение.

### Недостатки KVO

- Один из худших API в Cocoa:
  - https://www.mikeash.com/pyblog/key-value-observing-done-right.html
  - https://khanlou.com/2013/12/kvo-considered-harmful/
  - https://ianthehenry.com/posts/kvo-101/
  - https://nshipster.com/key-value-observing/
- `keyPath`, используемый для подписки, представляет собой строку, и он не может
  быть статически проверен. К счастью, для этого есть решение в Swift (директива
  `#keyPath`), но в Objective-C, если наблюдаемая сторона изменит имя
  `@property` - предупреждения компилятора об этом не будет, поэтому приложение
  просто завершит работу во время выполнения
- Каждый наблюдатель должен явно отказаться от подписки на `deinit` - в
  противном случае сбой неизбежен
- Мы должны вызвать реализацию `super` функции обратного вызова
  `observeValueForKeyPath`, чтобы убедиться, что мы не нарушаем реализацию
  суперкласса
- Относительно низкая производительность. Даже при компиляции с оптимизацией
  `-Os` для Objective-C одно уведомление KVO выполняется в _30 раз медленнее_,
  чем вызов функции, для Swift - в _200 раз медленнее_. Я использовал
  [этот проект](https://github.com/nalexn/PerformanceTestTools) для
  сравнительного анализа

К счастью для нас, есть более приятные альтернативные реализации KVO:

- https://github.com/facebookarchive/KVOController
- https://github.com/postmates/PMKVObserver
