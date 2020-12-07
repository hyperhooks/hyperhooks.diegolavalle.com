+++
title = "Computed properties as synthesized bindings"
subtitle = "Gain control over your bindings' behavior"
date = 2020-08-06T12:00:00Z
twitter = "diegolavalledev"
tags = ["ios", "swift", "swifui", "combine", "ui"]
featured = false
example = "/examples/fave-dishes"

[[resources]]
  name = "cover"
  src = "cover.png"
+++

Computed properties are useful in so many ways and within the context of _SwiftUI_ they are even more powerful. We can use them to split up our view body while keeping access to the same data. Or we can derive new values from our state that will recalculate with every change. We can also use computed properties in model objects, observing and even setting their values via _synthesized bindings_.

<!--more-->

# Italian dishes

Suppose we have a list of classic Italian dishes that we want to display. Each dish can be added or removed from our favorites but only one of them at a time can be pinned to the top of the list.

{{< figure src="example.gif" title="Fave dishes can also be pinned" >}}

We would start with a simple model:

{{< highlight swift  >}}
struct Dish: Identifiable {
  let id = UUID()
  let name: String
  var isFavorite = false
  
  static let allDishes = [
    Dish(name: "🍝 Spaghetti Carbonara"),
    Dish(name: "🧀 Eggplant Parmigiana"),
    Dish(name: "🍮 Classic Panna Cotta"),
    Dish(name: "🥗 Caprese Salad")
  ]
}
{{< / highlight >}}

We have our named dishes which we can favorite. Now let's store them in an `ObservableObject` together with a new `pinnedId` variable to help us keep track of which dish is currently featured at the top of the list. 

{{< highlight swift  >}}
class UserData: ObservableObject {
  
  @Published var dishes: [Dish] = Dish.allDishes  
  @Published var pinnedId: UUID = Dish.allDishes[0].id
  
  static let shared = UserData()
}
{{< / highlight >}}

We selected the first dish as the default one to be pinned and we added a `shared` singleton instance of our user data.

Now let's create a view to display all of these delicacies. We'll start with a single row showing the dish's name, favorite status and pinned status:

{{< highlight swift  >}}
struct DishRow: View {

  let dish: Dish
  let pinnedId: UUID
  
  var body: some View {
    HStack {
      if dish.id == pinnedId {
        Text("📌")
      }
      
      Text(dish.name)
      
      Text("\(dish.isFavorite ? "💗" : "🤍")")
    }
    .padding()
  }
}
{{< / highlight >}}

We can now show our list of all dishes:

{{< highlight swift  >}}
struct DishList: View {

  let dishes: [Dish]
  let pinnedId: UUID
  
  var body: some View {
    VStack {
      ForEach(dishes) { dish in
        DishRow(dish: dish, pinnedId: pinnedId)
      }
    }
  }
}
{{< / highlight >}}


# Pinning the top dish

Let's hide the pinned dish from the list as we will put it back later at the top of the view.

{{< highlight swift  >}}
…
ForEach(dishes) { dish in
  if dish.id != pinnedId {
    DishRow(dish: dish, pinnedId: pinnedId)
  }
}
…
{{< / highlight >}}

Now let's compose a vertical stack with our list and the pinned dish as a single row at the top. We can feed data to both list and row straight from our user data instance which we'll also observe for changes.  

{{< highlight swift  >}}
struct FaveDishes: View {

  @ObservedObject var data = UserData.shared

  var body: some View {
    VStack {
      DishRow(dish: data.pinnedDish, pinnedId: data.pinnedId)
      Divider()
      DishList(dishes: data.dishes, pinnedId: data.pinnedId)
      Spacer()
    }
  }
}
{{< / highlight >}}

This is all great but you may have noticed we didn't actually add any capabilities to change the status of our dishes. Every view so far has been display-only.

{{< figure src="display-only-dishes.png" title="Static list of dishes" >}}

# Enabling views to modify the dish's status

Let's then prepare our views for modifying the dishes' status by swapping a few `let`'s for some `@Binding var`'s starting with the row view:

{{< highlight swift  >}}
struct DishRow: View {

  @Binding var dish: Dish
  @Binding var pinnedId: UUID
  …
{{< / highlight >}}

This will allow each row to modify itself, changing the favorite status or selecting the dish for pinning. But first we need to make sure we generate and pass those bindings to the row initializer.

For this we work our way up to the dish list. We'll have to iterate over the _indices_ of our `dishes` array to be able to reference both the dish and its corresponding binding.

{{< highlight swift  >}}
struct DishList: View {

  @Binding var dishes: [Dish]
  @Binding var pinnedId: UUID?
  
