*This is a WIP, and may contain errors.*
*Note that this article is about what `async` and `await` **may** look like in Swift 5, the way this will actually work and when it will be available is subject to change.*

# Swift `async` and `await`
The first part of the [*Swift concurrency manifesto*](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782) is about the `async` and `await` keywords, they will be the first to be implemented, hopefully as early as Swift 5!
If you haven’t read the [concrete async/await proposal](https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619), I encourage you to do that before reading this article, the proposal is complete and gives concrete examples of how async/await will improve the experience of working with async APIs. 

The goal of this article is to make it easier to understand how `async`/`await` work if you have no or little experience with similar concepts in other languages. I am trying to cover points that may be misunderstood based on the difficulties I had myself when trying to understand the proposal.

**What do the `async` and `await` keywords do exactly?**
Let's start with some code, here is how you will be declaring an async function:
```swift
func myAsyncFunction() async -> SomeType { ... }
```
*Notice that this is done the same way as if you was declaring a throwing function, except you are using `async` instead of `throws`*

At first, the presence of the `async` keyword made me think that it was automatically modifying how the function was executed, probably because of the ressemblance to this:
```swift
someDispatchQueue.async { … }
```

In reality, the only thing the `async` keyword does is that it transforms your function into a *coroutine*. A coroutine can *return* as any usual function does, but it can also be *suspended* (and only *return* later) while letting the current thread working on other tasks that don’t depend on the coroutine.

So what suspends the execution of a coroutine?
In everyday code, the most common reason will be that your coroutine calls another coroutine that suspended itself, so it will be suspended too. For instance, if you have a `downloadData` function that asynchronously downloads the data at the specified URL:
```swift
func downloadData(from: URL) async -> Data? { … }
```

Then if you write a function like this:
```swift
func downloadProfileImage(for user: User) async -> Image {
	let rawData = await downloadData(from: user.profilePictureURL)!
	let userImage = Image(with: rawData, format: .jpeg)
	return userImage
}
```

The `downloadProfileImage` function will be suspended until `downloadData` finishes downloading, then its execution will be resumed and it will finish its execution normally.

There are several things to note about this:
1. As you can see, we called `downloadData` by prefixing it with the `await` keyword, Swift requires you to prefix calls to `async` functions with `await`, that’s in order to make it clear that the call can suspend the execution of your coroutine.
2. Because the call to an `async` function can suspend the calling context, it has to be done inside of a coroutine ; in other words, you can only call `await` within an `async` function.
3. You may have noticed that a paradox arises from point (1) and (2) : if an `async` function has to be called with `await` (1), and if `await` can only be called within an `async` function (2), then it seems like we are trapped! How do we call an `async` function from a non-async function? Don’t worry about this too much for now, we will come back to it later.
4. **Last but not least:** It is important to understand that `await` can *suspend* the current coroutine which is different from *blocking* the current thread. When a coroutine is suspended, the control is immediately given back to the current thread so that it can execute the rest of your code that does not depend on your async call and its only later that your coroutine will resume its execution.

Again, I want to insist on the fact that the **only** thing that `async` does is transforming your function into a coroutine, a function able to suspend itself.

While simply using async functions provided by various libraries will be really easy (seeing `async`/`await` as sugar for completion handlers should be enough if you are just using async APIs provided by frameworks), it is important to understand how things work under the hood if you are writing async functions for libraries.

Because `async` doesn’t do much on its own, you have to manually specify how the code of your coroutine is dispatched, for instance, let’s say you want to create a function making a really lengthy computation on a potentially huge array :
```swift
func lenghtyComputation(with elements: [SomeType]) async -> [SomeType] { … }
```

If you implemented it this way:
```swift
func lenghtyComputation(with elements: [SomeType]) async -> [SomeType] {
	var modifiedElements = [SomeType]()
	for element in elements {
		modifiedElements.append(tranformed(element))
	}
}
```

There would be no benefit from having marked your function as `async` because since your function does not suspend itself, it is treated as any usual function, so here:
```swift
let transformedArray = await lenghtyComputation(with: aBigArray)
```

The thread would be blocked until `lengthyComputation` returns!

How do we avoid blocking the thread?
The most « bare bones » way of doing that is to use:
```swift
func suspendAsync<T>(
  _ body: (_ continuation: @escaping (T) -> ()) -> ()
) async -> T
```

