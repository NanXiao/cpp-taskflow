# Cpp-Taskflow

[![Build Status](https://travis-ci.org/twhuang-uiuc/cpp-taskflow.svg?branch=master)](https://travis-ci.org/twhuang-uiuc/cpp-taskflow)
[![Gitter chat][Gitter badge]][Gitter]
[![License: MIT](./image/license_badge.svg)](./LICENSE)

A fast C++ header-only library to develop complex and parallel task dependency graphs.

# Why Cpp-Taskflow?

Cpp-Taskflow helps you build efficient computational graphs quickly using modern C++17.
It is by far faster and more expressive than existing libraries such as [OpenMP Tasking][OpenMP Tasking] and 
[TBB FlowGraph][TBB FlowGraph].

# Get Started with Cpp-Taskflow

The following example [simple.cpp](./example/simple.cpp) contains the basic syntax you need to use Cpp-Taskflow.

```cpp
// TaskA---->TaskB---->TaskD
// TaskA---->TaskC---->TaskD

#include "taskflow.hpp"

int main(){
  
  tf::Taskflow tf(std::thread::hardware_concurrency());

  auto [A, B, C, D] = tf.silent_emplace(
    [] () { std::cout << "TaskA\n"; },
    [] () { std::cout << "TaskB\n"; },
    [] () { std::cout << "TaskC\n"; },
    [] () { std::cout << "TaskD\n"; }
  );  

  A.precede(B);  // B runs after A
  A.precede(C);  // C runs after A
  B.precede(D);  // D runs after B
  C.precede(D);  // C runs after D

  tf.wait_for_all();  // block until all tasks finish

  return 0;
}

```
Compile and run the code with the following commands:
```bash
~$ g++ simple.cpp -std=c++1z -O2 -lpthread -o simple
~$ ./simple
TaskA
TaskC  <-- concurrent with TaskB
TaskB  <-- concurrent with TaskC
TaskD
```

# Create a Taskflow Graph
Cpp-Taskflow has very expressive, neat, and clear methods to create complex dependency graphs.
Most applications are developed through the following three steps.

## Step 1: Create a Task
To start a task dependency graph, 
create a taskflow object and specify the number of working threads in a shared thread pool
to carry out tasks.
```cpp
tf::Taskflow tf(std::max(1u, std::thread::hardware_concurrency()));
```
Create a task via the method `emplace` and get a pair of `Task` and `future`.
```cpp
auto [A, F] = tf.emplace([](){ std::cout << "Task A\n"; return 1; });
```
Or create a task via the method `silent_emplace`, if you don't need a `future` to retrieve the result.
```cpp
auto [A] = tf.silent_emplace([](){ std::cout << "Task A\n"; });
```
Both methods implement variadic templates and can take arbitrary numbers of arguments to create multiple tasks at one time.
```cpp
auto [A, B, C, D] = tf.silent_emplace(
  [] () { std::cout << "Task A\n"; },
  [] () { std::cout << "Task B\n"; },
  [] () { std::cout << "Task C\n"; },
  [] () { std::cout << "Task D\n"; }
);
```

## Step 2: Define Task Dependencies
Once tasks are created in the pool, you need to specify task dependencies in a 
[Directed Acyclic Graph (DAG)](https://en.wikipedia.org/wiki/Directed_acyclic_graph) fashion.
The class `Task` supports different methods for you to describe task dependencies.

**Precede**: Adding a preceding link forces one task to run ahead of one another.
```cpp
A.precede(B);  // A runs before B.
```

**Broadcast**: Adding a broadcast link forces one task to run ahead of other(s).
```cpp
A.broadcast(B, C, D);  // A runs before B, C, and D.
```

**Gather**: Adding a gathering link forces one task to run after other(s).
```cpp
A.gather(B, C, D);  // A runs after B, C, and D.
```

**Linearize**: Linearizing a task sequence adds a  preceding link to each adjacent pair.
```cpp
tf.linearize(A, B, C, D);  // A runs before A, B runs before C, and C runs before D.
```

## Step 3: Execute the Tasks
There are three methods to carry out a task dependency graph, `dispatch`, `silent_dispatch`, and `wait_for_all`.
```cpp
auto future = tf.dispatch();  // non-blocking, returns with a future immediately.
tf.dispatch();                // non-blocking, no return
```
Calling `wait_for_all` will block until all tasks complete.
```cpp
tf.wait_for_all();
```

# Debug a Taskflow Graph
Concurrent programs are notoriously difficult to debug. 
We suggest (1) naming tasks and dumping the graph, and (2) starting with single thread before going multiple.
Currently, Cpp-Taskflow supports [GraphViz][GraphViz] format.

```cpp
// debug.cpp
tf::Taskflow tf(0);  // force the master thread to execute all tasks
auto A = tf.silent_emplace([] () { /* ... */ }).name("A");
auto B = tf.silent_emplace([] () { /* ... */ }).name("B");
auto C = tf.silent_emplace([] () { /* ... */ }).name("C");
auto D = tf.silent_emplace([] () { /* ... */ }).name("D");
auto E = tf.silent_emplace([] () { /* ... */ }).name("E");

A.broadcast(B, C, E); 
C.precede(D);
B.broadcast(D, E); 

std::cout << tf.dump();
```
Run the program and inspect whether dependencies are expressed in the right way.
```bash
~$ ./debug
digraph Taskflow {
  "A" -> "B"
  "A" -> "C"
  "A" -> "E"
  "B" -> "D"
  "B" -> "E"
  "C" -> "D"
}
```
There are a number of free [GraphViz tools][AwesomeGraphViz] you could find online
to visualize your Taskflow graph.

<img src="image/graphviz.png" width="25%"> Taskflow with five tasks and six dependencies, generated by [Viz.js](http://viz-js.com/).

# API Reference

## Taskflow

The class `tf::Taskflow` is the main place to create taskflow graphs and carry out task dependencies.
The table below summarizes its commonly used methods.

| Method          | Argument    | Return                           | Description |
| --------------- | ----------- | ------------ | ----------- |
| emplace         | callable(s) | task, future | insert nodes to the graph to execute the given callables; results can be retrieved from the returned future object |
| silent_emplace  | callable(s) | task         | insert nodes to the graph to execute the given callables |
| placeholder     | none        | task         | insert a node to the graph without any work; work can be assigned later |
| linearize       | task list | none         | create a linear dependency in the given task list |
| parallel_for    | range, callable | task pair | apply the callable in parallel to each element in the given range and return a task pair as barriers; the range is specified in two iterators | 
| dispatch        | none        | future | dispatch the current graph and return a shared future to block on completeness |
| silent_dispatch | none        | none | dispatch the current graph | 
| wait_for_all    | none        | none | dispatch the current graph and block until all graphs including previously dispatched ones finish |
| num_nodes       | none        | size | return the number of nodes in the current graph |  
| num_workers     | none        | size | return the number of working threads in the pool |  
| num_topologies  | none        | size | return the number of dispatched graphs |
| dump            | none        | string | dump the current graph to a string of GraphViz format |

## Task

Each `tf::Taskflow::Task` object is a light-weight handle for you to create dependencies in its associated graph. 
The table below summarizes its methods.

| Method         | Argument    | Return | Description |
| -------------- | ----------- | ------- | ----------- |
| name           | string    | self    | assign a human-readable name to the task |
| work           | callable  | self    | assign a work of a callable object to the task |
| precede        | task      | self    | enable this task to run *before* the given task |
| broadcast      | task list | self    | enable this task to run *before* the given tasks |
| gather         | task list | self    | enable this task to run *after* the given tasks |
| num_dependents | none        | size | return the number of dependents (inputs) of this task |
| num_successors | none        | size | return the number of successors (outputs) of this task |

# Caveats
While Cpp-Taskflow enables the expression of very complex task dependency graph that might contain 
thousands of task nodes and links, there are a few amateur pitfalls and mistakes to be aware of.

+ Having a cycle in a graph may result in running forever.
+ Trying to modify a dispatched task can result in undefined behavior.
+ Touching a taskflow from multiple threads are not safe.

The current version is known to work on most Linux distributions and OSX.
We havn't found issues in a particular platform.
Please report to us if any.

# System Requirements
To use Cpp-Taskflow, you only need a C++17 compiler:
+ GNU C++ Compiler G++ v7.2 with -std=c++1z
+ Clang 5.0 C++ Compiler with -std=c++17

# Compile Unit Tests and Examples
Cpp-Taskflow uses [CMake](https://cmake.org/) to build examples and unit tests.
We recommend using out-of-source build.
```bash
~$ cmake --version  # must be at least 3.9 or higher
~$ mkdir build
~$ cmake ../
~$ make 
```
## Unit Tests
Cpp-Taskflow uses [Doctest](https://github.com/onqtam/doctest) for unit tests.
```bash
~$ ./unittest/taskflow
```

Alternatively, you can use CMake's testing framework to run the unittest.
```bash
~$ cd build
~$ make test
```

## Examples
The folder `example/` contains several examples and is a great place to learn to use Cpp-Taskflow.

# Get Involved
+ Report bugs/issues by submitting a [Github issue][Github issues].
+ Submit contributions using [pull requests][Github pull requests].
+ Live chat and ask questions on [Gitter][Gitter].

# Contributors

+ Tsung-Wei Huang
+ Chun-Xun Lin
+ You!

# License

Copyright © 2018, [Tsung-Wei Huang](http://web.engr.illinois.edu/~thuang19/), [Chun-Xun Lin](https://github.com/clin99), and Martin Wong.
Released under the [MIT license](https://github.com/twhuang-uiuc/cpp-taskflow/blob/master/LICENSE).


[Gitter]:                https://gitter.im/cpp-taskflow/Lobby
[Gitter badge]:          ./image/gitter_badge.svg
[Github releases]:       https://github.com/twhuang-uiuc/cpp-taskflow/releases
[Github issues]:         https://github.com/twhuang-uiuc/cpp-taskflow/issues
[Github pull requests]:  https://github.com/twhuang-uiuc/cpp-taskflow/pulls
[GraphViz]:              https://www.graphviz.org/
[AwesomeGraphViz]:       https://github.com/CodeFreezr/awesome-graphviz
[OpenMP Tasking]:        http://www.nersc.gov/users/software/programming-models/openmp/openmp-tasking/
[TBB FlowGraph]:         https://www.threadingbuildingblocks.org/tutorial-intel-tbb-flow-graph


