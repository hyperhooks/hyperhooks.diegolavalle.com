+++
title = "You Don't Need UIImage"
date = 2019-09-07T12:00:00Z
twitter = "diegolavalledev"
tags = ["swiftui", "swift", "ui", "ios", "apple"]
featured = false
example = "/examples/none"

[[resources]]
  name = "cover"
  src = "screenshot.pngf"

[[resources]]
  name = "icons"
  src = "icons.gif"
+++

In _SwiftUI_ `Image` view provides a way to initialize it using UIKit's `UIImage`. This was used to work-around some [early issues](https://forums.developer.apple.com/thread/119331) with Swift UI which prevented [custom symbols](https://developer.apple.com/documentation/uikit/uiimage/creating_custom_symbol_images_for_your_app) to be scaled and colored.

<!--more-->

Here's how we display, color and scale a built-in image: 

{{< highlight swift >}}
Image(systemName: "heart.circle")
  .resizable()
  .aspectRatio(contentMode: .fit)
  .foregroundColor(.red)
{{< / highlight >}}

And here's how we _used to_ display a custom SVG image using `UIImage` as intermediary: 

{{< highlight swift  >}}
  Image(uiImage: UIImage(named: "heart.circle.custom")!)
    .resizable() // Dynamic sizing
    .aspectRatio(contentMode: .fit)
    .foregroundColor(.red)
{{< / highlight >}}

Except that it would not take the foreground color and the scaling would be done by resampling a bitmap producing a low resolution result.

{{< figure src="icons.png" title="Ways of displaying custom symbols" width="250" >}}

To work around these issue we would wrap the image in a dummy `Button` view, which would take care of the color. Then we would have to use a font size to scale the vector image in high resolution, like so:

{{< highlight swift  >}}
Button(action: {}) { // Dummy button
  Image(uiImage: UIImage(named: "heart.circle.custom")!
    .withConfiguration(
        // Fixed size in points
        UIImage.SymbolConfiguration(pointSize: 100)
    )
  )
}
.foregroundColor(.red) // Color applied to button
{{< / highlight >}}

But this still poses the problem of having to specify a static size. The solution is quite simple: avoid the use of `UIImage` and use the standard _fixed_ method.

{{< highlight swift  >}}
Image("heart.circle.custom")
  .resizable()
  .aspectRatio(contentMode: .fit)
  .foregroundColor(.red)
{{< / highlight >}}

Works like a charm!

See below for a result comparison of each of the methods described on this post. 

For sample code featuring this and other techniques please checkout our [working examples](https://github.com/swift-you-and-i/working-examples/tree/master/Sources/WorkingExamples/) repo.
