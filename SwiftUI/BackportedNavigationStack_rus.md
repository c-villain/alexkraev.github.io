# Навигация

Один из блоков вопросов на iOS - собеседовании - архитектура приложений. 
При этом почти в любой архитектуре вопросы навигации всегда находятся сбоку от обсуждения. 
Более того, для навигации разрабатывают свои паттерны. Одними из таких являются координатор и навигатор.

Начиная с SwiftUI 1.0 Apple практически на каждом WWDC рассказывает про работу с MVVM, как будто забывая про роутинг. 
Да, нам показали [`NavigationView`](https://developer.apple.com/documentation/swiftui/navigationview), 
[`NavigationLink`](https://developer.apple.com/documentation/swiftui/navigationlink), 
но не покидало ощущение, что Apple опять представили что-то промежуточное. 
Многие стали писать свои обертки над этим API, чтобы сделать работу удобнее. 
И наконец в iOS 16 Apple представили новое API навигации, которое так долго ждали. 

Вместо [`NavigationView`](https://developer.apple.com/documentation/swiftui/navigationview) (deprecated) 
теперь нужно использовать [`NavigationStack`](https://developer.apple.com/documentation/swiftui/navigationstack). 
Экран для перехода будет определять модификатор 
[`navigationDestination(for:destination:)`](https://developer.apple.com/documentation/swiftui/list/navigationdestination(for:destination:)).

Будем честны, многие команды до сих пор используют роутинг на UIKit в проектах на SwiftUI. 
Даже те, кто пытались разобраться с [`NavigationView`](https://developer.apple.com/documentation/swiftui/navigationview), в конечном итоге возвращались обратно в UIKit. 
С появлением нового API навигации такой подход - явно поворот не туда. 
Но новое API требует минимальный таргет у проекта iOS 16.0 .  

Что делать?  Использовать [бэкпорт](https://github.com/johnpatrickmorgan/NavigationBackport)! 
Можете создать свой тестовый проект, чтобы поработать с [этой библиотекой](https://github.com/johnpatrickmorgan/NavigationBackport). 

Мой сэмпл [здесь](https://github.com/c-villain/NaviBackportExample).
