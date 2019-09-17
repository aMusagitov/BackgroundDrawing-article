# Генерация изображений с использованием CGContext

Цель:
Генерирование изображений путем наложения различных объектов (кривые, символы, текст, изображения) в определенной последовательности на базовое изображение в фоновом потоке.

Решение:
Создать изображение можно несколькими способами:
+ отрисовывать объекты UIKit: UIView, UIImageView и т.д.
+ отрисовывать объекты Core Animation: CALayer, CATextLayer и т. д., и объекты Core Animation и объекты UIKit отрисовываются в контексте CGContext. 
Основные отличия в том, что объекты UIKit нельзя отрисовывать в фоновом потоке, что необходимо для поставленной задачи, поэтому были выбраны объекты CALayer и его подклассы.

Сначала нужно создать основной слой(CALayer), который будет представлять оригинальное изображение. Чтобы не терять качество, размер основного слоя должен быть равен размеру оригинального изображения. Далее для каждого объекта наложения создается свой слой (CALayer для символов, CAShapeLayer для кривых, CATextLayer для текста). На каждом слое применяется необходимая конфигурация (цвет, толщина кривых, прозрачность, расположение в основном слое - origin). Все эти слои нужно вставить в основной слой.

После конфигурации всех слоев необходимо создать контекст CGContext. Это можно сделать несколькими способами:
1) Прописать необходимую конфигурацию вручную.
2) Воспользоваться методом: *UIGraphicsBeginImageContext(size)* или *UIGraphicsBeginImageContextWithOptions(size,opaque,scale)*.
3) Создать рендерер UIGraphicsImageRenderer с необходимым контекстом, воспользовавшись методом: *UIGraphicsImageRenderer(size: rect.size)*.
Попробовав вышеперечисленные методы выбор пал на первый вариант, так как он показал более быструю скорость выполнения.

Моя настройка выглядит так:
[snippet](https://gitlab.com/snippets/1895570)
```swift
let width = Int(layer.frame.size.width)
let height = Int(layer.frame.size.height)
let colorSpace = CGColorSpaceCreateDeviceRGB()
let bytesPerPixel = 4
let bytesPerRow = bytesPerPixel * width
let rawData = malloc(height * bytesPerRow)
let bitsPerComponent = 8
guard let context = CGContext(data: rawData, width: width, height: height, 
                              bitsPerComponent: bitsPerComponent, bytesPerRow: bytesPerRow,
                              space: colorSpace, 
                              bitmapInfo: CGImageAlphaInfo.premultipliedFirst.rawValue | CGImageByteOrderInfo.order32Big.rawValue) else { return }
```

Перед рендером слоя в контексте необходимо проверить, перевернут ли слой.
[snippet](https://gitlab.com/snippets/1895576)
```swift
if layer.contentsAreFlipped() {
    let flipVertical = CGAffineTransform(a: 1, b: 0, c: 0, d: -1, tx: 0, ty: layer.frame.size.height)
    context.concatenate(flipVertical)
}
```
Вызываем рендер слоя в контексте: *layer.render(in: context)* и создаем изображение: *guard let cgImage = context.makeImage()*

В процессе работы я столкнулся с утечками памяти при работе с контекстом. Сначала я попробовал высвободить память, которую выделял для контекста, функцией *free(rawData)*. Это чуть-чуть помогло, но все равно утечка осталась, хоть и уменьшилась.

Следующим действием было оборачивание всей работы с контекстом в autorelease {} блок. Казалось бы, что утечка пропала, но при одновременном рендере нескольких изображений в фоновом потоке main thread блокировался, и утечки появились вновь. В итоге было решено каждый рендер изображения обернуть в операцию и использовать operationQueue, autorelease блок был убран, main thread перестал блокироваться и утечки пропали.

[Тестовый проект](https://gitlab.com/aMusagitov/backgroundimagecomposer)
