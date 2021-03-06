# Beyond Erlang Actors: Kahn Process Networks

The most obvious starting point for deciding when you want something like a Kahn network is any time you find yourself
thinking in terms of a unix pipeline- data needing sequential processing, ideally with the additional ability to split
down separate paths and join back up later. These sorts of problems come up often in signal processing, input/output
transformations, complex event processing, "bit-shoveling," almost any problem that lends itself to distributed
computing, etc. In fact, with apologies to Greenspun, many Erlang systems end up as ad-hoc, informally-specified,
bug-ridden implementations of Kahn Process Networks.

A Kahn Network is best for any problem that makes the most sense as a data-flow model- as streams of data flowing
through various stages (possibly in parallel). Like the unix pipeline, it allows you develop the stages (processes- or,
in Erlang terminology, things you would normally supervise like gen_servers and gen_fsms) individually, in a very
functional way- optimizing abstraction, reuse, testability, and usually keeping the system simple and intuitive.

I will be logging the development of the Erlang Kahn library here as I go because it illustrates many important aspects
of the Erlang ecosystem and distributed computation. Because these are blog entries in the original sense of the word
(web-log), everything is subject to change. If you're just interested in using the final product as it currently stands,
head over to https://github.com/josephwecker/kahn for the simplified version of usage etc.

## Background for Kahn Process Networks

After several years of study and research at Stanford, Gilles Kahn returned to France and in 1974 published the paper
introducing what came to be known as Kahn Process Networks (or KPNs). It is a rather straight-forward formalization of
work and ideas that had been percolating mostly over at Bell Labs related to operating system design. It took the idea
of pipelines, gave them a graph structure, a nice concise mathematical treatment, and showed that it had some very
desirable behaviors for parallel systems.

> Summary--what's most important.
> To put my strongest concerns into a nutshell:
> 1. We should have some way of connecting programs like garden hose--screw in another segment when it becomes ...
> necessary to massage data in another way. This is the way of IO also.
> ...
>   - M. D. McIlroy, Oct 11, 1964, As the Multics project was beginning.
>   http://cm.bell-labs.com/cm/cs/who/dmr/mdmpipe.html

[Douglas McIlroy - (left) Original, (right) With simulated proper Unix beard courtesy of Dennis Ritchie.]

I bring some of this historical context up because it seems to be one of those topics that, since the early 60's, gets
reinvented/rediscovered every several years and given a new name, new semantics, and sometimes slightly different
analytical properties.

- http://en.wikipedia.org/wiki/Flow-based_programming
- http://en.wikipedia.org/wiki/Dataflow_programming
- http://en.wikipedia.org/wiki/Data-driven_programming
- http://en.wikipedia.org/wiki/Pipeline_programming
- http://en.wikipedia.org/wiki/Reactive_programming
- http://en.wikipedia.org/wiki/Functional_reactive_programming
- http://en.wikipedia.org/wiki/Event-driven_programming
- http://en.wikipedia.org/wiki/Stream_processing
- http://en.wikipedia.org/wiki/Complex_event_processing
- http://en.wikipedia.org/wiki/Event_Stream_Processing

- http://en.wikipedia.org/wiki/Dataflow
- http://en.wikipedia.org/wiki/Data_flow_diagram
- http://en.wikipedia.org/wiki/Dataflow_architecture
- http://en.wikipedia.org/wiki/Data-flow_analysis
- http://en.wikipedia.org/wiki/Data_stream
- http://en.wikipedia.org/wiki/Pipeline_(computing) 
- http://en.wikipedia.org/wiki/Pipeline_(software)
- http://en.wikipedia.org/wiki/Unix_pipeline
- http://en.wikipedia.org/wiki/Hartmann_pipeline
- http://en.wikipedia.org/wiki/Functional_flow_block_diagram
- http://en.wikipedia.org/wiki/Streaming_algorithm

- All of the entries on state machines, petri nets, activity diagrams, workflow modeling, most articles on visual
  programming languages, etc. etc...




* Processes are monotonic. They don't have to output at the same rate as incoming data, but their output must not
  re-order the flow.

* Possibly data sources that continue forever

* Possibly data sinks that do not output anything

