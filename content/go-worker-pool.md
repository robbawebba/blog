---
title: "An Exploration of the Worker Pool Pattern in Go"
date: 2018-06-20T20:25:03-07:00
draft: true
author: "Rob Weber"
---

Go's built-in concurrency features are some of the most attractive and productive tools for software construction.
I believe they've played a pretty pivotal role in the growth that the language and community has been experiencing.

The language provides three simple tools for concurrent software: gorountines for execution, channels for communication, and `select` statements for coordination.
Given these tools, it is the software engineer's responsibility to create concurrent and practical applications that meet their needs.

One of the more popular designs for concurrent software I've seen is the **worker pool**.
In this post, I will discuss two different types of worker pools I've identified, the **finite** worker pool and the **recursive** worker pool, and possible error-handling techniques for each.

## But First, Some Background Info
Before we dig deeper into the worker pools, I want to define some of the concepts that are common for any worker pool implementation.

A **worker** is a single entity whose only responsibility is to receive and complete tasks.
A worker is most commonly implemented in Go as a single goroutine that listens for new tasks on a channel.

```go
// Task is an item that can be processed by a worker
type Task interface {
	Work()
}

// Worker receives tasks to be completed from newTasks
// and runs them until completion. The worker will only
// stop listening for tasks once newTasks is closed.
func Worker(newTasks chan Task){
	for t := range newTasks {
		t.Work()
	}
}
```
The above worker implementation receives the channel `newTasks` as a parameter and waits for new tasks by ranging over the channel.
Once the worker receives a new task `t`, the worker will run the task by calling `t.Work()`.
The worker will exit the `for` loop once `newTasks` is closed.

The `Task` interface defined above is the unit of work for the worker.
Thanks to the power of Go interfaces, the worker is pretty flexible with the types of tasks it can process.
As long as a type has a `Work` method, the worker can handle it.

Let's create a task that implements this interface:
```go
// SleepyTask implements the Task interface
type SleepyTask struct {
	NapLength int // the number of seconds this task should sleep for
}

// Work sleeps for t.NapTime seconds
func (t SleepyTask) Work() {
	fmt.Printf("sleeping for %d seconds\n", t.NapLength)
	time.Sleep(time.Duration(t.NapLength) * time.Second)
	fmt.Printf("Best %d-second nap ever!\n", t.NapLength)
}
```

The `SleepyTask` is a simple but complete example of something that can be processed by our worker pool since we've implemented the `Work` method.

Workers and tasks are the two necessary pieces to create a worker **pool**. A pool is a fixed number of workers who all listen from the same tasks queue.
The pool usually begins by creating the shared task channel.
The pool will then create the predetermined number of workers and pass them the shared task channel as an argument (as we've seen in the definition of the `Worker()` function.

```go
// maxWorkers is the fixed size of the worker pool
var maxWorkers = 4

// Modifying our Worker method to accept a sync.WaitGroup parameter
func Worker(newTasks chan Task, wg *sync.WaitGroup) {
	for t := range newTasks {
		t.Work()
	}
	wg.Done()
}

func main() {
	// create the channel on which we'll send new tasks to the workers
	tasks := make(chan Task)

	// We'll want to make sure we properly close down our workers
	var wg sync.WaitGroup

	// create worker pool
	for i := 0; i < maxWorkers; i++ {
		go Worker(tasks, &wg)
	}
	wg.Add(maxWorkers)

	// generate some SleepyTasks that sleep for 1-5 seconds
	for i := 1; i < 6; i++ {
		tasks <- SleepyTask{NapLength: i}
	}

	// signal to other workers that there are no more tasks
	close(tasks)

	// wait for all of the workers to return
	wg.Wait()
}
```

Although the worker pool is not a novel concept (also known as a thread pool), it's utility is very different for Go. Goroutines are famous for having low overhead costs related to memory usage and context switching (the details of this are worth a whole blog series, or at least a few links).
So why would we want to limit ourselves to only a handful of workers if the runtime infrastructure is just as efficient with hundreds of goroutines?
The worker pool pattern is stil a helpful tool when the tasks are heavy in file or network I/0 (TODO: Research a little more).

