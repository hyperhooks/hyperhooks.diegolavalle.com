+++
title = "Codable observable objects"
subtitle = "Integrate your models with Combine"
date = 2020-01-05T12:00:00Z
twitter = "diegolavalledev"
tags = ["combine", "observable", "encode", "decode", "json"]
featured = false
example = "/examples/realtime-json"

[[resources]]
  name = "cover"
  src = "screenshot.png"
+++

Complying to the `Codable` protocol is simple thanks to synthesized initializers and coding keys. Similarly making your class observable using the _Combine_ framework is trivial with `ObservableObject`. But attempting to merge these two protocols in a single implementation poses a few obstables. Let's find out!

<!--more-->

A simple class can be made `Encodable` and `Decodable` simultaneously by simply declaring it as `Codable`.

{{< highlight swift  >}}
class CodableLandmark: Codable {

  var site: String = "Unknown site"
  var visited: Bool = false
}
{{< / highlight >}}

After this you can easily convert objects to _JSON_ format and back.

{{< highlight swift  >}}
import Foundation

class CodableLandmark: Codable {
  …
  
  // Encode this instance as JSON data
  var asJsonData: Data? {
    try? JSONEncoder().encode(self)
  }
  
  // Create an instance from JSON data
  static func fromJsonData(_ json: Data) -> Self? {
    guard let decoded = try? JSONDecoder().decode(Self.self, from: json) else {
      return nil
    }
    return decoded
  }
}
{{< / highlight >}}

An identical class could alternatively serve as a model for some _SwiftUI_ view simply by adhering to `ObservableObject`. Annotating properties with the `@Published` wrapper will ensure that notifications are generated when these values change.

{{< highlight swift  >}}
import Combine

class ObservableLandmark: ObservableObject {

  @Published var site: String = "Unknown site"
  @Published var visited: Bool = false
}
{{< / highlight >}}

Initially just adding the `ObservableObject` to our codable class' protocol list produces no error. But when we try to mark the first variable as _published_ we encounter error messages `Type 'LandmarkModel' does not conform to protocol 'Decodable'` and  `Type 'LandmarkModel' does not conform to protocol 'Encodable'`  error.

{{< highlight swift  >}}
class LandmarkModel: Codable, ObservableObject {

  @Published // Error
  var site: String = "Unknown site"
  
  @Published // Error
  var visited: Bool = false
}
{{< / highlight >}}


The solution is to simply implement some of the  `Codable` requirements manually rather than have them synthesized. Namely the initializer (for decoding) and the `encode` function for encoding. This forces us to additionally declare our coding keys explicitly.

{{< highlight swift  >}}
class LandmarkModel: Codable, ObservableObject {
  …

  // MARK: - Codable
  enum CodingKeys: String, CodingKey {
    case site
    case visited
  }

  // MARK: - Codable
  required init(from decoder: Decoder) throws {
    let values = try decoder.container(keyedBy: CodingKeys.self)
    site = try values.decode(String.self, forKey: .site)
    visited = try values.decode(Bool.self, forKey: .visited)
  }

  // MARK: - Encodable
  func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    try container.encode(site, forKey: .site)
    try container.encode(visited, forKey: .visited)
  }
}
{{< / highlight >}}

This approach of tightly coupling the codable and observable models can be useful for simple projects as it avoids a lot of boilerplate code.

Check out the associated _Working Example_ to see this technique in action.
