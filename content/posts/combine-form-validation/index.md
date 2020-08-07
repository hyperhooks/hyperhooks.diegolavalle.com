+++
title = "Form Validation with Combine"
date = 2019-09-14T12:00:00Z
twitter = "diegolavalledev"
tags = ["swiftui", "combine", "form", "validation"]
featured = false
example = "/examples/fake-signup"

[[resources]]
  name = "cover"
  src = "screenshot.gif"

+++

In _WWDC 2019_ session [Combine in Practice](https://developer.apple.com/videos/play/wwdc2019/721) we learned how to apply the _Combine Framework_ to perform validation on a basic sign up form built with _UIKit_. Now we want to apply the same solution to the _SwiftUI_ version of that form which requires some adaptation.

<!--more-->

We begin by declaring a simple form model **separate** from the view…

{{< highlight swift  >}}
class SignUpFormModel: ObservableObject {
  @Published var username: String = ""
  @Published var password: String = ""
  @Published var passwordAgain: String = ""
}
{{< / highlight >}}

And link each property to the corresponding `TextField` control…

{{< highlight swift  >}}
struct SignUpForm: View {

  @ObservedObject var model = SignUpFormModel()

  var body: some View {
    …
    TextField("Username", text: $model.username)
    …
    TextField("Password 'secreto'", text: $model.password)
    …
    TextField("Password again", text: $model.passwordAgain)
    …
{{< / highlight >}}

Now we can begin declaring the publishers in our `SignUpFormModel`. First we want to make sure the password has more than six characters, and that it matches the confirmation field. For simplicity we will not use an error type, we will instead return `invalid` when the criteria is not met…

{{< highlight swift  >}}
var validatedPassword: AnyPublisher<String?, Never> {
  $password.combineLatest($passwordAgain) { password, passwordAgain in
    guard password == passwordAgain, password.count > 6 else {
      return "invalid"
    }
    return password
  }
  .map { $0 == "password" ? "invalid" : $0 }
  .eraseToAnyPublisher()
}
{{< / highlight >}}

For the user name we want to simultate an asynchronous network request that checks whether the chosen moniker is already taken…

{{< highlight swift  >}}
func usernameAvailable(_ username: String, completion: @escaping (Bool) -> ()) -> () {
  DispatchQueue.main .async {
    if (username == "foobar") {
      completion(true)
    } else {
      completion(false)
    }
  }
}
{{< / highlight >}}


As you can see, the only available name in our fake server is _foobar_.

We don't want to hit our API every second the user types into the name field, so we leverage `debounce()` to avoid this…

{{< highlight swift  >}}
var validatedUsername: AnyPublisher<String?, Never> {
  return $username
    .debounce(for: 0.5, scheduler: RunLoop.main)
    .removeDuplicates()
    .flatMap { username in
      return Future { promise in
        usernameAvailable(username) { available in
          promise(.success(available ? username : nil))
        }
      }
  }
  .eraseToAnyPublisher()
}
{{< / highlight >}}

Now to make use of this publisher we need some kind of indicator next to the text box to tell us whether we are making an acceptable choice. The indicator should be backed by a private `@State` variable in the view and outside the model.

To connect the indicator to the model's publisher we leverage the `onReceive()` modifier. On the completion block we manually update the form's current state…

{{< highlight swift  >}}
Text(usernameAvailable ? "✅" : "❌")
.onReceive(model.validatedUsername) {
  self.usernameAvailable = $0 != nil
}
{{< / highlight >}}


An analog indicator can be declared for the password fields.

Finally, we want to combine our two publishers to create an overall validation of the form. For this we create a new publisher…

{{< highlight swift  >}}
var validatedCredentials: AnyPublisher<(String, String)?, Never> {
  validatedUsername.combineLatest(validatedPassword) { username, password in
    guard let uname = username, let pwd = password else { return nil }
    return (uname, pwd)
  }
  .eraseToAnyPublisher()
}
{{< / highlight >}}

We can then hook this validation directly into our _Sign Up_ button and its disabled state.

{{< highlight swift  >}}
  Button("Sign up") { … }
  .disabled(signUpDisabled)
  .onReceive(model.validatedCredentials) {
    guard let credentials = $0 else {
      self.signUpDisabled = true
      return
    }
    let (validUsername, validPassword) = credentials
    guard validPassword != "invalid"  else {
      self.signUpDisabled = true
      return
    }
    self.signUpDisabled = false
  }
}
{{< / highlight >}}

For sample code featuring this and other techniques please checkout our [working examples](https://github.com/swift-you-and-i/working-examples/tree/master/Sources/WorkingExamples/combine-form-validation) repo.
