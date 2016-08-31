---
layout: post
title: Threading issues in iOS
---

Recently during my development I faced a common threading related [producer-consumer](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem) problem. 

So I thought lets understand the problem and solve it. I have created the simplified producer-consumer scenario below.

```
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

In the above piece of code I have created a concurrent dispatch queue, that is, the blocks dispatched in this queue would be executed in concurrent fashion and we cannot predict, which block will finish first.Let us understand why there is a problem in above code. We can see that both `add()` and `remove()` operations are done in concurrent queues.So there is a possibility that more than one blocks might be appending an integer in the array. And similarly there might be the case that more than one blocks trying to remove the last element. So all this blocks occurring simulataneously may lead to spurious results and with high probability a possible crash. 

Lets understand what will cause a crash. Take a case that two blocks say B1 and B2 are trying to remove the last element simulataneously, but the `items` only has one element in it. Since `items` has one element in it, and both blocks are executed almost simulataneously, they may satisfy this condition `!items.isEmpty`. And suppose block B1 executes `items.removeLast()` before block B2 then there will be a crash when block B2 tries to execute the same command.

Lets solve this issue, We can solve this issue in three ways. Lets start exploring the solutions.

* **Locks**
	
The simple solution that comes in the mind would be to lock the other execution blocks when `items` is being modified.Lets implement it
	
```
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
In the above code we can see that, `lock` blocks the execution of other blocks, when `items` is being modified.The other blocks wait till the `lock` is unlocked.In this way at any time `items` would be modified by only one block, thus solving our problem.Wasn't this easy? Lets use GCD to solve this issue.

* **Serial Queue**

We can implement the same thing using GCD's serial queue. We can dispatch the task of addition and deletion in the serial queue.

```
let executionQueue = dispatch_queue_create("com.prit.TestGCD.executionqueue", DISPATCH_QUEUE_SERIAL)

//Producer
func add(x: Int) {
    dispatch_async(executionQueue) { [weak self] in
        guard let strongSelf = self else {
            return
        }
        strongSelf.items.append(x)
    }
}
    
//Consumer
func remove() {
    dispatch_async(executionQueue) { [weak self] in
        guard let strongSelf = self else {
            return
        }
        guard !strongSelf.items.isEmpty else {
            return
        }
        strongSelf.items.removeLast()
    }
}

```

So how does this solve the issue. By dispatching each task of addition and deleetion in a serial queue avoids the possibility of accessing `items` simulataneously.
We can solve this issue by using semaphore too. 

* **Using Semaphore**

Semaphore allows more than one thread to access a shared resource if its configured in that way. In our case we just need only one thread to execute the resource at any time. Lets understand the semaphore first.

```
let semaQueue = dispatch_queue_create("com.prit.TesttGCD.Sema", DISPATCH_QUEUE_CONCURRENT)
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

```
Sema block 1
Sema block 2
Sema block 3
Sema block 4
```	
Which is what is exepected, as we are delaying the execution of the first block by 5 secs.

Lets use this idea in our case.

```
    let semaQueue = dispatch_queue_create("com.prit.TesttGCD.SemaQueue", DISPATCH_QUEUE_CONCURRENT)
    let semaphore = dispatch_semaphore_create(1)
   
    func add(x: Int) {
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
        print("Add Execution \(x)")
        items.append(x)
        dispatch_semaphore_signal(semaphore)
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

If we run the above code, we wouldn't face any problems of producer-consumer problem and all threads do their work as a disciplined child :P

I hope, the blog was useful, if any doubts feel free to contact me at <mailto:prit.nandgaonkar@gmail.com>