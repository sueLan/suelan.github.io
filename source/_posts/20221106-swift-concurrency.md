---
title: Exploring Swift Concurrency 
date: 2022-11-06 18:22:03
categories:
  - iOS
---

> Since Apple introduce Swift Concurrency, I've been always having an idea to write something about swift concurrency. But then I got a new job, and work has taken quite a lot of my time and energy; thus hardly i can find spare time to write something interesting.


In iOS development realm, we have several ways to write asynchronous code and parallel code.  People usually use `closures` and `completion handlers` to write asynchronous code. But too many closures and completion handler make code become more complicated. 

Here is an example from WWDC, it is to fetch thumbnail from cloud. 

```swift
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    guard let url = URL(string: "https://itunes.apple.com/search?term=taylor+swift&entity=album") else {
        completion(nil, NSError(domain: FetchUrlDomain, code: -1))
        return
    }
    
    let task = URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(nil, error)
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(nil, NSError(domain: FetchUrlDomain, code: -4))
        } else {
            guard let image = UIImage(data: data!) else {
                completion(nil, NSError(domain: FetchUrlDomain, code: -2))
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40), completionHandler: { thumbnail in
                guard let thumbnail = thumbnail else {
                    completion(nil, NSError(domain: FetchUrlDomain, code: -3))
                    return
                }
                
                completion(thumbnail, nil)
            })
        }
    }
    task.resume()
}
```
We can see some problem from the above codes snippet: 