When you call `suspendAsync`, it calls the `body` closure before suspending the coroutine, its inside of body that you specify how your asynchronous code will be dispatched (using the concurrency framework of your choice) and once the long running task you asynchronously dispatched is finished, you must resume the execution of the coroutine by calling the continuation closure (if needed, you can pass a value to `continuation`, which is similar to returning a value).
An example may be easier to understand ! Using *Grand Central Dispatch* as your concurrency framework, the previous (intentionally wrong) example would be rewritten as:
```swift
func lenghtyComputation(with elements: [SomeType]) async -> [SomeType] {
	let result = await suspendAsync { continuation in // (1)
		DispatchQueue(label: "someQueue", qos: .userInitiated).async { // (2)
			var modifiedElements = [SomeType]()
			for element in elements {
				modifiedElements.append(tranformed(element))
			}
			continuation(modifiedElements) // (5)
		} // (3)
	} // (4)

	return result // (6)
	
}
```

Let’s explore what happens :
1. suspendAsync calls its body closure (1-4) 
2. We asynchronously dispatch the closure (2-3) on a *GCD* `DispatchQueue` with `.userInitiated` quality of service.
3. We nearly instantly reach (3) as we dispatched the closure asynchronously
4. We reach the end of the `body` closure, at this point, the current coroutine (`lenghtyComputation`) gets suspended, giving back control to the current thread.
5. When the loop finishes its execution, it calls its continuation : `continuation(modifiedElements)`, this resumes the `lengthyComputation` closure, assigning `modifiedElements` to the result variable (at (1))
6. `lenghtyComputation` returns the `result` variable

There are quite a lot of elements to think about at the same time here :
- We are using `suspendAsync` to suspend a coroutine, enabling it to yield control to its calling thread.
- Before suspending the coroutine, `suspendAsync` calls its `body` closure, letting us dispatch our work using whatever threading mechanism we want (here I used *GCD* but it could be anything else).
- Inside of the `body` closure, we are given access to the *continuation* of the coroutine we are suspending. In this context, a *continuation* represents « the rest of the code » whose execution « depends » on the call to your coroutine. We will define what « the rest of the code » and « depends » exactly mean later. If you pass a value to the continuation (as we did here) `suspendAsync` will return this value when the execution of the coroutine resumes.

You learned how `suspendAsync` and `continuation` let us *suspend* and *resume* coroutines, now lets see how to *start* a coroutine — an async function — from a synchronous context — a *non-async* function.

Aw we saw earlier, you need to call your `async` functions with `await` but await can only be called within `async`, the issue is that at some point, you’ll need to start your asynchronous function from a synchronous context. Again, as for `suspendAsync`, chances are that unless you are writing lower level library code, you won’t ever need to directly call it because the frameworks your app is based on will probably provide you with « async hooks », for instance, `UIKit` could provide you with `@IBAction`  that supports `async` functions, like this:
```swift
@IBAction func buttonClicked(sender: UIButton) async {
	let image = await downloadImage()
	imageView.image = image
}
```

This would « just work », as the framework would take the right steps under the hood to call your async function.

If you need to implement it yourself at a lower level, you would go from synchronous code to asynchronous code using `beginAsync` :
```swift
func beginAsync(_ body: () async throws -> Void) rethrows -> Void
```

As you can see, the `body` closure that `beginAsync` takes is `async`, meaning that you can call `await` inside of this closure. If UIKit didn’t provided `async` IBActions and you needed to call an async function from your event handler, you would do:
```swift
@IBAction func buttonClicked(sender: UIButton) {
	beginAsync {
		let image = await downloadImage()
		imageView.image = image
	}
}
```

`beginAsync` returns either when the `async` functions you called from it returns **or** when they suspend. This actually explains why our earlier example where we declared an `async` function without specifying how its lengthy operations were performed didn’t had any advantage and actually blocked the thread as any usual function would : as long as your function doesn’t suspend itself, beginAsync (and thus the thread) will be « waiting » for it to return.

Earlier, I also mentioned that the `continuation` you get in `suspendAsync` represented « the rest of your code » that « depends » on your await, this wasn’t a really good definition so let’s define it more accurately.

The `continuation` closure represents (*reifies*) the code **after** the call to `suspendAsync` up to the last line of the closest **beginAsync** closure in the call stack.
Here is an example:
```swift
func foo() async {
    print("Before suspendAsync in foo")
    suspendAsync { continuation in
        print("Hello world!")
        continuation()
    }
    print("After suspend async")
}

func bar() async {
    print("Before call to baz() inside of bar()")
    await baz()
    print("After call to baz() inside of bar()")
}

func baz() {
    print("Before beginAsync in baz()")
    beginAsync {
        print("Inside of beginAsync in baz()")
        await bar()
        print("After call to bar()")
    }
		print("After beginAsync")
}
```

In this example, the `continuation` closure inside of `bar` will represent those instructions :
```swift
print("After suspend async")
print("After call to baz() inside of bar()")
print("After call to bar()")
```

Notice that `print("After beginAsync")` isn’t contained inside of this continuation because it is outside of the closure we gave to `beginAsync`.

TBC





