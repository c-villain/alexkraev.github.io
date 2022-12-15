# Navigation 

Hey guys! I’ve created [test project](https://github.com/c-villain/NaviBackportExample) with sample of 
new navigation API back ported in older SwiftUI versions.
 
One of the main question at every iOS-interview is app architecture. 
At the same time routing in the app almost always is the side theme of discussion. 
Moreover there are different approaches in the navigation in general and all of them have their own patterns such as navigator or coordinator.

Starting from SwiftUI 1.0 in almost every WWDC Apple talks about working with MVVM as if forgetting about routing. 
Yes, we were introduced to  [`NavigationView`](https://developer.apple.com/documentation/swiftui/navigationview), 
[`NavigationLink`](https://developer.apple.com/documentation/swiftui/navigationlink). 
But the feeling that Apple again has presented intermediate solution does not leave us. 
Some developers started to create their own wrappers over this API for convenience. 
Finally in iOS 16 Apple introduced the long-awaited new navigation API. 

Instead of [`NavigationView`](https://developer.apple.com/documentation/swiftui/navigationview) (starting with iOS 16 it has been deprecated) 
we have to use [`NavigationStack`](https://developer.apple.com/documentation/swiftui/navigationstack). 
Now the [`navigationDestination(for:destination:)`](https://developer.apple.com/documentation/swiftui/list/navigationdestination(for:destination:)) 
modifier defines a view to display. 

Frankly speaking many teams are still using UIKit routing in SwiftUI projects.
Even those who tried to understand [`NavigationView`](https://developer.apple.com/documentation/swiftui/navigationview), gave up and went back to UIKit. 
With the release of the new navigation API, 
this approach is a wrong turn.
Apart from that the new API requires a minimum deployments as iOS 16.0. 
So what to do? Use a [backport](https://github.com/johnpatrickmorgan/NavigationBackport)! 

Try [my sample](https://github.com/c-villain/NaviBackportExample)! 
You may create your own test project to try to work with [this package](https://github.com/johnpatrickmorgan/NavigationBackport) 
using it.