  var body: some View {
    VStack {
      ForEach(dishes.indices) { i in
        if dishes[i].id != pinnedId {
          DishRow(dish: $dishes[i], pinnedId: $pinnedId)
        }
        …
{{< / highlight >}}

As you can see we used the index to get our `Binding<Dish>` from the synthesized `$dishes` variable which is itself an array of dish bindings.

Now let's try to apply the same treatment to our root view:

{{< highlight swift  >}}
struct FaveDishes: View {

  @ObservedObject var data = UserData.shared

  var body: some View {
    VStack {
      DishRow(
        dish: $data.dishes[
          data.dishes.firstIndex { $0.id == data.dishes.pinnedDish }!
        ],
        pinnedId: $data.pinnedId
      )
      Divider()
      DishList(dishes: $data.dishes, pinnedId: $data.pinnedId)
      Spacer()
    }
  }
}
{{< / highlight >}}


Noticed anything odd? We will come back to that snippet later on the following section.

And finally we can modify the status of each dish represented in a row. While we are at it, let's split the body in three using _computed properties_:

{{< highlight swift  >}}
struct DishRow: View {

  @Binding var dish: Dish
  @Binding var pinnedId: UUID
  
  var body: some View {
    HStack {
      pinIndicator
      Text(dish.name)
      favoriteIndicator
    }
  }
  
  var pinIndicator: some View {
    VStack {
      if dish.id == pinnedId {
        Text("📌")
      } else {
        Button("Pin") {
          pinnedId = dish.id
        }
      }
    }
  }

  var favoriteIndicator: some View {
    Button("\(dish.isFavorite ? "💗" : "🤍")") {
      self.dish.isFavorite.toggle()
    }
  }
}
{{< / highlight >}}


Much cleaner and we now can express our love for these delicious items.

# Bound computed properties

You may have noticed though at the end of the previous section a particular expression, so intricately beautiful that it had to be split among three lines of code. Here it is in all its splendor:

{{< highlight swift  >}}
$data.dishes[
  data.dishes.firstIndex { $0.id == data.dishes.pinnedDish }!
]
{{< / highlight >}}

Once again we are referencing an array of dish bindings except this time we don't have the index to the pinned dish so we need to look that up.

One possible improvement would be to provide the pinned dish index as a computed property on our data store. We could also manually create an explicit binding which would obviate the use of `@ObservedObject`. But what if we could compute the whole pinned dish variable – complete with getter and setter – for observing views to bind to? 

{{< highlight swift  >}}
class UserData: ObservableObject {
  …
  var pinnedDish: Dish? {
    get {
      return dishes.first { $0.id == pinnedId }!
    }
    set {
      if let i = dishes.firstIndex(where: { $0.id == pinnedId }) {
        dishes[i] = newValue
      }
    }
  }
}
{{< / highlight >}}

So the great thing about computed properties on observed objects is that they get their own synthesized binding! This way our ugly duckling expression can turn into a beautiful one-line swan:

{{< highlight swift  >}}
…
DishRow(dish: $data.pinnedDish, pinnedId: $data.pinnedId)
…
{{< / highlight >}}


And it's all transparent to the views.

# Optional bindings

Our widget now totally works as expected but it would be nice if we had the option _not_ to feature any specific dish at all since we love them all the same.

First thing we'd need to do is turn `pinnedId` into an _optional_:

{{< highlight swift  >}}
class UserData: ObservableObject {

  @Published var pinnedId: UUID?
  …
{{< / highlight >}}

After this modification most of our views will only need to adapt slightly and in fact so will our `pinnedDish` computed property with a few minor changes. But what will happen to its synthesized binding?

Well since `$data.pinnedDish` now returns a `Binding<Dish?>` we can't simply unwrap it to pass it to `DishRow`. Fortunately the framework provide us with a tool for this one specific situation: a binding initializer that takes a _binding of an optional_ and returns a _binding optional_ `Binding<Dish>?`. 

{{< highlight swift  >}}
struct FaveDishes: View {

  @ObservedObject var data = UserData.shared

  var body: some View {
    VStack {
      if data.pinnedDish != nil {
        DishRow(
          dish: Binding<Dish>($data.pinnedDish)!,
          pinnedId: $data.pinnedId
        )
        Divider()
      }
      DishList(dishes: $data.dishes, pinnedId: $data.pinnedId)
    }
  }
}
{{< / highlight >}}

Now we can feature a dish only if we want. Otherwise we'll just show the regular full menu.

That's all, please check out the associated _Working Example_ to see this technique in action and to view its full source code.