1. Pyramid of doom
- deeply-nested closures make the code difficult to read and keep trace of.   
2. Error handling 
- callbacks make error handling difficult and verbose. There are 6 places in a single function to handle the error 
3.  Conditional execution is hard and error-prone
- After get data from cloud, it has to go to different condition branch to handle different situation
4. Many mistakes are easy to make
- simply returning without calling the correct completion-handler block. When forgotten, it is hard to debug 
You can check out more problems for using too many closures and completion handlers here [proposal-async-await.md](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md#motivation-completion-handlers-are-suboptimal)

As we've seen so many problems from closures and completion handlers. Apple introduce asynchronous function or so-called async-await to Swift. In fact, async/await pattern has been embraced in many languages

> F# added asynchronous workflows with await points in version 2.0 in 2007
> Microsoft released a version of C# with async/await for the first time in the Async CTP (2011). 
> Haskell lead developer Simon Marlow created the async package in 2012.[9]
> Python added support for async/await with version 3.5 in 2015
> TypeScript added support for async/await with version 1.7 in 2015.
> Javascript added support for async/await in 2017 as part of ECMAScript 2017 JavaScript edition.
> Rust added support for async/await with version 1.39.0 in 2019
> C++ added support for async/await with version 20 in 2020
> Swift added support for async/await with version 5.5 in 2021, adding 2 new keywords async and await.

After applying async/await to `fetchThumbnail` function, it becomes far more concise.  
```swift
func fetchThumbnailV2(for id: String) async throws -> UIImage {
    guard let url = URL(string: "https://itunes.apple.com/search?term=taylor+swift&entity=album") else {
        throw NSError(domain: FetchUrlDomain, code: -1)
    }
    
    let(data, response) = try await URLSession.shared.data(for: URLRequest(url: url))
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw NSError(domain: FetchUrlDomain, code: -4)
    }
    let image = UIImage(data: data)
    guard let thumbnail = await image?.byPreparingThumbnail(ofSize: CGSize(width: 40, height: 40)) else {
        throw NSError(domain: FetchUrlDomain, code: -3)
    }
    return thumbnail
}
```


## What is async/await function

### define async function: 
`async`: enable a function to suspend. 
 - The thread wont' be blocked and other tasks can run on it when current func is suspended.
`await`: marks where an async function may suspend execution;
 -  indicate `suspension point` in an async function where it has to give up its thread. 
 - the thread is free to execute other work 
 - suspend its caller too, so the caller should be async too 
 - Onced awaited call completes, execution resumes after the await. 

Function types can be marked explicitly as async, indicating that the function is asynchronous. 
```swift 
func asyncFuncA() async -> [String] {
    return xxx; 
}

func ayncFuncB(param: String) async -> [String] {
    let result =  await asyncFucA() // ayncFuncB suspended and give up control of the trhead. System come to schedule task executions on the thread  
    return result                   // ayncFuncB resumes and return result to its caller    
}
```

```

thread: -->Fun B              -->FuncB
        (await)               (resume)
system:         --------------
                  (schedule)

```


Besides, a closure can have async function type.
```swift
{ () async -> Int in
  print("here")
  return await getInt()
}
```

## How to use  async/await function

### call async function 

1. call it in asynchronous context, or you would get the following error if you don't want to do so
```
error: `async` function cannot be called from non-asynchronous context
```
asynchronous context could be: 
- asynchronous function body 
- asynchronous clossures 
- ashnchronous `Task` body

2. write `await` in front of the call to mark the possible suspension point
```swift 
    let result =  await asyncFucA() // ayncFuncB suspended and give up control of the trhead. System come to schedule task executions on the thread  
```

The possible suspension points in this piece of code marked with `await` indicate that the current piece of code might pause execution while waiting for the asynchronous function or method to return. 

> This is also called `yielding the thread` because, behind the scenes, Swift suspends the execution of your code on the current thread and runs some other code on that thread instead


### call async function in a sync context 

`async` enables a function to suspend - when a function suspends itself, it suspends its callers too. So its callers must be `async` as well. When you want to call an async function from a sync function, you can use an `async task` function. An async task packages up the work in the closure and sends it to the system for immediate execution on the next available thread, like the async function on a global dispatch queue.

```swift
async {
    await ayncFuncB()
}
```

### use asynchronous sequences
> [wwdc: Meet AsyncSequence](https://developer.apple.com/videos/play/wwdc2021/10058/)
basically, AsyncSequence is just a sequence, but async 

 A `for-await-in loop` potentially suspends execution at the beginning of each iteration, when it’s waiting for the next element to be available. 
 
```swift
extension URL {
  struct Lines: AsyncSequence { /* ... */ }
  func lines() async -> Lines
}

for try await line in myFile.lines() {
 // doing something
}
```

 Although different from synchronous `for-in-loop`, `for-await-in loop` potentially give up thread control at the beginning of each iteration.But when using asynchronous sequence, elements in the asynchronous sequence is executed one after another. Besides, an error might occurs at each iteration, and you can use ``for-try-await-in loop` 

 This time, Apple bring a bunch of new APIs leveraging asynchronous sequence 

```swift
// bytes: AsyncBytes in FileHanlde
for try await line in FileHandle.standardInput.bytes.lines {
    // ...
}
```

``` swift

let (bytes, response) = try await URLSession.shared.bytes(from: url)
```

```swift
let notification = await NotificationCenter.default.notifications(named: .NSSystemTimeZoneDidChange).first(where: { noti in })
```

you can use your own types in a for-await-in loop by adding conformance to the 
- [Sequence](https://developer.apple.com/documentation/swift/sequence)
- [AsyncSequence](https://developer.apple.com/documentation/swift/asyncsequence)

Check out an example from [proposals-asyncsequence](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md#proposed-solution) 


## Concurrency Structure
> Structured concurrency provides a paradigm for spawning concurrent child tasks in scoped task groups, establishing a well-defined hierarchy of tasks which allows for cancellation, error propagation, priority management, and other tricky details of concurrency management to be handled transparently.


### async let bindings

async-let 
-  left side of the `=`, it defines a local constant, a placeholder waiting to be initialed 
-  right side of the `=`, initializer expression is evaluated in a separate, concurrently-executing child task. After child task is completed, fullfill the value or propogate error

![][./let-binding.png]


When trying to call asynchronous functions in parallel, async-let will come to help. For example: 

```swift
let firstPhoto = await downloadPhoto(named: photoNames[0])
let secondPhoto = await downloadPhoto(named: photoNames[1])
let thirdPhoto = await downloadPhoto(named: photoNames[2])

let photos = [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```

 This piece of code has some drawbacks: although the download is asynchronous and lets other work happen while it progresses, only one call to downloadPhoto(named:) runs at a time. Each photo downloads completely before the next one starts downloading.


 To call an asynchronous function and let it run in parallel with code like this: 

```swift
async let firstPhoto = downloadPhoto(named: photoNames[0])
async let secondPhoto = downloadPhoto(named: photoNames[1])
async let thirdPhoto = downloadPhoto(named: photoNames[2])

let photos = await [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```

While using async let binding, 3 child tasks created and executed parellely while not blocking current thread and they run parallelly, and fullfill `firstPhoto` 
, `secondPhoto` and `thirdPhoto` later. After 3 constants all fullfiled values, it goes to show photos.

```
-> download firstPhoto --------------   |
---> download secondPhoto -----------   | 
-------> download downloadPhoto -----   | 
                                        |  await --> show photos
```

### Task 

A task is the basic unit of concurrency in the system. Every asynchronous function is executing in a task.

- A task can be in one of three states:

```
    suspended      <--->       running    --->  completed
```

- task operation
    - [cancellation](https://developer.apple.com/documentation/swift/task/cancel()), task will not stop immediately after cancel  
    

- `Child tasks`: Each task can create and have child tasks. Child tasks inherit some of the structure of their parent task, including its priority, but can run concurrently with it.

![](./task_parent.png)
In this picture, `fetchOne` task create two child tasks, data and metadata by using async-let binding, construct a well-structured task tree. 

#### Structured Task 

> [Explore structured concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134)

- Parent task can be completed only after all child tasks completed 
![](./task_parent_tree_complete.png)

- one child task failed, an error throw; Swift automatically marks another task as cancelled 
![](./task-tree.png)

- when a task is called, it won't be stoped immediately, just marked the result of it is unneeded. After data task is cancelled, its subtasks will be cancelled as well.  
![](./task_tree_cancel.png)
- use task group and get result from task group 
![](./task_group.png)


## Actor
> - [actor proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md) 
- [Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/)

### Basic

- provide a way to solve the data race in Swift concurrency programming 
- similar capabilities to structs, enums, and classes
- it is reference type 
- Unlike classes, actors allow only a single thread to access their mutable state at a time, which is called as `data isolation`, enforced statically by the Swift compiler through a set of limitations on the way in which actors and their instance members can be used,

```swift
actor Counter{
    var value = 0
    func increment() -> Int {
        value += 1
        return value
    }
    
    func resetSlowly(to newVlaue: Int) {
        value = 0
        for _ in 0..<newVlaue {
            increment()
        }
    }
}

let counter = Counter()

for _ in 0...100 {
    Task.detached {
        print(await counter.increment())
    }
}

```

Because the actor allows only one task at a time to access its mutable state, if code from another task is already interacting with the logger, this code suspends while it waits to access the property.

### User case 

This is a common pattern: a class with a private queue and some properties that should only be accessed on the queue. We replace this manual queue management with an actor class:

```swift
actor class PlayerRefreshController {
  var players: [String] = []
  var gameSession: GameSession

  func refreshPlayers() async { ... }
}
```

#### Things to note about this example:
- Declaring a class to be an actor is similar to giving a class `a private queue` and `synchronizing all access to its private state through that queue`.
- Because this synchronization is now understood by the compiler, you cannot forget to use the queue to protect state: the compiler will ensure that you are running on the queue in the class's methods, and it will prevent you from accessing the state outside those methods.
- Because the compiler is responsible for doing this, it can be smarter about optimizing away synchronization, like when a method starts by calling an async function on a different actor.

### Sendable types 
> [senable type roposal](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

- safe to share concurrently, each copy independent
    - Value types
    - Actor types
    - Immutable classes
    - internally-synchronized class 
    - @Sendable function types


## Behind the scenes

> [Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

The widely-used GCD has its darkside like excessive concurrency, like thread explosion, sheduling overhead due to too many threads and memory overhead caused by blocked threads holding stacks.

**Excessive context switch**: CPU runs less efficiently because CUP clock wasted in context switch instead of executing real task. 
![](./thread_context_switch.png)

- [Building Responsive and Efficient Apps with GCD](https://developer.apple.com/videos/play/wwdc2015/718/)
- [Modernizing Grand Central Dispatch Usage](https://developer.apple.com/videos/play/wwdc2017/706)
 
Swift concurrency leverage [Continuations](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md) to get task's continuation which suspends the task, and produce code that synchronous can then use a handle to resume to task. 
> [CheckedContinuation](https://developer.apple.com/documentation/swift/checkedcontinuation)
A mechanism to interface between synchronous and asynchronous code, logging correctness violations.

So, there is no thread switching cost but only function calls. And the concurrency runtime only creates as many threads as device's CPU cores and maintain cooperative thread pool to avoid thread explorsion and excessive context switches. 


 ### Swift Runtime 

 In one thread, when a function is called, a function frame will be allocated and put into the thread's stack. A function frame basically a chunk of memory to store local variables, return address. When that function finishes, function frame's pop from the stack.

![](./function_stack.png)

But in async function, things work a little bit different. 

![](./async_function.gif)

In this short video, there is an example 

```swift
func add(_ newArtibles: [Articles]) async throws {
    let ids = try await database.save(newArtibles, for:self)
    for(id, article) in zip(ids, newArtibles) {
        articles[id] = article
    }
}

func updateDatabase(...) async {
    await feed.add(articles)
}

// on Database 
func save() async throws -> [ID] { // ... }

```

When `updateDatabase` hits the thread suspend point, its function frame is put in the heap and then `add` function frame added to the stack, local variables like `id`, `article` will be stored inside it; also, there are two frames in heap, storing vriables like `newAriticles` which is available before and after suspend point `await database.save`. Such a variable has to be available cross the suspend point. 

![](./async_heap.png)

While system schedues and run other tasks on current thread, frame `save`, `add` `updateDatabase` will be store in the heap.  
 
![](./thread_schedule.png)  

When task is resumed, function frame will be in the stack. 
 ![](./reume_async.png)