+++
title = "Detecting When an Animation Has Ended"
subtitle = "Perform actions on completion"
date = 2019-09-21T12:00:00Z
twitter = "diegolavalledev"
tags = ["swiftui", "animation"]
featured = true
example = "/examples/moonshot"

[[resources]]
  name = "cover"
  src = "screenshot.gif"

[[resources]]
  name = "feature"
  src = "demo.gif"
  title = "To the moon!"

+++

Whenever a SwiftUI animation is triggered, its state is updated immediately regardless of the duration. The Animation struct does not provide us with any sort of callback to indicate whether it has completed. So how can we detect when our views have stopped animating?

<!--more-->

Suppose we want to animate the image of a little plane flying towards the moon. We start by declaring a view for the plane, the moon background and a state variable indicating the current distance from the ground. We lay all of this out on our main view's body.

{{< highlight swift  >}}
struct PlaneMoonScene: View {
  
  @State var distance: CGFloat = 0

  var plane: some View {
    Image(systemName: "paperplane")
    …
  }

  var moonBackground: some View {
    Image(systemName: "moon.stars")
    …
  }

  var body: some View {
    ZStack(alignment: .bottomLeading) {
      moonBackground
      plane.offset(x: distance, y: -distance)
    }
  }
}
{{< / highlight >}}

To kick off our flight we need to define a _launch_ button. The button's action will animate the distance from 0 to 200 by applying the `easeInOut()` timing function for a total duration of 1 second.

{{< highlight swift  >}}
struct PlaneMoonScene: View {
  …
  var launchButton: some View {
    Button("Launch!") {
      withAnimation(.easeInOut(duration: 1)) {
        self.distance = 200
      }
    }
  }

  var body: some View {
    ZStack {
      …
      launchButton
    }
  }
}
{{< / highlight >}}

As expected pressing _Launch!_ causes our little plane to shoot up and reach the moon. But what if we want to display a congratulatory message once our destination has been attained? For this we need to replace the `offset()` modifier that places the plane over the background with our own _FlyModifier_.

{{< highlight swift  >}}
struct FlyModifier: AnimatableModifier {

  var totalDistance: CGFloat
  var percentage: CGFloat
  var onReachedDestination: () -> () = {}

  private var distance: CGFloat { percentage * totalDistance }

  …

  func body(content: Content) -> some View {
    content
    .offset(x: distance, y: -distance)
  }
}
{{< / highlight >}}

Our `FlyModifier` also positions the plane at a specified distance but because it inherits from `AnimatableModifier` it has the ability to look at the animation parameters as they transition from initial value to target. Our modifier is applied similarly to `offset()` except this time around we _do_  get a chance to provide a completion handler.

{{< highlight swift  >}}
struct PlaneMoonScene: View {

  @State var percentage: CGFloat = 0
  …

  var body: some View {
    …
    plane.modifier(
      FlyModifier(totalDistance: 200, percentage: percentage) {
        // We have reached the moon!
      }
    )
  }
}
{{< / highlight >}}

So how do we actually detect within `FlyModifier` that the total distance has been covered - or that the percentage value is 1, as in 100 percent of the distance? Well we need to implement the variable `animatableData` to comply with `AnimatableModifier` and in the setter of that variable we check for completion. If the animation has finished we asynchronously invoke the handler given to us by the main view. 

{{< highlight swift  >}}
struct FlyModifier: AnimatableModifier {
  …

  var animatableData: CGFloat {
    get { percentage }
    set {
      percentage = newValue
      checkIfFinished()
    }
  }

  func checkIfFinished() -> () {
    if percentage == 1 {
      DispatchQueue.main.async {
        self.onReachedDestination()
      }
    }
  }

  …
}
{{< / highlight >}}

Finally we can celebrate reaching the moon!

{{< highlight swift  >}}
struct PlaneMoonScene: View {

  @State var reachedMoon = false
  …

  var congrats: some View {
    Text("Congrats!!")
    …
  }

  var launchButton: some View {
    Button("Launch!") {
      withAnimation(.easeInOut(duration: 1)) {
        self.percentage = 1
      }
    }
  }

  var body: some View {
    ZStack {
      moonBackground
      plane.modifier(
        FlyModifier(totalDistance: 200, percentage: percentage) {
          withAnimation { self.reachedMoon.toggle() }
        }
      }
      launchButton
      if reachedMoon {
        congrats
      }
    }
  }
}
{{< / highlight >}}

For sample code featuring this and other techniques please checkout our [working examples](https://github.com/swift-you-and-i/working-examples/tree/master/Sources/WorkingExamples/animation-ended) repo.