* A system where the algorithms can all be continuous/online. That is, they can't access arbitrary past data or rely on
  having all the data in hand before outputting something. (For example, some of these:
  http://www.cs.ucr.edu/~marek/pubs/online.bib). 

* If a process holds state- ideally it will be steady-state, bounded in size.

* Processes are properly abstracted- they don't care who gives them input or who receives their output (implicitly at
  least, explicit cycles are fine if you know what you're doing)- just that they are of the correct type. Also, like any
  good Erlang program, these processes don't care about any sort of global state- their state (if any) is purely
  determined by the tokens that it has consumed.

* In a proper Kahn Network the processes and therefore network are completely deterministic. This is accomplished by
  assuming that writing outputs is always nonblocking but that reading inputs always blocks. This determinism makes
  certain analysis very tractable. We have two caveats:
  
  - Our Kahn network implementation basically uses Erlang's message-passing semantics, so token matching, blocking for a
    certain amount of time, waiting for certain data, etc.- are all possible if you really want. Even then most of then
    most of the time we'll still be deterministic, but in theory the Erlang VM could, on a second run using the exact
    same processes and tokens, timeslice the processes in such a way that a 'receive after ...' statement gets triggered
    where on the earlier run it did not. In other words, as soon as you include 'after' (or, worse, look at the incoming
    message queue and detect a token without consuming it- usually frowned on anyway in Erlang) you're system is no
    longer technically deterministic (and no longer technically a Kahn Process Network). So far in practice though I've
    found this to be rarely needed and easily accounted for.
  - Erlang's message queues (equivalent to the channels or buffered fifo structures between processes) are bounded by
    available physical memory. To ensure things stay balanced and that no queue gets too big, we'll occasionally (with a
    possible warning) block for a moment when sending the output from a process.

What Erlang adds to the mix: Natural semantic fit, supervisors and fault tolerance, easy scalability over multiple
machines, easily integrated visualization and high-level graph analysis, type-checking, hot-code-swapping, & lots of
other Erlang goodness. Also, because this is a practical endeavor and not an academic or theoretical endeavor, the
ability to dynamically create and remove, connect and disconnect, update and restart individual processes on the fly
will be available.


## Some Terminology

### Processes

The first problem- what to name these data-flow processes- the nodes in the data-flow graph...

- "Process" - Nope, used. In Erlang it already refers to the primitive light-weight processes that are ubiquitous in Erlang-
              our Kahn processes will often be Erlang processes but we can't call them the same thing.
- "Node"    - Used. Also already used by Erlang to describe a separate Erlang VM, possibly running on a separate machine
              across a network.
- "Module"  - Used.
- "Vertex"  - Meh. It makes sense from a graph-theory point of view but is long and a bit too generic.
- "Stage"   - Meh. Would work except that it implies the process always does some form of staging or transformation-
              not generic enough.
- "Function/Fun/Fn/Lambda..." - Can't. As we'll see, they tend to be much more like state machines (or in some cases
              gen-servers) with several functions- which would make calling it something like a function, even though it
              will hopefully be composable like a function, a misnomer.

To distinguish from Erlang processes and nodes, then, I'm simply going to call these data-flow processes 'nodum'
(or usually the singular, 'nodus'); the root word for node; Latin for knot. There will be three primary ways to define
or implement a nodus:

1. Using an Erlang function (either anonymous or by name) for nodum that are simple enough to be implemented in a simple
   loop (like multiplying two incoming values or other primitives).
2. Using an Erlang external port's input/output (if you want a more complicated port write a nodus module)- that is, a
   straightforward way to insert a system command into the flow just like a Unix pipeline.
3. Using the `-behaviour(nodus)` directive in a module that implements the proper callbacks which can act basically as a
   state-machine or server, receiving messages and outputting responses as appropriate.


### Ports

Not to be confused with Erlang Ports (basically external interfaces), we'll say that each Nodus has any number of input
ports and any number of output ports- each named and with a type specification that determines what kind of token it
accepts.

### Tokens

Equivalent to Erlang messages in the following form: `{token, InPort, FromPID, Data}` where the InPort is an atom that
corresponds to one of the recipient nodus' input-ports, the FromPID is an Erlang PID for the sending nodus and the
Data is any term that matches the type for that input-port.


With state




## Creating a new Erlang behaviour

