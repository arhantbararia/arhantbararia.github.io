---
layout: post
title: Orchastrator, skeleton
date: 2025-11-11
---


 I like to translate
 that model into skeleton code.
 This typically allows me to start thinking
 about the implementation in a concrete way without getting
 too deep into the weeds just yet



 most obvious components are the manager, worker, and
 scheduler. The foundation of each of these components,
 however, is the task,


The basic skeleton I found online.
![alt text](<../images/Screenshot 2025-12-09 014025.jpg>)



## The task skeleton

### `State`: 
Pending:  task has been enqueued
Scheduled: system has determined there
 is a machine that can run the task, but it is in the process of
 sending the task to the selected machine, or the selected
 machine is in the process of starting the task. 

Running:  if the
 selected machine successfully starts the task

 Completed:  task completing its work
successfully or being stopped by a user, the task moves into

Failed:  at any point the task crashes or
 stops working as expected, the task then moves into a state


![alt text](<../images/Screenshot 2025-12-09 014218.jpg>)

###  `ID `:
we’ll use universally
 unique identifiers (UUID)

 Task struct { 
    ID      uuid.UUID 
    Name    string 
    State   State 
}

 Note that the State field is of type State, which we
 defined previously.

 ###`Image`:
 what Docker image a task should use,

### `Memory` and `Disk` will help the:
 system identify the number of resources a task needs.


### `ExposedPorts` and `PortBindings` are used by Docker

### `RestartPolicy` 

### `StartTime`

### `FinishTime`

```
type Task struct { 
    ID            uuid.UUID 
    ContainerID   string 
    Name          string 
    State         State 
    Image         string 
    CPU           float64 
    Memory        int64 
    Disk          int64 
    ExposedPorts  nat.PortSet 
    PortBindings  map[string]string 
    RestartPolicy string 
    StartTime     time.Time 
    FinishTime    time.Time 
}

```


 how does a user tell the system to stop a
task


 . 
├── main.go 
├── manager 
│   └── manager.go 
├── node 
│   └── node.go 
├── scheduler 
│   └── scheduler.go 
├── task 
│   └── task.go 
└── worker 
    └── worker.go



 ## TaskEvent struct 
  The event will need a
 State, which will indicate the `state` the task should
 transition to (e.g., from Running to Completed)


 Timestamp   Timestamp to record the time the event
 was requested

  Task
 It
 will be an internal object that our system uses to trigger
 tasks from one state to another.



 type TaskEvent struct { 
    ID        uuid.UUID 
    State     State 
    Timestamp time.Time 
    Task      Task 
}


## The Worker Skeleton 
 we can think of the worker as the next layer
 that sits atop the foundation


 1. Run tasks as Docker containers
 2. Accept tasks to run from a manager
 3. Provide relevant statistics to the manager for the
 purpose of scheduling tasks
 4. Keep track of its tasks and their state

  worker will need to
 run and keep track of tasks. To do that, the worker will use a
 field named `Db`, which will be a map of UUIDs to tasks.

 accepting tasks from a
 manager, the worker will want a `Queue` field.


  `TaskCount` field as a convenient way of
 keeping track of the number of tasks a worker has at any
 given time


type Worker struct {
	Name  string
	Queue queue.Queue
	Db    map[uuid.UUID]*task.Task
	TaskCount int
	
}


some methods that will do the actual work.
`RunTask` method
will handle running a task on the machine where the worker
 is running.
RunTask method will be responsible for identifying the
 task’s current state and then either starting or stopping a
 task based on the state


 `StartTask` `StopTask` method, which will do exactly as their name says


` CollectStats` method, which can be used to periodically
 collect statistics about the worker



## The Manager Skeleton

1. Accept requests from users to start and stop tasks
 2. Schedule tasks onto worker machines
 3. Keep track of tasks, their states, and the machine on
 which they run

  The Manager will have a queue, represented by
 the `pending` field, The queue will allow the Manager to
 handle tasks on a FIFO basis


 two in-memory databases: one to store tasks and another to
 store task events. The databases are maps of strings to
 `Task` and `TaskEven`t, respectively.

 workers, which will be a slice of strings need to keep track of the workers in the
 cluster


 the jobs that are assigned to each worker.

the jobs that are assigned to each worker. We’ll use a field
 called `WorkerTaskMap`, which will be a map of strings to
 task UUIDs


  easy way to
 find the worker running a task given a task name. Here we’ll
 use a field called `TaskWorkerMap`, which is a map of task
 UUIDs to strings, where the string is the name of the worker.


  you can see that the manager needs
 to schedule tasks onto workers.

 create a method on
 our Manager struct called `selectWorker` to perform that
 task. This method will be responsible for looking at the
 requirements specified in a Task evaluating the
 resources available in the pool of workers to see which
 worker is best suited to run the task.


  `UpdateTasks` method Ultimately, this
 method will end up triggering a call to a worker’s
 CollectStats method

  `SendWork`  the Manager obviously needs to send tasks
 to workers.




 ##  The scheduler skeleton

  1. Determine a set of candidate workers on which a task
 could run
 2. Score the candidate workers from best to worst
 3. Pick the worker with the best score

 round-robin algorithm

 identified which worker got the most recent task. Then,
 when the next task came in, the scheduler simply picked the
 next worker in its list


 Furthermore, I might want more flexibility
 in how tasks are assigned to workers. Can try different algos then

 interface to specify the methods a type
 must implement to be considered a Scheduler.

 these methods are
 SelectCandidateNodes, Score, and Pick

 
 Scheduler interface { 
    SelectCandidateNodes() 
    Score() 
    Pick() 
}

## The node skeleton

Worker has a physical aspect to it, however, in
 that it runs on a physical machine itself, and it also causes
 tasks to run on a physical machine. Moreover, it needs to
 know about the underlying machine to gather stats about
 the machine that the manager will use for scheduling
 decisions. We’ll call this physical aspect of the Worker a
 `Node`

 a node is an object that represents
 any machine in our cluster
 `Name`


 `Ip`which the manager will want
 to know in order to send tasks to it

 physical machine also
 has a certain amount of `Memory` and `Disk` space that tasks
 can use. These attributes represent maximum amounts


 , the tasks on a machine will be using some
 amount of memory and disk, which we can call
 `MemoryAllocated` and `DiskAllocated`

  zero or more tasks, which we can track using a
 `TaskCount` field.

 ## The main runner

we will define the main.go as: 

  Create a Task object
 Create a TaskEvent object
 Print the Task and TaskEvent objects
 Create a Worker object
 Print the Worker object
 Call the worker’s methods
 Create a Manager object
 Call the manager’s methods
 Create a Node object
 Print the Node objec

 