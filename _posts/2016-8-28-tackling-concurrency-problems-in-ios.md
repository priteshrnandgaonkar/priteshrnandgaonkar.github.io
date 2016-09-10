---
layout: post
title: Tackling Concurrency Problems in iOS
---

The common concurrency side-effects faced are the one related to acessing and modifying the shared resource from different threads. One can think of this as a [producer-consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem).

Consider the following simplest concurrency scenario.

```swift
var items = [Int]()

//Producer
func add(x: Int) {
    items.append(x)
}
    
//Consumer
func remove() {
    guard !items.isEmpty else {
        return
    }
    items.removeLast()
}

let dispatchQueue dispatch_queue_create("com.prit.TestGCD.queue", DISPATCH_QUEUE_CONCURRENT)

for x in 1..<10 {
    dispatch_async(dispatchQueue, {[weak self] in
        self?.add(x)
        })
}
    
for _ in 1..<10 {
    dispatch_async(dispatchQueue, { [weak self] in
        self?.remove()
        })
}
```
The above code appends and removes integer 10 times in an array. Since all these calls are done in a concurrent way, one cannot predict the order in which they are executed.Thus the above code models the concurrent scenario appropriately.

Problems(side-effects) in the above code - Since addition and deletion occurs in concurrent way there is the possibility of crash due to race condition, which we will inspect below.  

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
 var lock = NSLock()
 func add(x: Int) {
    lock.lock() // Locks the thread
    items.append(x)
    lock.unlock() // Unlocks the thread
}
    
//Consumer
func remove() {
    lock.lock()
    defer {
    	lock.unlock()
    }
    guard !items.isEmpty else {
        return
    }
    items.removeLast()
}

```
The above code blocks the execution of other threads whenever the collection `items` is modified. Thus whenever a block is being executed, the lock ensures that no other block is being taken up for execution and hence making the above code thread-safe.

Let's solve this with GCD API's.

## Serial Queue ##

The above solution gave us the clue that the problem is solved if the queues are being executed serially, so lets use GCD's `DISPATCH_QUEUE_SERIAL`

```swift
let concurrentQueue = dispatch_queue_create("com.prit.TestGCD.ConcurrentQueue", DISPATCH_QUEUE_CONCURRENT)

//Producer
func add(x: Int) {
    dispatch_async(concurrentQueue) {
        self.items.append(x)
    }
}
    
//Consumer
func remove() {
    dispatch_async(concurrentQueue) {
        guard !(self.items.isEmpty) else {
            return
        }
        self.items.removeLast()
    }
}

``` 

## Semaphore ##

Semaphore allows more than one thread to access a shared resource if it is configured accordingly. In our case we just need only one thread to execute the resource at any time. Lets understand the semaphore first.

```swift
let semaQueue = dispatch_queue_create("com.prit.TestGCD.Sema", DISPATCH_QUEUE_CONCURRENT)
let semaphore = dispatch_semaphore_create(1)
    
dispatch_async(semaQueue) {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
    NSThread.sleepForTimeInterval(5)
    print("Sema block 1")
    dispatch_semaphore_signal(semaphore)
}
    
dispatch_async(semaQueue) {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
    print("Sema block 2")
    dispatch_semaphore_signal(semaphore)
}
    
dispatch_async(semaQueue) {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
    print("Sema block 3")
    dispatch_semaphore_signal(semaphore)
}
    
dispatch_async(semaQueue) {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
    print("Sema block 4")
    dispatch_semaphore_signal(semaphore)
}
```

If you run the above code, you will see that after 5 secs the following will be printed

```swift
Sema block 1
Sema block 2
Sema block 3
Sema block 4
```	
Which is exepected, as we have configured the semaphore to execute just one block at a time and the first block has a delay of 5 secs. 

Lets use this idea in our case.

```swift
    let semaQueue = dispatch_queue_create("com.prit.TestGCD.SemaQueue", DISPATCH_QUEUE_CONCURRENT)
    let semaphore = dispatch_semaphore_create(1)
   
    func add(x: Int) {
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
        print("Add Execution \(x)")
        items.append(x)
        di
        spatch_semaphore_signal(semaphore)
    }
    
    //Consumer
    func remove() {
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
        defer {
            dispatch_semaphore_signal(semaphore)
        }
        guard !items.isEmpty else {
            print("Remove Execution returned because of 0 items")
            return
        }
        print("Remove Execution \(items.last)")
        items.removeLast()
    }  
```

Still you might be thinking that, there is lot of steps involved to solve this problem. But hold on, GCD has a simplest solution which precisely solves the above problem.

## Dispatch Barrier ##

GCD’s barrier API ensures that the submitted block is the only item executed on the specified queue for that particular time. This means that all items submitted to the queue prior to the dispatch barrier must complete before the block will execute.

```swift
let concurrentQueue = dispatch_queue_create("com.prit.TestGCD.ConcurrentQueue", DISPATCH_QUEUE_CONCURRENT)

 //Producer
    func add(x: Int) {
        dispatch_barrier_async(concurrentQueue) { 
            print("Add Execution \(x)")
            self.items.append(x)
        }
    }
    
    //Consumer
    func remove() {
        dispatch_barrier_async(concurrentQueue) {
            guard !(self.items.isEmpty) else {
                print("Remove Execution returned because of 0 items")
                return
            }
            print("Remove Execution \(self.items.last)")
            self.items.removeLast()
        }
    }

```
