---
title: "An Exploration of the Worker Pool Pattern in Go"
date: 2018-06-20T20:25:03-07:00
draft: true
author: "Rob Weber"
---

Go's built-in concurrency features are some of the most attractive and productive tools for software construction. I believe they've played a pretty pivotal role in the recent widespread adoption the language and community is experiencing.

While the language provides gorountines, channels, and select statements, it's the software engineer's responsibility to use these tools to design concurrent and performant software that meet their needs. One of the most popular patterns I've seen is the worker pool pattern. 

## The Worker Pool Pattern
A **worker** is a single entity whose only responsibility is to receive and complete tasks.
A worker is most commonly implemented in Go as a single goroutine that listens for new tasks on a channel.

{{< highlight go >}}
// Worker receives tasks to be completed from newTasks
// and runs them until completion. The worker will only
// stop listening fo tasks by closing newTasks.
func Worker(newTasks chan Task) {
	for t := range newTasks {
		t.Work()
	}
}
{{< / highlight >}}

A **pool** is a fixed number of workers who all listen from the same tasks queue.
The pool usually begins by creating the predetermined number of workers.
The pool also creates the shared task queue and shares it with the workers.

{{< highlight go >}}
// maxWorkers is the fixed size of the worker pool
var maxWorkers = 4
 
func main() {
	// create the channel on which we'll send new tasks to the workers
	tasks := make(chan Task)
	// We'll want to make sure we properly close down our workers
	var wg sync.WaitGroup
	// create worker pool
	for i := 0; i < maxWorkers; i++ {
		go Worker(tasks)
	}
	wg.Add(maxWorkers)
	// signal to other workers that there are no more tasks
	close(tasks)
	// wait for all of the workers to return
	wg.Wait()
}
{{< / highlight >}}

Although the worker pool is not a novel concept (also known as a thread pool), it's utility is very different for Go. Goroutines are famous for having low overhead costs related to memory usage and context switching (the details of this are worth a whole blog series, or at least a few links).
So why would we want to limit ourselves to only a handful of workers if the runtime infrastructure is just as efficient with hundreds of goroutines?
The worker pool pattern is stil a helpful tool when the tasks are heavy in file or network I/0 (TODO: Research a little more).

