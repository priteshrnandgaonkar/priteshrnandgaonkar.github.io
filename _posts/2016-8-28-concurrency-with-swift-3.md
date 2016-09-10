---
layout: post
title: Concurrency with swift 3
---

The common concurrency issues faced are the one related to acessing and modifying the shared resource from different threads. One can think of this as a [producer-consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem).

Consider the following simplest concurrency scenario

```swift
var items = [Int]()

//Producer:
func add(num: Int) {
    print("Add \(num)")
    items.append(num)
}
    
//Consumer:
func remove() {
    guard !items.isEmpty else {
        return
    }
   let num = items.removeLast()
   print("Remove \(num)")
}

let dispatchQueue = DispatchQueue(label: "com.prit.TestGCD.DispatchQueue", attributes: DispatchQueueAttributes.concurrent)

for x in 0..<10 {
    dispatchQueue.async {
        self.add(num: x)
    }
}
    
for _ in 0..<10 {
    dispatchQueue.async {
        self.remove()
    }
}
```
The above code appends and removes integer 10 times in an array. Since all these calls are done in a concurrent way, one cannot predict the order in which they are executed.Thus the above code models the concurrent scenario appropriately.

Problems in the above code - Since addition and deletion occurs in concurrent way, there is the possibility of a crash, which we will inspect below.  

### Problem Inspection ###

Consider the following scenario :-

1. Nine addition blocks are already executed and just one addition block is left for execution.
2. Whereas all 8 `remove` blocks are executed and currently the last two are being executed.Thus at this point the count of `items` will be 1.
3. Due to race consition assume that the last addition block is being executed after the two `remove` blocks.

In the above scenario, since the `items` count is 1 and two remove blocks are being executed simulataneously, thus calling `removeLast` two times on an array with count 1 will lead to a crash.

The above problem can be solved in many ways, which are listed below.

## Locks ##
	
The simplest solution would be to lock the other execution blocks when `items` is being modified. 
	
```swift
 var lock = Lock()
 
//Producer:
func add(num: Int) {
    lock.lock()
    print("Add \(num)")
    items.append(num)
    lock.unlock()
}
    
//Consumer:
func remove() {
    lock.lock()
    defer {
        lock.unlock()
    }
    guard !items.isEmpty else {
        return
    }
    let num = items.removeLast()
    print("Remove \(num)")
}
```
The above code blocks the execution of other threads whenever the `items` is modified. Thus whenever a block is being executed, the lock ensures that no other block is being taken up for execution and hence making the above code thread-safe.

The above code uses `defer` in `remove`, which was not used earlier.It is used because, the code in `defer` is called just before leaving the function. Since the `remove` has two paths to exit, replicating `lock.unlock()` at two places is not recommended. Hence in this case `defer` is useful.

The other approach is to use GCD API's.

## Serial Queue ##

The crux of the above solution is that it made the execution of `add` and `remove` serial. So this can be done through GCD too, which is as follows.

```swift
let serialQueue = DispatchQueue(label: "com.prit.TestGCD.SerialQueue", attributes: DispatchQueueAttributes.serial)

//Producer:
func add(num: Int) {
    serialQueue.async {
        print("Add \(num)")
        self.items.append(num)
    }
}
    
//Consumer:
func remove() {
    serialQueue.async {
        guard !self.items.isEmpty else {
            return
        }
        let num = self.items.removeLast()
        print("Remove \(num)")
    }
}

``` 

## Semaphore ##

Semaphore allows more than one thread to access a shared resource if it is configured accordingly. The following code snippet will make the working of semaphore clear.

```swift 
let dispatchQueue = DispatchQueue(label: "com.prit.TestGCD.DispatchQueue", attributes: DispatchQueueAttributes.concurrent)

let semaphore = DispatchSemaphore(value: 2)
    
dispatchQueue.async {
    semaphore.wait(timeout: DispatchTime.distantFuture)
    Thread.sleep(forTimeInterval: 5)
    print("Sema block 1")
    semaphore.signal()
}
    
dispatchQueue.async {
    semaphore.wait(timeout: DispatchTime.distantFuture)
    Thread.sleep(forTimeInterval: 2)
    print("Sema block 2")
    semaphore.signal()
}
    
dispatchQueue.async {
    semaphore.wait(timeout: DispatchTime.distantFuture)
    print("Sema block 3")
    semaphore.signal()
}
    
dispatchQueue.async {
    semaphore.wait(timeout: DispatchTime.distantFuture)
    print("Sema block 4")
    semaphore.signal()
}
 

```

After running the above code, we will observe that after 5 secs the following will be printed

```swift
Sema block 2
Sema block 3
Sema block 4
Sema block 1
```	
Which is expected, as we have configured the semaphore to execute two blocks at a time. Initially the first two blocks would be taken up for execution, the second block would finish first(i.e why `sema block 2` will be printed) and thus would allow the program to take one more block for execution, which will be block 3.Since block 3 has no delay, it would finish in no time just after block 2 and consequently the program would pick up block 4, which would end soon like block 3. Block 1 would end last as it has maximum delay amongst other blocks.

Thus to solve the above concurrency issue, we can use semaphore with capacity of one.

```swift
let semaphore = DispatchSemaphore(value: 1)
    
//Producer:
func add(num: Int) {
    semaphore.wait(timeout: DispatchTime.distantFuture)
    print("Add \(num)")
    items.append(num)
    semaphore.signal()
}
    
//Consumer:
func remove() {
    semaphore.wait(timeout: DispatchTime.distantFuture)
    defer {
        semaphore.signal()
    }
    guard !items.isEmpty else {
        return
    }
    let num = items.removeLast()
    print("Remove \(num)")
}
```



## Dispatch Barrier ##

GCD’s barrier API ensures that the submitted block is the only item executed on the specified queue for that particular time. This means that all items submitted to the queue prior to the dispatch barrier must complete before the block will execute.

```swift
let dispatchQueue = DispatchQueue(label: "com.prit.TestGCD.DispatchQueue", attributes: DispatchQueueAttributes.concurrent)

//Producer:
func add(num: Int) {
    dispatchQueue.async(flags: .barrier) {
        print("Add \(num)")
        self.items.append(num)
    }
}
    
//Consumer:
func remove() {
    dispatchQueue.async(flags: .barrier) {
        
        guard !self.items.isEmpty else {
            return
        }
        let num = self.items.removeLast()
        print("Remove \(num)")
    }
}
```
