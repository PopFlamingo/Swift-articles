#  New `async` keyword usage
In the *Async/Await for Swift* proposal, `async` is only used in order to declare an asynchronous function.
I suggest extending its usage to call site, I see two main usages.


## Usage of `async` inside of an asynchronous function 

### For returning (throwing/non-throwing) functions
The usage of `async` could be particularly interesting in the case of returning `async` (throwing/non-throwing) functions, here I propose to directly *synthesise* async preceded function calls as futures, which would enable this kind of design:
```swift
func someAsyncFunc() async {
	var userData = async downloadUserData()
	var image = async downloadImage()
	return await User(userData, image)
}
```

This would have the advantage of reducing boilerplate code, as an equivalent direct usage of Future would have looked like this:
```swift
func someAsyncFunc() async {
	var userData = Future { await downloadUserData() }
	var image = Future { await downloadImage() }
	return await User(userData.get(), image.get())
}
```

This also integrates well with throwing code: if `downloadUserData()` and `downloadImage()` were throwing, we could simply do:
```swift
func someAsyncFunc() async throws {
	var userData = try async downloadUserData()
	var image = try async downloadImage()
	return await User(userData, image)
}
```

Whereas with direct usage of future this would probably look like this:
```swift
func someAsyncFunc() async throws {
	var userData = Future { try await downloadUserData() }
	var image = Future { try await downloadImage() }
	return try await User(userData.get(), image.get())
}
```





### Usage of `async` for non-returning, throwing functions
For throwing functions that doesn’t return anything, we could do, when inside of an `async` throwing function:
```swift
func foo() async throws {
	try async someOtherThrowingFunction()
}
```

## Simplifying call of `async` functions from a synchronous context
Just as `await` is used to explicitly indicate that some piece of code depends on the completion of an asynchronous function, `async` at call site could be used to explicitly indicate that we do wait.

If we have this `async` function:
```swift
func downloadAndUpdateImageView() async {
	let image = await downloadImage()
	self.imageView.image = image
}
```

Then, inside of a non-`async` function, we could do:
```swift
@IBAction func buttonClicked() {
	async downloadAndUpdateImageView()
}
```

Instead of:
```swift
@IBAction func buttonClicked() {
	beginAsync {
		await downloadAndUpdateImageView()
	}
}
```

Additionally `async` could also be used in order to begin a block when there is no need to declare an external `async` function (here this would have the same usage as `beginAsync`) but with the advantage of feeling like a first-class language construct:
```swift
@IBAction func buttonClicked() {
	async {
		let image = await downloadImage()
		self.imageView.image = image
	}
}
```


### Other cases that need to be considered
How would **throwing** async functions be called from synchronous functions ?




