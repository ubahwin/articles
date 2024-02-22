- [Мощь key path в Swift](#мощь-key-path-в-swift)
  - [Основы](#основы)
  - [Синтаксический сахар](#синтаксический-сахар)
    - [`map`](#map)
    - [`sorted`](#sorted)
  - [Не требуется экземпляр](#не-требуется-экземпляр)
  - [Преобразование в функции](#преобразование-в-функции)

# Мощь key path в Swift

> Это перевод
> [этой статьи](https://www.swiftbysundell.com/articles/the-power-of-key-paths-in-swift/)

Поскольку Swift изначально разрабатывался с акцентом на безопасность во время
компиляции и статическую типизацию, в нем в основном отсутствуют динамические
функции, которые обычно встречаются в языках, более ориентированных на время
выполнения, таких как Objective-C, Ruby и JavaScript. Мы могли бы динамически
получать доступ к любому свойству или методу объекта — и даже менять местами его
реализацию — во время выполнения.

Хотя этот недостаток динамизма (dynamism) является основной причиной того,
почему Swift так хорош, поскольку он помогает нам писать код, который более
предсказуем и имеет более высокие шансы на корректность, но иногда возможность
работать с нашим кодом более динамичным образом была бы действительно полезна.

К счастью, Swift продолжает получать все больше и больше возможностей, которые
являются более динамичными по своей природе, сохраняя при этом акцент на
безопасном наборе кода. Одной из таких фичей являются key path.

## Основы

Key Path, по сути, позволяют нам ссылаться на любое свойство экземпляра как на
отдельное значение.

Они бывают трех основных вариантов:

- `KeyPath`: Предоставляет доступ к свойству только для чтения
- `WritableKeyPath`: Предоставляет доступ для чтения и записи к изменяемому
  свойству с семантикой значения (поэтому рассматриваемый экземпляр также должен
  быть изменяемым, чтобы была разрешена запись).
- `ReferenceWritableKeyPath`: Может использоваться только со ссылочными типами
  (такими как экземпляры класса) и предоставляет доступ для чтения и записи к
  любому изменяемому свойству

> Также существует несколько дополнительных типов key path, которые
> предназначены для уменьшения дублирования внутреннего кода и облегчения
> удаления типов, но в этой статье мы сосредоточимся на основных типах

## Синтаксический сахар

#### `map`

Есть функция `.map` у `Sequence`, которая позволяет делать вот так:

```swift
struct Article {
    let title: String
    let body: String
}

let articleIDs = articles.map { $0.title }
let articleSources = articles.map { $0.body }
```

Однако, если мы расширим `Sequence`, перегрузив метод `map`:

```swift
extension Sequence {
    func map<T>(_ keyPath: KeyPath<Element, T>) -> [T] {
        return map { $0[keyPath: keyPath] }
    }
}
```

Мы сможем писать вот так:

```swift
let articleIDs = articles.map(\.title)
let articleSources = articles.map(\.body)
```

#### `sorted`

Более сложный пример:

```swift
extension Sequence {
    func sorted<T: Comparable>(
        by keyPath: KeyPath<Element, T>
    ) -> [Element] {
        return sorted { a, b in
            return a[keyPath: keyPath] < b[keyPath: keyPath]
        }
    }
}
```

И теперь мы можем:

```swift
playlist.songs.sorted(by: \.name)
```

Стандартная библиотека способна автоматически сортировать любую
последовательность, содержащую сортируемые (`Sortable`) элементы, но для всех
других типов мы должны предоставить собственные замыкания (closure) сортировок.
Однако, используя key path, мы можем легко добавить поддержку сортировки любой
последовательности на основе key path для сравниваемого (`Comparable`) значения.

## Не требуется экземпляр

Хотя правильное количество синтаксического сахара может быть приятным, истинная
сила key path заключается в том, что они позволяют нам ссылаться на свойство без
необходимости связывать его с каким-либо конкретным экземпляром. Давайте
предположим, что мы работаем над приложением, которое отображает списки песен, и
чтобы настроить `UITableViewCell` в пользовательском интерфейсе для такого
списка, мы используем тип конфигуратора, который выглядит следующим образом:

```swift
struct SongCellConfigurator {
    func configure(_ cell: UITableViewCell, for song: Song) {
        cell.textLabel?.text = song.name
        cell.detailTextLabel?.text = song.artistName
        cell.imageView?.image = song.albumArtwork
    }
}
```

Структура работает только с типами `Song`, давайте сделаем ее универсальной,
добавив дженериков и key path:

```swift
struct CellConfigurator<Model> {
    let titleKeyPath: KeyPath<Model, String>
    let subtitleKeyPath: KeyPath<Model, String>
    let imageKeyPath: KeyPath<Model, UIImage?>

    func configure(_ cell: UITableViewCell, for model: Model) {
        cell.textLabel?.text = model[keyPath: titleKeyPath]
        cell.detailTextLabel?.text = model[keyPath: subtitleKeyPath]
        cell.imageView?.image = model[keyPath: imageKeyPath]
    }
}
```

И мы можем теперь:

```swift
let songCellConfigurator = CellConfigurator<Song>(
    titleKeyPath: \.name,
    subtitleKeyPath: \.artistName,
    imageKeyPath: \.albumArtwork
)
```

## Преобразование в функции

До сих пор мы только считывали значения с помощью key path, но теперь давайте
посмотрим, как мы можем использовать их и для динамической записи значений.

Допустим у нас есть вот такая _ViewModel_:

```swift
class PostsViewModel: ObservableObject {
    @Published var posts = [Post]()

    private let service: Service

    init(posts: [Post] = [Post](), service: Service) {
        self.posts = posts
        self.service = service
    }

    func loadPosts() {
        service.load { [weak self] posts in
            self?.posts = posts
        }
    }
}
```

На этот раз мы будем использовать тип `ReferenceWritableKeyPath`, поскольку мы
хотим ограничить функциональность только ссылочными типами (в противном случае
мы бы вносили изменения только в локальную копию значения). Учитывая объект и
key path для этого объекта, мы затем автоматически зафиксируем объект как слабую
(`weak`) ссылку и присвоим свойство, соответствующее key path, как только будет
вызвана наша функция — вот так:

```swift
func setter<Object: AnyObject, Value>(
    for object: Object,
    keyPath: ReferenceWritableKeyPath<Object, Value>
) -> (Value) -> Void {
    return { [weak object] value in
        object?[keyPath: keyPath] = value
    }
}
```

Теперь мы можем переписать `loadPosts`:

```swift
func loadPosts() {
    service.load(then: setter(for: self, keyPath: \.posts))
}
```
