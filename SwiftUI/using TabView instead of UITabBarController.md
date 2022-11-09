# Moving to SwiftUI: using TabView instead of UITabBarController.
___

Converting UIKit project to SwiftUI (or SUI) in 2022 is no longer a matter of time. 
It rather depends on availability of appropriate skills. 
I’ve been working for [Utkonos](https://www.utkonos.ru/) — one of the leaders of e-commerce in the Russian market. 
We started to develop the [application](https://apps.apple.com/ru/app/id582201863) using SUI at the end of 2020 when we picked iOS 13 as the lowest supported version (yes, we’ve decided not to wait for the iOS 14). 
This was also triggered by the task of a complete redesign of the application. 
At the moment, we have implemented two main screens out of five on SUI.
One of the main tasks facing iOS-developers is to implement navigation flow. 
It’s now rare to find a single-page application. 
The tab bars are used in iOS to support user interfaces where multiple screens can be accessed in no particular order. 
If the application is developed using SUI with no legacy code at all, then the typical development pattern is still the following: 
the screens are made up with SUI but the tab bar with UIKit. 
With the growth of the SUI-code in Utkonos, we began to gradually abandon navigation on UIKit, 
a big step in this direction was to convert the tab bar to the TabView instead of the UITabBarController.
___
Hello everyone! My name is [Alexander Kraev](http://linkedin.com/in/alex-kraev)! 
In this article I want to share with you my own experience of adding TabView with all the pitfalls: when you have screens developed 
both using SUI and UIKit. 
This article is not for those who have just started to learn SUI. 
If you are just like that, then I advise you to begin with some small features. 
You can find more interesting posts on [my telegram channel](https://t.me/swiftui_dev) dedicated to iOS development on SwiftUI.
___
### Preparing the structure
In our team we work using [Trunk Based Development](https://trunkbaseddevelopment.com/) (TDD). 
If you aren’t familiar with this version control management practice then I would advise you 
to watch [this session](https://www.youtube.com/watch?v=ykZbBD-CmP8). 
In one word, the development goes through Feature flags and toggles.
Let’s get start with adding a toggle for a new Tab Bar on SUI:

```swift
struct SwiftUI {
    struct TabView {
        static var isActive: Bool = true
    }
}
```

In the part of the code where the root view controller is created for the main window we should write:

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
```setupTabBarController()``` is the function of creating the tab bar using UIKit, and ```RootTabView()``` is a view of the tab bar 
on SUI integrated through ```UIHostingViewController```

The flow of UIKit’s tab bar is quite familiar: navigation controller is created with the root view controller for each screen:

```swift
let profileVC: ProfileViewController = .init()
let profileNav: NavigationController = .init(rootViewController: profileVC)
```

First view controllers (used as navigation controllers) are created. Then, the tab bar controller is initialized:

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

As you can see above ```NavigationController``` is a class inherited from ```UINavigationController``` but with custom behavior 
of the navigation bar, its appearance of the back button and as simple as that.
Let’s go back to the SUI Tab Bar. Obviously, ```RootTabView()``` will consist of child screen views. It’s time to start writing 
```UIViewControllerRepresentable``` wrappers for view controllers on UIKit. I’ll give you an example of one for the user profile screen:

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

As I said earlier, we have two screens fully developed using SUI. 
In order not to break the routing between these screens and other legacy screens developed on UIKit, 
we decided to wrap them also using ```UIViewControllerRepresentable``` in ```NavigationController```:

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

You don’t have to think about the design of the tab bar yet, we will come back to this later. 
First it’s necessary to get the efficiency of the current structures. 
Let’s implement ```RootTabView``` in the simplest possible way. Let’s declare an enum with screens:

```swift
enum TabType: Int {
    case main
    case catalog
    case search
    case profile
    case cart
}
```

Next, let’s design ```RootTabView {...}``` using icons from SF Symbols:

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

Let’s launch the project for test to see how switching between tabs works:

