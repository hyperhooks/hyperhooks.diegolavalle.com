+++
title = "Pausing and Reversing SwiftUI Animations"
date = 2020-08-12T12:00:00Z
twitter = "diegolavalledev"
tags = ["ios", "swift", "swifui", "combine", "ui"]
featured = false
example = "/examples/count-down-up"

[[resources]]
  name = "cover"
  src = "cover.png"
+++

Animations and transitions are decisively one of _SwiftUI_’s fortes. The framework allows us to go as deep as we want in handling specifics like timing function and duration parameters. Moreover it gives us sophisticated tools like the [Animatable](https://developer.apple.com/documentation/swiftui/animatable) protocol to completely customize behavior. Here we are going to rely on the more constricted [AnimatableModifier](https://developer.apple.com/documentation/swiftui/animatablemodifier) protocol to show how we can manually pause and resume and animation as well as adding the ability to reverse and loop over.

<!--more-->

# A Boring Animation

To focus on function rather than visuals we are going to design what is probably the simplest animation possible: a counter. We will display a number on screen which can be counting up or down within a range like some kind of stopwatch.

{{< figure src="counter.png" title="Simple stopwatch-like UI" >}}

We’ll start by creating a basic _ViewModifier_ containing a numeric value formatted to display as a zero-padded positive integer. The value is calculated as a percentage over a maximum which is taken as parameter.

{{< highlight swift  >}}
struct CountModifier: ViewModifier {

  var maxValue: CGFloat // Maximum count value
  var timeDuration: Double // How long it will take to reach max, in seconds

  private let percentValue: CGFloat = 0 // Percentage of the maximum count

  // Counter value as integer
  var value: Int {
    Int(percentValue * maxValue)
  }

  func body(content: Content) -> some View {
    Text("\(value, specifier: "%03d")") // Formatted count
    .font(.system(.largeTitle, design: .monospaced))
    .font(.largeTitle)
  }
}
{{< / highlight >}}

Notice how in this case we are simply ignoring the input view and returning a fresh body. So we are not really modifying anything but rather replacing it entirely. Additionally we added a `timeDuration` argument which will be useful to customize the duration of the animation.

# Adding External Controls

Outside of the numeric display we want to show a set of controls for pausing and reversing the count. These can be regular buttons which modify state variables defined on the main content view.

{{< highlight swift  >}}
struct CountDownUp: View {

  @State var isCounting = false // Whether we are counting (moving)
  @State var isReversed = false // The direction of the count: up or down
  
  var body: some View {
    VStack {
      EmptyView()
      .modifier(
        CountModifier(
          maxValue: 100, // We'll count to 100
          timeDuration: 10 // In 10 seconds
        )
      )
      HStack {
        Button(isCounting ? "⏸" : "▶️") {
          isCounting.toggle() // Pause / resume
        }
        Button(isReversed ? "🔽" : "🔼") { 
          isReversed.toggle()  // Count up / down
        }
      }
      .frame(maxWidth: .infinity, maxHeight: .infinity, alignment: .top)
    }
  }
}
{{< / highlight >}}

As an added touch we made the buttons adapt to the current status of the counter thus providing an instant feedback.

# Communicating with the Modifier

Because the animation will change depending on the state determined by our external controls we'll necessarily have to pass these variables as _bindings_:

{{< highlight swift  >}}
fileprivate struct CountModifier: ViewModifier {

  var maxValue: CGFloat // Maximum count value
  var timeDuration: Double
  @Binding var percentage: CGFloat
  @Binding var isCounting: Bool
  @Binding var isReversed: Bool

  private var percentValue: CGFloat // Percentage of maximum count

  init(
    maxValue: CGFloat,
    timeDuration: Double,
    percentage: Binding<CGFloat>,
    isCounting: Binding<Bool>,
    isReversed: Binding<Bool>
  ) {
    self.maxValue = maxValue
    self.timeDuration = timeDuration
    _percentage = percentage // Bindings initialization 
    _isCounting = isCounting
    _isReversed = isReversed
    // The percent value is copied  from the binding, later on it will be used for the animation
    percentValue = percentage.wrappedValue
  }

  var value: Int {
    Int(percentValue * maxValue)
  }

  func body(content: Content) -> some View {
    Text("\(value, specifier: "%03d")")
    .font(.system(.largeTitle, design: .monospaced))
    .font(.largeTitle)
  }
}

{{< / highlight >}}

We also included a binding to a `percentage` which indirectly initializes the internal `percentValue` variable by copying its current value. Remember this percentage drives the number on display at any given time during the course of the animation.

The reason why this piece of state needs to be declared outside of the modifier will be revealed momentarily. For now let's just add `percentage` to our root view:

{{< highlight swift  >}}
struct CountDownUp: View {

  @State var percentage = CGFloat(0)
  …
  
  var body: some View {
    VStack {
      EmptyView()
      .modifier(
        CountModifier(
          …
          percentage: $percentage,
          …
        )
      …
{{< / highlight >}}

Great! We now have state shared between controls and display but there is still no actual animation happening.

# Animating the Counter

The way _AnimatableModifier_ works, in order to animate a certain value we need to declare a writable computed property named `animatableData` backed by some stored property. To turn `CountModifier` into an _AnimatableModifier_ we'll base our animatable data on the aforementioned percentage value.

{{< highlight swift  >}}
struct CountModifier: AnimatableModifier {

  var animatableData: CGFloat {
    get { percentValue }
    set { percentValue = newValue }
  }
  …
{{< / highlight >}}

Apart from needing to use `CGFloat` as opposed to regular double/single precision floating point scalars which is required by the protocol, the implementation of the property is as trivial as it gets.

# Triggering the Animation

To kick-start the counter we will be adding an `onChange` handler to watch over the contents of `isCounting`. This is an _iOS 14+_ only feature. In the accompanying source code (see working example) you'll find a work-around for _iOS 13_ that uses  `PassthroughSubject` to send the start signal.

{{< highlight swift  >}}
struct CountModifier: AnimatableModifier {
  …
  func body(content: Content) -> some View {
    Text("\(value, specifier: "%03d")")
    …
    .onChange(of: isCounting) { _ in
      handleStart()
    }
  }

  func handleStart() -> () {
    if isCounting {
      withAnimation(.linear(duration: timeDuration)) {
        self.percentage = 1
      }
    }
  }
  …
{{< / highlight >}}

We handle `isCounting` being set by using that binding directly to set off the animation.

# Pausing and Resuming

{{< figure src="example.gif" title="Counter in action" >}}

Now in order to pause the animation we need to enrich our change handler for the `isCounting` variable as it now can switch from on to off. We will also introduce a `timeRemaining` property that will tell us exactly how long we have left on the full duration of the count. Finally for the sake of semantics we'll rename  `handleStart` into the more descriptive  `handleStartStop`:

{{< highlight swift  >}}
struct CountModifier: AnimatableModifier {
  …
  var timeRemaining: Double {
    timeDuration * Double(1 - percentValue)
  }

  func body(content: Content) -> some View {
    Text("\(value, specifier: "%03d")")
    …
    .onChange(of: isCounting) { _ in
      handleStartStop()
    }
  }

  func handleStartStop() -> () { // Formerly handleStart()
    if isCounting {
      withAnimation(.linear(duration: timeRemaining)) {
        self.percentage = 1
      }
    } else {
      withAnimation(.linear(duration: 0)) {
        self.percentage = percentValue
      }
    }
  }
  …
{{< / highlight >}}

The thing to notice here is that this time remaining approach only works for linear animations. More advanced timing functions would absolutely complicate the calculation that enables us to resume movement after pausing.

# Looping over

It would be nice if our counter doesn’t just stop when it reaches the limit. For this we are forced into hijacking `body()` to check whether the percentage has reached its maximum value of 1, in which case we'll force it back to zero.

{{< highlight swift  >}}
struct CountModifier: AnimatableModifier {
  …
  func body(content: Content) -> some View {
    // Loop when we reach completion
    if percentValue == 1 {
      DispatchQueue.main.async {
        self.percentage = 0
        withAnimation(.linear(duration: self.timeDuration)) {
          self.percentage = 1
        }
      }
    }
    return actualBody(content) // Original body of the view
  }
  
  // Moved the original body into a separate function
  func actualBody(_ content: Content) -> some View {
    Text("\(value, specifier: "%03d")")
    …
  }
  …

{{< / highlight >}}

After resetting the percentage we'll kick it off again with an asynchronous call. To keep things clean we'll move the previous `body` function into  `actualBody` to keep taking advantage of the enhanced _ViewBuilder_ syntax.

# Shift into Reverse

Whether the counting is on or now, we want to be able to change the direction. For this we monitor `isReversed` for changes and handle the new value using the same technique as before. We'll also need to alter the start/stop and the loop logic to take into consideration the current direction. 

{{< highlight swift  >}}
struct CountModifier: AnimatableModifier {
  …
  var timeRemaining: Double {
    isReversed
    ? timeDuration * Double(percentValue)
    : timeDuration * Double(1 - percentValue)
  }

  func body(content: Content) -> some View {
    if (isReversed && percentValue == 0) || (!isReversed && percentValue == 1) {
      DispatchQueue.main.async {
        self.percentage = self.isReversed ? 1 : 0
        withAnimation(.linear(duration: self.timeDuration)) {
          self.percentage = self.isReversed ? 0 : 1
        }
      }
    }
    return actualBody(content)
  }
  
  func actualBody(_ content: Content) -> some View {
    Text("\(value, specifier: "%03d")")
    …
    .onChange(of: isReversed) { _ in
      handleReverse()
    }
  }

  func handleStartStop() -> () {
    if isCounting {
      withAnimation(.linear(duration: timeRemaining)) {
        self.percentage = isReversed ? 0 : 1
      }
    } else { … }
    }
  }

  func handleReverse() -> () {
    if isCounting {
      withAnimation(.linear(duration: 0)) {
        self.percentage = percentValue
      }
      withAnimation(.linear(duration: timeRemaining)) {
        self.percentage = isReversed ? 0 : 1
      }
    }
  }
}
{{< / highlight >}}

Notably the time remaining was re-calculated by pondering the percentage or its inverse in the case of counting downwards. 

# Final Thoughts

This is a fun way of gaining control over events within a linear animation. It's definitely not the only one and might not even the preferred way as it does involve some _hacks_. But on the other hand it has as an advantage the simplicity of not having to deal with the _Animatable_ protocol directly.

That's it, as always check out the associated _Working Example_ for the complete source code and interactive demo.
