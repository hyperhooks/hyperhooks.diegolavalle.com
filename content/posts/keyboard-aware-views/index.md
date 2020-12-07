+++
title = "Keyboard-aware views"
subtitle = "Learn how to make room for the software keyboard"
date = 2020-01-12T12:00:00Z
twitter = "diegolavalledev"
tags = ["keyboard-aware", "software-keyboard", "form", "text-field", "combine"]
featured = false
example = "/examples/all-rise"

[[resources]]
  name = "cover"
  src = "screenshot.png"
+++

Sometimes we want to give our views a chance to react to the software keyboard being pulled up. This could be particularly useful in forms with text input where the keyboard might cover the very field we are typing into. We can solve this in SwiftUI with a little help from some old friends.

<!--more-->

UIKit's `UIResponder` is an interface that provides notifications for a variety of UI-related events, including keyboard events. By subscribing to  `keyboardDidShowNotification` we get the timely information we need via `keyboardFrameEndUserInfoKey` which contains the keyboard frame.

We'll start by creating an `ObservableObject` containing the current keyboard frame. There can only be one software keyboard so we'll make our class a _singleton_. The `frame` property is published so that we can subscribe to it from our views.

{{< highlight swift  >}}
import Foundation
import UIKit
import CoreGraphics
import Combine

class KeyboardProperties: ObservableObject {
  
  static let shared = KeyboardProperties()
  
  @Published var frame = CGRect.zero

  var subscription: Cancellable?

  init() {
    subscription = …
  }
}
{{< / highlight >}}

Now to populate our `frame` property we'll need to tap into `NotificationCenter`. In particular we are interested in the following `UIResponder` events: `keyboardDidShowNotification` and `keyboardDidHideNotification`. We'll use the _Combine_ framework to filter out the information we want from these notifications - namely the keyboard frame - and assign it to the corresponding instance property.

{{< highlight swift  >}}
subscription = NotificationCenter.default
  .publisher(for: UIResponder.keyboardDidShowNotification)
  .compactMap { $0.userInfo }
  .compactMap {
    $0[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect
  }
  .merge(
    with: NotificationCenter.default.publisher(for: UIResponder.keyboardDidHideNotification).map { _ in
      CGRect.zero
    }
  )
  .assign(to: \.frame, on: self)
{{< / highlight >}}

For demonstration purposes we are going to create a simple _SwiftUI_ view based on a vertical stack with a text field at the bottom. Without any additional logic, the text field will be covered by the rising software keyboard.

{{< highlight swift  >}}
@State var textValue = ""

var body: some View {
  VStack {
    // Pushes TextField to the bottom of view
    Spacer()

    // Text field would be covered by keyboard
    TextField("Enter some text", text: $textValue)
  }
}
{{< / highlight >}}

To help us we can now bring in our `KeyboardProperties` singleton using the `@ObservedObject` property wrapper. Since we're specifically interested in the keyboard's height, we'll put that in a calculated `kbHeight` property. Finally, we use this height to offset our text field, so that it is no longer covered up by the keyboard.

{{< highlight swift  >}}
…

@ObservedObject var keyboardProps = KeyboardProperties.shared

var kbHeight: CGFloat {
  keyboardProps.frame.height
}

var body: some View {
  VStack {
    …
    TextField("Enter some text", text: $textValue)
      // Text field will now be pushed up by the keyboard
      .offset(y: -kbHeight)
      // A little animation to soothe things up
      .animation(.easeIn(duration: 0.2))
  }
}
{{< / highlight >}}

That's it, we have successfully made our form _keyboard-aware_ solving a serious user experience issue. Please check out the associated _Working Example_ to see this technique in action.
