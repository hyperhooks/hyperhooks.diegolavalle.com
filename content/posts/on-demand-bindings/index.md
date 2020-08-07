+++
title = "On-demand bindings"
date = 2020-01-19T12:00:00Z
twitter = "diegolavalledev"
tags = ["binding", "state", "toggle", "swiftui", "array"]
featured = false
example = "/examples/toggles"

[[resources]]
  name = "cover"
  src = "screenshot.png"
+++

Bindings are used to pass parts of our state down to a subview that might later on make changes to said state. We typically rely on the `@State` and `@Binding` property wrappers to generate bindings for us but bindings can also be constructed programmatically. In fact there's a handful of cases in which creating bindings in an _on-demand_  manner may just be the preferred option.

<!--more-->

Like most UI controls in SwiftUI, _Toggle_ is backed by a binding. In this case the initializer argument _isOn_ of type `Binding<Bool>` informs the toggle whether it is _on_ or _off_ but most importantly, it also allows the control to alter the value of this variable from the inside. Now imagine that we have -as part of our state- a whole set of toggable models which we want represented in our user interface as individual toggle controls.

{{< highlight swift  >}}
struct Toggles: View {

  struct Model: Identifiable {
    var id: String // Also used as label
    var on = false
  }

  @State var items = [
    Model(id: "foo"),
    Model(id: "bar"),
    Model(id: "baz"),
  ]

  var body: some View { … }
}
{{< / highlight >}}

We can leverage SwiftUI's auto-generated bindings and reference the `$items` property which is bound to the `items` state variable. Given that _ForEach_ can only iterate over sequences -not bindings of sequences- we need initialize it with `items.indices` instead. Each index gives us direct access to both an element of our state -a `Model` instance- and its corresponding binding. 

{{< highlight swift  >}}
var body: some View {
  VStack {
    ForEach(items.indices) {
      Toggle(
        self.items[$0].id, // Label
        isOn: self.$items[$0].on // Binding
      )
    } // ForEach
  } // VStack
} // body
{{< / highlight >}}

This works fine except for one thing: the specification of `ForEach` warns us that iterating over ranges only applies to constant data. Now as far as our view is concerned, the element count for `items` _could_ change overtime since the whole array is tagged with `@State` on its declaration. Let's suppose we want to enable the possibility of removing a toggle altogether.

{{< highlight swift  >}}
ForEach(…) {
  HStack {
    Toggle(…)
    Button("remove") {
      // TODO: remove item from items array
    }
  }
}
{{< / highlight >}}

Because we made our `Model`  _struct_ comply with the `Identifiable` protocol it's easy iterate over our items. But we still need a binding for _Toggle_ to work with. We can instantiate the binding _on-the-fly_ provided that we always use up-to-date indices to look up for items inside our array.

{{< highlight swift  >}}
…
func makeBinding(_ item: Model) -> Binding<Bool> {
  let i = self.items.firstIndex { $0.id == item.id }!
  return .init(
    get: { self.available[i].on },
    set: { self.available[i].on = $0 }
  )
}

var body: some View {
  …
  ForEach(items) { item in
    HStack {
      Toggle(
        item.id,
        isOn: self.makeBinding(item)
      )
      Button("remove") {
        self.items.removeAll { $0.id == item.id }
      }
    }
  }
  …
}
{{< / highlight >}}

We call the `makeBinding` function every time a toggle is rendered. The function first finds the current index for the specified item and uses it to produce both the _getter_ and _setter_ that make up the binding object. The _remove_ button simply mutates our _items_ array normally since it's declared as part of the view's state. 

{{< figure src="mac.png" title="macOS version of the example" >}}

Check out the associated _Working Example_ to see this technique in action.
