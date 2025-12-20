---
layout: post
title: Orchestrator, The Worker II, API
date: 2025-12-19
---


Last time we defined methods for pulling
tasks off its queue and then starting or stopping them.

We need a way to
expose those core features so a manager, which will be the exclusive user,
running on a different machine can make use of them.

## API Comes to rescue

 API will be simple, perform these basic operations:
Send a task to the worker (which results in the worker starting the task
as a container)
Get a list of the worker’s tasks
Stop a running task

![alt text](</images/Screenshot 2025-12-19 225711.jpg>)


 web API. This choice means that the
worker’s API can be exposed across a network and that it will use the HTTP
protocol.

 three primary
components:
Handlers—Functions that are capable of responding to requests
Routes—Patterns that can be used to match the URL of incoming
requests
Router—An object that uses routes to match incoming requests with
the appropriate handler

define parameterized
routes. What is a parameterized route? It’s a route that defines a URL
where one or more parts of the URL path are unknown and may change
from one request to the next

Like `/tasks` is different from `/tasks/{taskID}`


 worker’s API is composed of an HTTP server from the
Go standard library; the routes, router, and handlers from the chi package; and,
finally, our own worker

| Method | Route           | Description                     |
|--------|-----------------|---------------------------------|
| GET    | `/tasks`        | Gets a list of all tasks        |
| POST   | `/tasks`        | Creates a task                  |
| DELETE | `/tasks/{taskID}` | Stops the task identified by taskID |


## The API struct

 defined in a file named api.go in the worker/

  `Address` and `Port` fields for server machine address and listening port

   it contains the` Worker` field, which will be a reference to
an instance of a `Worker` object. Remember, we said the API will wrap the
worker to expose the worker’s core functionality to the manager.

 the struct
contains the `Router` field, which is a pointer to an instance of `chi.Mux.`

type Api struct { 
    Address string 
    Port    int 
    Worker  *Worker 
    Router  *chi.Mux 
}



## handling requests


a handler is a function capable of responding to a
request. For our API to handle incoming requests, we need to define
handler methods on the API struct. We’re going to use the following three
methods, which I’ll list here with their method signatures:

```
StartTaskHandler(w http.ResponseWriter, r *http.Request)
GetTasksHandler(w http.ResponseWriter, r *http.Request)
StopTaskHandler(w http.ResponseWriter, r *http.Request)
```

 `StartTaskHandler`

 At a high level, this method reads the body of a request from r.Body,
converts the incoming data it finds in that body from JSON to an instance
of our task.TaskEvent type, and then adds that task.TaskEvent to
the worker’s queue. It wraps up by printing a log message and then adding
a response code to the http.ResponseWriter

 uses `DisallowUnknownFields()` method of `json.Decoder` struct, which will cause the Decode() method to return an error if there are
fields in the body that are not defined in the destination struct—in our case, task.TaskEvent

`GetTasksHandler`
 This method looks simple, but there is a lot going on inside it. It
starts off by setting the Content-Type header to let the client know we’re sending it JSON data. Then, similar to StartTaskHandler, it adds a
response code.
Gets an instance of a json.Encoder type by calling the
json.NewEncoder() method
Gets all the worker’s tasks by calling the worker’s GetTasks method
Transforms the list of tasks into JSON by calling the Encode method on
the json.Encoder object

 `StopTaskHandler`
see that stopping a task is accomplished by
sending a request with a path of DELETE /tasks/{taskID}.
This is all that’s needed to stop a task
because the worker already knows about the task: it has it stored in its Db field.  taskID, we have to convert it from a string, which
is the type that chi.URLParam returns to us, into a uuid.UUID type.
This conversion is done by calling the uuid.Parse()
we want to do is check whether the worker knows about
this task. If it doesn’t, we should return a response with a 404 status code.
If it does, we change the state to task.Completed and add it to the
worker’s queue.

in our StopTaskHandler we’re making a copy of the task in the worker’s
datastore


since  we’re using the worker’s datastore to
represent the current state of tasks, while we’re using the worker’s queue
to represent the desired state of tasks. So we need to dereference and then assign so it creates a copy.



## Serving The API (routes)

API struct that contains the two components that will
make this possible: the `Worker` and `Router` fields. Each of these is a
pointer to another type. The Worker field is a pointer to our own Worker
start and stop tasks and get a list of tasks the worker knows about.

`Router` field is a pointer to a Mux object provided by the chi package, and it will provide the functionality for defining routes and routing requests 
To serve the worker’s API we need to make two additions to the code
we’ve written so far. Both additions will be made to the `api.go` file


The first addition is to add the `initRouter()` method to the Api struct
It starts by creating an instance of a Router by calling
`chi.NewRouter()`. Then it goes about setting up the routes we defined in the table


Second Addition is add the `Start()` method to the Api struct,
 This method calls the `initRouter` method  then it starts an HTTP server that will listen for requests.
The ListenAndServe function is provided by the `http` package from
Go’s standard library.]

Third Addition is the in the `worker.go` library. A function in the `worker` struct GetTasks()
that returns all the tasks in the workers db. For the `GetTaskHandler` to use



## lets let our API serve us (main.go)

we’re going to continue our use of 
main.go, in which we’ll write our main function 
creates an
instance of our worker, w, which has a `Queue` and a `Db`. It creates an
instance of our `API`, api, which uses the host and port values that it
reads from the local environment. Finally, the` main()` function performs
the two operations that bring everything to life

call a function `runTasks` and pass it a
pointer to the worker `w`. But it also does something else. It has this funny
go term before calling the runTasks function. What is that about? If
you’ve used threads in other languages, the go `runTasks(&w)` line is
similar to using threads
start our API by calling `api.Start()`

`runTasks` function

fairly simple. It’s a continuous loop that checks the worker’s queue
for tasks and calls the worker’s RunTask method when it finds tasks that
need to run. For our own convenience, we’re sleeping for 10 seconds
between each iteration of the loop. This slows things down for us so we can
easily read any log messages.


 API is not calling any worker methods that perform task
operations (i.e., it is not starting or stopping tasks). Structuring our code in
this way allows us to separate the concern of handling requests from the
concern of performing the operations to start and stop tasks