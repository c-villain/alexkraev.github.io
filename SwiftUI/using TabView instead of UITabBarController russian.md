# Переход на SwiftUI: внедряем TabView взамен UITabBarController.
___

*English version of this article you may find [here](https://c-villain.github.io/SwiftUI/using%20TabView%20instead%20of%20UITabBarController) 
or  on [medium](https://medium.com/@lexkraev/moving-to-swiftui-using-tabview-instead-of-uitabbarcontroller-ac9fee481cda).*

*Вы можете также прочитать эту статью на [хабре](https://habr.com/ru/company/lenta_utkonos_tech/blog/674888/).*
___
Внедрение SwiftUI (далее — SUI) в уже существующее приложение, написанное на UIKit, в середине 2022 г. у
же не является вопросом времени, а скорее, определяется наличием соответствующих навыков. 
Перевод [приложения]((https://apps.apple.com/ru/app/id582201863)) [Утконоса]((https://www.utkonos.ru/)) – одного из лидеров 
e-commerce на российском рынке – на SUI мы начали в конце 2020 года, 
когда подняли минимальную поддерживаемую версию iOS до 13-ой (и, да, мы не стали ждать 14-ой). 
Этому же способствовала поставленная долгосрочная задача полного редизайна приложения. 
На текущий момент из пяти главных экранов на SUI у нас реализованы два.

Одна из главных задач, стоящих перед разработчиками — проектирование навигации в приложении. 
Сейчас уже редко можно встретить одностраничное приложение. Панель вкладок (или таб-бар) позволяет реализовать 
пользовательский интерфейс, в котором доступ к нескольким экранам не выполняется строго в определенном порядке.
Если приложение пишется с нуля на SUI, то типичным сценарием разработки все еще является следующий: 
экраны верстаются на SUI, а таб-бар на UIKit. 
С ростом кодовой базы на SUI в Утконосе мы стали постепенно отказываться от навигации на UIKit, 
большим шагом в этом направлении стало внедрение TabView взамен UITabBarController.
___
Всем привет! Меня зовут [Краев Александр](http://linkedin.com/in/alex-kraev) и ниже хочу поделиться опытом перевода UIKit-вого таб-бара на 
TabView со всеми подводными камнями: когда у вас есть экраны, написанные как на SUI, так и на UIKit. 
Сразу оговорюсь, что данная статья не рассчитана на тех, кто только начал знакомиться со SUI: внедрение 
нового фреймворка я советовал бы начать с какого-нибудь небольшого уже существующего экрана или новой продуктовой фичи.
Больше всяких фишек и интересных разборов вы сможете найти на [моем telegram-канале](https://t.me/swiftui_dev), посвященном iOS-разработке на SwiftUI.
___
## Подготавливаем инфраструктуру
В нашей команде мы работаем по [Trunk Based Development](https://trunkbaseddevelopment.com/) (TBD). 
Если вы не знакомы с данной моделью ветвления, то советую вам посмотреть [это выступление]((https://www.youtube.com/watch?v=ykZbBD-CmP8)) 
или прочитать [эту статью](https://habr.com/ru/post/519314/). 
Если коротко, то разработка новых фичей идет через Feature Flags. 

Заведем флаг для нового таб-бара на SUI:

```swift
struct SwiftUI {
    struct TabView {
        static var isActive: Bool = true
    }
}
```

В той части кода, где создается корневой view controller у главного window, уже можно написать:

```swift
var main: UIWindow?
func createMainWindow(windowScene: UIWindowScene) {
    main = UIWindow(windowScene: windowScene)
    let mainTab = FeatureFlags.SwiftUI.TabView.isActive ? 
                        UIHostingController(rootView: RootTabView()) : 
                        setupTabBarController()
    main?.rootViewController = mainTab
}
```

где ```setupTabBarController()``` – это функция создания прежнего таб-бара на UIKit, а ```RootTabView()``` – это view нового таб-бара на SUI, 
проброшенная через ```UIHostingController```. 

Иерархия в прежнем таб-баре весьма привычная: для каждого экрана создается его navigation controller с корневым view controller-ом:

```swift
let profileVC: ProfileViewController = .init()
let profileNav: NavigationController = .init(rootViewController: profileVC)
```

После инициализируется сам tab bar controller, у которого view controller-ы это созданные на предыдущем шаге navigation controller-ы:

```swift
private func setupTabBarController() -> UIViewController {
    ...
    let profileVC: ProfileViewController = .init()
    let profileNav: NavigationController = .init(rootViewController: profileVC)
    ...
    let tabbarController: TabBarController = .init()
    tabbarController.viewControllers = [..., profileNav, ...]
    return tabbarController
}
```

Здесь стоит пояснить, что ```NavigationController``` – это класс, наследуемый от ```UINavigationController```,
с кастомным поведением навигационной панели, в том числе ее внешнего вида, кнопки назад, но не более.

Вернемся к новому таб-бару, очевидно, что в ```RootTabView()``` будут располагаться view главных экранов. 
Самое время начать писать SUI-обертки на ```UIViewControllerRepresentable``` для UIKit-экранов, приведу пример одной такой 
для экрана профиля пользователя:

```swift
import SwiftUI

struct ProfileSUIView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> NavigationController {
        let profileVC: ProfileViewController = .init()
        let profileNav: NavigationController = .init(rootViewController: profileVC)
        return profileNav
    }

    func updateUIViewController(_ uiViewController: NavigationController, 
                                context: Context) {
    }
}
```

Как уже сказал ранее, у нас есть два экрана, реализованных целиком на SUI. 
Чтобы не ломать роутинг на этих экранах ввиду других legacy экранов на UIKit, 
решено было их также обернуть через ```UIViewControllerRepresentable``` в ```NavigationController```:

```swift
struct CartSUIView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> NavigationController {

        let cartScreen = CartScreen()
            .environmentObject(...)
        let suiCartVC = UIHostingController(rootView: cartScreen)
        
        let cartNav: NavigationController = .init(rootViewController: suiCartVC)
        return cartNav
    }

    func updateUIViewController(_ uiViewController: NavigationController, 
                                context: Context) {
    }
}
```

Дизайном нового таб-бара пока не стоит забивать голову, мы к этому еще вернемся, 
сначала необходимо добиться работоспособности текущих конструкций. 
```RootTabView``` приведем к максимально простому виду. Объявим ```enum``` с экранами:

```swift
enum TabType: Int {
    case main
    case catalog
    case search
    case profile
    case cart
}
```

Далее соберем ```RootTabView {...}```, используя иконки из [SF Symbols](https://developer.apple.com/sf-symbols/):

```swift
struct RootTabView: View {

    @State private var selectedTab: TabType = .main

    var body: some View {
        TabView(selection: $selectedTab) {
            main.tag(TabType.main)
            catalog.tag(TabType.catalog)
            search.tag(TabType.search)
            profile.tag(TabType.profile)
            cart.tag(TabType.cart)
        }
    }

    private var main: some View {
        MainSUIView()
            .tabItem {
                Label("Catalog", systemImage: "house")
            }
    }
		...
    private var profile: some View {
        ProfileSUIView()
            .tabItem {
                Label("Profile", systemImage: "person")
            }
    }

    private var cart: some View {
        CartSUIView()
            .tabItem {
                Label("Cart", systemImage: "cart")
            }
    }
}
```

Запускаем проект, видим, что переключение между табами работает:

<p align="center">
  <img src="sources/using TabView/1.gif" alt="">
  </p>
  
Вместе с тем, сломалась навигация на экранах SUI: любой дочерний экран открывается модально, 
появилась белая полоса в области safe area. Разберемся по порядку.

Если коротко и просто, то роутинг, доставшийся нам в наследство как легаси, 
представляет из себя enum из списка экранов для навигации и фабрику, 
где этот ```enum``` разруливается:

```swift
enum Route {
    case trackOrder
    case qrAccessCode
    case safari(String)
    ...
}
...
func route(to direction: Route, 
           from viewController: UIViewController? = nil, 
           previousScreen: AmplitudeScreenNames? = nil) {
    let viewController = previousScreen == .bottomSheet ? 
  					UIApplication.topViewController() : viewController

    switch direction {
    case .trackOrder(let id):
        self.trackOrder(id: id, from: viewController)
    case .qrAccessCode:
        self.showQRAccessCode(from: viewController)
    case .safari(let url):
        routeToSafari(url: url)
    ....
}
```

Если явно не указан view controller – экран, с которого переходишь, 
то по умолчанию берем top view controller (код показывать не буду, он легко гуглится). 
Как раз в этом и причина модального открытия любого окна. 
Top view controller в нашей схеме это уже не ```UINavigationController``` или ```UITabBarController```, а hosting view controller:

```
po topViewController
▿ Optional<UIViewController>
  ▿ some : <_TtGC7SwiftUI19UIHostingControllerV7Utkonos11RootTabView_: 0x139f2f640>
```

Раньше до navigation controllera-а можно было добраться следующим образом:

```
po (topViewController as? UITabBarController)?.selectedViewController
▿ Optional<UIViewController>
  ▿ some : <Utkonos.NavigationController: 0x12f882600>
```

Таким образом, в SUI-экраны теперь явно надо передавать ссылку на navigation controller, 
чтобы ее использовать при роутинге. Одним из способов это сделать является создание ```EnvironmentKey```: 

```swift
struct NavigationControllerKey: EnvironmentKey {
    static let defaultValue: UINavigationController? = nil
}

extension EnvironmentValues {
    var navigationController: NavigationControllerKey.Value {
        get {
            return self[NavigationControllerKey.self]
        }
        set {
            self[NavigationControllerKey.self] = newValue
        }
    }
}
```

Далее объявим ```@Environment``` переменную в SUI-экране:

```swift
struct CartScreen: View {
    ...
    @Environment(\.navigationController) var navigationController
    ...
  
    var body: some View {
    ...
    }
}
```

Заинжектим navigation controller непосредственно в момент создания hosting view controller-а экрана, 
таким образом код ```CartSUIView``` преобразуется к виду:

```swift
struct CartSUIView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> NavigationController {
        let cartNav: NavigationController
        
        let emptyView: UIViewController = UIHostingController(rootView: EmptyView())
        cartNav = NavigationController.init(rootViewController: emptyView)
        
        let cartScreen = CartScreen()
            .environment(\.navigationController, cartNav)
            .environmentObject(...)
        let suiCartVC = UIHostingController(rootView: cartScreen)
        
        cartNav.addChild(suiCartVC) // child here is a root
        cartNav.setNavigationBarHidden(true, animated: false)
        
        return cartNav
    }

    func updateUIViewController(_ uiViewController: NavigationController, 
                                context: Context) {
    }
}
```

Здесь стоит пояснить, что для инжектирования ```.environment(.navigationController, cartNav)```
необходим экземпляр объекта navigation controller-а cartNav, создадим его используя  проксирующий ```UIHostingController```
c пустым ```EmptyView```. 
Далее мы добавляем как child основной экран: ```cartNav.addChild(suiCartVC)```,
но при этом надо «заглушить» navigation bar от пустого view: ```cartNav.setNavigationBarHidden(true, animated: false)```.

Кроме этого, необходимо принудительно скрыть кнопку назад (на пустое view) на экране:

<p align="center">
  <img src="sources/using TabView/2.jpeg" alt="" width="300">
  </p>
  
Сделать это весьма просто, применив следующий модификатор:

```swift
struct CartScreen: View {
    
    ...
    var body: some View {
        content
            .navigationBarBackButtonHidden(true)
    }
    ...
}
```

Далее прокинем зависимость во все дочерние View экрана: 

```swift
@Environment(\.navigationController) var navigationController
```

Здесь стоит отметить, что если одно и то же view используется на 
экранах с разными экземплярами navigation controller-а 
(например, у нас переход на товар может быть как с главного экрана, так и с экрана корзины),
то благодаря дереву зависимостей SwiftUI ```@Enviroment``` значение у view будет браться именно родительского view, 
то есть ошибки не будет. 

Пример роутинга во view:

```swift
Button {
		Router.injected.routeToGoodsItem(goodsItemID: goods.id, 
                                 from: navigationController)
} label: { ... }
```

Теперь вернемся к белой полосе в области safe area, исправляется это весьма легко. Определим следующий модификатор:

```swift
public extension View {
    @ViewBuilder
    func expandViewOutOfSafeArea(_ edges: Edge.Set = .all) -> some View {
        if #available(iOS 14, *) {
            self.ignoresSafeArea(edges: edges)
        } else {
            self.edgesIgnoringSafeArea(edges) // deprecated
        }
    }
}
```

Применим его к контенту tabItem-ов:

```swift
private var main: some View {
    MainSUIView()
        .expandViewOutOfSafeArea()
        .tabItem {
            Label("Catalog", systemImage: "house")
        }
}
```

Запускаем приложение, видим, что проблемы ушли:

<p align="center">
  <img src="sources/using TabView/3.gif" alt="">
  </p>
  
___
## Верстаем таб-бар
Теперь можно приступить к верстке таб-бара. Модификатор ```tabItem(:)```, 
доступный из коробки, имеет весьма ограниченный функционал в верстке, 
поэтому если чего-то не хватает, надо кастомизировать. 

К счастью, SUI позволяет это сделать весьма легко: 

```swift
struct RootTabView: View {

    @State private var selectedTab: TabType = .main

    var body: some View {
        
        ZStack(alignment: Alignment.bottom) {
            TabView(selection: $selectedTab) {
                main.tag(TabType.main)
                catalog.tag(TabType.catalog)
                search.tag(TabType.search)
                profile.tag(TabType.profile)
                cart.tag(TabType.cart)
            }
            
            HStack(spacing: 0) {
                /*
                 Здесь будем верстать кнопки
                 таб-бара
                 */
            }
        }
    }
}
```

Как выглядят кнопки тап-бара в различных состояниях:

<p align="center">
  	<img src="sources/using TabView/4.png" alt="" width="100">
	<img src="sources/using TabView/5.png" alt="" width="100">
	<img src="sources/using TabView/6.png" alt="" width="100">
  </p>
*кнопка не активна, кнопка активна и активная кнопка с бэйджем*

Так как дизайн весьма простой, просто приведу код:

```swift
struct TabBarItem: View {
    
    @Environment(\.colorScheme) var colorScheme
    
    let icon: Image
    let title: String
    let badgeCount: Int
    let isSelected: Bool
    let itemWidth: CGFloat
    let onTap: () -> ()
    
    var body: some View {
        Button {
            onTap()
        } label: {
            VStack(alignment: .center, spacing: 2.0) {
                ZStack(alignment: .bottomLeading) {
                    Circle()
                        .foregroundColor(colorScheme == .dark ? ... )
                        .frame(width: 20.0, height: 20.0)
                        .opacity(isSelected ? 1.0 : 0.0)
                    ZStack {
                        icon
                            .resizable()
                            .renderingMode(.template)
                            .frame(width: 28.0, height: 28.0)
                            .foregroundColor(isSelected ? (colorScheme == .dark ? ...) : ...)
                        Text("\(badgeCount > 99 ? "99+" : "\(badgeCount)")")
                            .kerning(0.3)
                            .lineLimit(1)
                            .truncationMode(.tail)
                            .foregroundColor(Color.white)
                            .boldFont(11)
                            .padding(.horizontal, 4)
                            .background(Color.Button.primary)
                            .cornerRadius(50)
                            .opacity(badgeCount == 0 ? 0.0 : 1.0)
                            .offset(x: 16.0, y: -8.0)
                    }
                }
                Text(title)
                    .boldFont(12.0)
                    .foregroundColor(isSelected ? (colorScheme == .dark ? ...) : ... )
            }
            .frame(width: itemWidth)
        }
        .buttonStyle(.plain)
    }
}
```

Комментировать особо нечего, кроме того, что ```boldFont``` – кастомный модификатор для шрифта 
и что сознательно в свойства не вынесены цвета для модификаторов ```foregroundColor```, ```background```, 
так как других таких же кнопок, но с другими цветами, в приложении не будет, 
в ином случае, конечно, я рекомендовал бы это делать. 
Дополню, что в нашем проекте элементы дизайн-системы, 
к которым безусловно относится и кнопка таб-бара, вынесены в отдельный package. 


<p align="center">
  	<img src="sources/using TabView/7.png" alt="">
  </p>
  
Данный подход советую применить и у вас.
  
Посмотрим, как изменится ```RootTabView```:

```swift
struct RootTabView: View {
    @Environment(\.colorScheme) var colorScheme
    @State private var cartCount: Int = 0
    @State private var cartTitle: String = "Shopping cart".localized
    
    @State private var selectedTab: TabType = .main

    var body: some View {
        GeometryReader { geometry in
            ZStack(alignment: Alignment.bottom) {
                TabView(selection: $selectedTab) {
                    main.tag(TabType.main)
                    catalog.tag(TabType.catalog)
                    search.tag(TabType.search)
                    profile.tag(TabType.profile)
                    cart.tag(TabType.cart)
                }

                HStack(spacing: 0) {
                    TabBarItem(icon: Image.TabBar.home,
                               title: "Utkonos".localized,
                               badgeCount: 0,
                               isSelected: selectedTab ==  .main,
                               itemWidth: geometry.size.width / 5) {
                        selectedTab = .main
                    }
										...
                    TabBarItem(icon: Image.TabBar.cart,
                               title: cartTitle,
                               badgeCount: cartCount,
                               isSelected: selectedTab == .cart,
                               itemWidth: geometry.size.width / 5) {
                        selectedTab = .cart
                    }
                }
            }
        }
        .onCartChanged { count, price in
            ...
            cartTitle = price == 0 ? "Shopping cart".localized : price.stringCurrency
	    			cartCount = count
            ...
        }
    }

    private var main: some View {
        MainSUIView()
            .expandViewOutOfSafeArea()
    }
  	...
}
```

Дополнительно скажу, что к этому view применен модификатор ```onCartChanged```, 
который отлавливает события изменения корзины, реализация крайне проста: 
все строится вокруг отслеживания ```onReceive``` нужного события в NotificationCenter. 
В этом модификаторе и происходит изменение заголовка у кнопки с экраном корзины и бэйджа.

Запускаем проект:

<p align="center">
  	<img src="sources/using TabView/8.gif" alt="">
  </p>
  
Отлично!
Видим, что кнопки отрисованы правильно, изменение бэйджа и тайтла работает. 
Баг с поднятием кнопок таб-бара вместе с клавиатурой исправляем модификатором: ```ignoresSafeArea(.keyboard)```:
  
  ```swift
  struct RootTabView: View {
    ...
    var body: some View {
        GeometryReader { geometry in
            ZStack(alignment: Alignment.bottom) {
                TabView(selection: $selectedTab) {
                	...
                }

                HStack(spacing: 0) {
		   						...
                }
            }
        }.ignoresSafeArea(.keyboard)
    }
}
  ```
  ___
 ## Добавляем анимацию
 
Редко, когда дизайнер ограничивает себя только отрисовкой макетов кнопок, забыв об анимации. 
И действительно, ведь удачная анимация делает приложение удобным и привлекает внимание, но не отвлекает пользователя. 
Одна из задач анимации — повысить отзывчивость приложения.
Цель одной из предложенных к реализации анимаций в нашем таб-баре — дать визуальный фидбэк пользователю о том, 
что изменился состав корзины при добавлении новых товаров. 

<p align="center">
  	<img src="sources/using TabView/9.gif" alt="">
  </p>

Выглядит весьма стильно, давайте реализуем. 

Для начала нам необходим массив координат – смещений иконки и 
текущий индекс смещения в этом массиве:

```swift
struct RootTabView: View {
    ...
    private let offsets: [CGPoint] = [.init(x: 0, y: 0),
                                      .init(x: 0, y: 4),
                                      .init(x: 0, y: 0)]
    @State private var currentOffsetIndex: Int = 0

    var body: some View {
        ...
    }
}
```

В описанном выше модификаторе ```onCartChanged```, отслеживающем изменение состояния корзины будем изменять ```currentOffsetIndex``` 
в цикле по всему массиву ```offsets```:

```swift
struct RootTabView: View {
    ...
    private let offsets: [CGPoint] = [.init(x: 0, y: 0),
                                      .init(x: 0, y: 4),
                                      .init(x: 0, y: 0)]
    @State private var currentOffsetIndex: Int = 0

    var body: some View {
        content
        ...
            .onCartChanged { count, price in
                ...
                withAnimation {
                    for index in 1..<offsets.count {
                        Task.delayed(byTimeInterval: Double(index)/10) {
                            await MainActor.run {
                                currentOffsetIndex = index
                                
                                if index == 1 {
                                    cartCount = count
                                    cartTitle = price == 0 ? "Shopping cart".localized : price.stringCurrency
                                }
                            }
                        }
                    }
                }
                ...
            }
        ...
    }
    ...
}
```

Поясню, ```Task.delayed(byTimeInterval: ...)``` — это по сути то же, что и ```asyncAfter(deadline:execute)``` только в New Concurrency Model.

```swift
public extension Task where Failure == Error {
    @discardableResult
    public static func delayed(
        byTimeInterval delayInterval: TimeInterval,
        priority: TaskPriority? = nil,
        operation: @escaping @Sendable () async throws -> Success
    ) -> Task {
        Task(priority: priority) {
            let delay = UInt64(delayInterval * 1_000_000_000)
            try await Task<Never, Never>.sleep(nanoseconds: delay)
            return try await operation()
        }
    }
}
```

Внутри неизолированного контекста ```Task.delayed {…}``` 
мы оборачиваем в ```await MainActor.run {…}```, потому что получить доступ к ```@State``` свойствам можно только изнутри актора.

Теперь приступим к самому интересному – модификатору ```.offset``` в сочетании с ```spring```-анимацией.

```swift
.offset(x: offsets[currentOffsetIndex].x, 
        y: offsets[currentOffsetIndex].y)
    .animation(.spring(response: 0.15, 
                       dampingFraction: 0.75, 
                       blendDuration: 0), 
               value: currentOffsetIndex)
```

Где его применить? Очевидно, что смещаться должна сама иконка с бэйджем, то есть в ```TabBarItem```:

```swift
public struct TabBarItem: View {
    
    ...
    
    public var body: some View {
        Button {
            ...
        } label: {
            VStack(...) {
                ZStack(...) {
                    ...
                    ZStack {
                        icon
                            ...
                        Text(...)
                            ...
                    }
                    .offset(x: offsets[currentOffsetIndex].x,
                            y: offsets[currentOffsetIndex].y)
                    .animation(.spring(response: 0.15,
                                       dampingFraction: 0.75,
                                       blendDuration: 0),
                               value: currentOffsetIndex)
                }
                ...
            }
            ...
        }
        ...
    }
}
```

Но здесь есть нюанс, а что если потом дизайнер предложит добавить еще одну анимацию, 
уже не связанную со смещением. 

Давайте вынесем модификатор для анимации в параметр ```TabBarItem```, обернем в дженерик:

```swift
public struct TabBarItem<VModifier>: View where VModifier: ViewModifier {
    
    ...
    let animation: VModifier
    ...
    
    public init(...,
         animation: VModifier,
            ...) {
      	...
        self.animation = animation
      	...
    }
    
    public var body: some View {
        Button {
            onTap()
        } label: {
            VStack(alignment: .center, spacing: 2.0) {
                ZStack(alignment: .bottomLeading) {
                    ...
                    ZStack {
                        icon
                            .resizable()
                            ...
                        Text("\(badgeCount > 99 ? "99+" : "\(badgeCount)")")
                            ...
                    }
                    .modifier(animation)
                }
                ...
            }
            ...
        }
        ...
    }
}
```

Чтобы не пришлось ничего менять в коде тех tab item-ов, у которых не будет анимации, напишем ```extension```:

```swift
public extension TabBarItem where VModifier == EmptyModifier {
    public init(icon: Image,
         title: String,
         badgeCount: Int,
         isSelected: Bool,
         itemWidth: CGFloat,
         onTap: @escaping () -> ()) {
        self.icon = icon
        self.title = title
        self.badgeCount = badgeCount
        self.isSelected = isSelected
        self.itemWidth = itemWidth
        self.onTap = onTap
        self.animation = EmptyModifier()
    }
}
```

Теперь нам нужен модификатор, реагирующий на анимацию извне, чтобы передать его как параметр в  ```TabBarItem```. 
Раньше для таких целей был протокол [AnimatableModifier](https://developer.apple.com/documentation/swiftui/animatablemodifier), 
который Apple, недавно выпустив, немногим после назвала его устаревшим, 
взамен предложив использовать [Animatable](https://developer.apple.com/documentation/swiftui/animatable):

```swift
public struct OffsetAnimation<V>: Animatable, ViewModifier where V: Equatable {
    
    private var offset: CGPoint
    private var value: V
    
    public init(offset: CGPoint,
                value: V) {
        self.offset = offset
        self.value = value
    }
    
    public var animatableData: CGPoint {
        get { offset }
        set { offset = newValue }
    }
    
    public func body(content: Content) -> some View {
        content
            .offset(x: offset.x, y: offset.y)
            .animation(.spring(response: 0.15, 
                               dampingFraction: 0.75, 
                               blendDuration: 0), 
                       value: value)
    }
 
}
```

Стоит пояснить, что ```animatableData``` – это данные для анимации, в нашем случае как раз точка смещения.
Важный нюанс, Apple назвала устаревшим модификатор ```.animation(:)```, 
который подарил разработчикам [много багов с анимацией]((https://t.me/swiftui_dev/156)), взамен предложив 
использовать [animation(:value:)](https://developer.apple.com/documentation/swiftui/view/animation(_:value:)). 
Основный смысл последнего в том, чтобы проигрывать анимацию тогда, когда меняется конкретный ```value```. 
Поэтому наш ```OffsetAnimation``` и является дженериком, чтобы передавать этот ```value```, как параметр.

Таким образом ```RootTabView``` с анимированной кнопкой выглядит так:

```swift
struct RootTabView: View {
    ...
    private let offsets: [CGPoint] = [.init(x: 0, y: 0),
                                      .init(x: 0, y: 4),
                                      .init(x: 0, y: 0)]
    @State private var currentOffsetIndex: Int = 0

    var body: some View {
        GeometryReader { geometry in
            ZStack(alignment: Alignment.bottom) {
                TabView(selection: $selectedTab) {
                    ...
                    cart.tag(TabType.cart)
                }
                
                HStack(spacing: 0) {
                    ...
                    TabBarItem(icon: Image.TabBar.cart,
                               title: cartTitle,
                               badgeCount: cartCount,
                               isSelected: selectedTab == .cart,
                               itemWidth: geometry.size.width / 5,
                               animation: OffsetAnimation(offset: offsets[currentOffsetIndex],
                                                          value: currentOffsetIndex)) {
                        selectedTab = .cart
                    }
                }
            }
        }
        ...
            .onCartChanged { count, price in
                ...
                withAnimation {
                    for index in 1..<offsets.count {
                        Task.delayed(byTimeInterval: Double(index)/10) {
                            await MainActor.run {
                                currentOffsetIndex = index
                                
                                if index == 1 {
                                    cartCount = count
                                    cartTitle = price == 0 ? "Shopping cart".localized : price.stringCurrency
                                }
                            }
                        }
                    }
                }
                ...
            }
        ...
    }
    ...
}
```

Запустим проект, видим, что задумка дизайнера осуществлена:


<p align="center">
  	<img src="sources/using TabView/10.gif" alt="">
  </p>

___
## Заключение

Давайте теперь вынесем из ```RootTabView``` ```@State``` свойство ```selectedTab``` - выбранного экрана на таб баре:

```swift
struct RootTabView: View {
    ...
    @State private var selectedTab : TabType = .main
    ...
}
```

У нас в проекте мы придерживаемся архитектуры MVVM-S, где за роутинг отвечает соответствующий сервис, перенесем ```selectedTab``` в него:

```swift
final class Router : ObservableObject {
    ...
    @Published public var selectedTab: TabType = .main
    ...
    
    func openTabCart() {
        selectedTab = .cart
    }
    ...
}
```

```RootTabView``` преобразуется к виду:

```swift
struct RootTabView: View {
    ...
    @ObservedObject private var router: Router
    ...
    
    var body: some View {
        TabView(selection: $router.selectedTab) {
            ...
            cart.tag(TabType.cart)
        }
        
        ...
        TabBarItem(...) {
            router.openTabCart()
        }
        ...
    }
}
```

На этом у меня все, спасибо, что дочитали до конца!

Подписывайтесь на [мой Telegram-канал](https://t.me/swiftui_dev), посвященный iOS-разработке на SwiftUI.

