# RxJS-DFD Notation v0.1

**Full name:** RxJS Dataflow Diagram Notation with Timeline Semantics  
**Short name:** RxJS-DFD  
**Version:** v0.1  
**Purpose:** A reusable visual notation for RxJS pipeline visualizations.

RxJS-DFD combines a **vertical dataflow diagram** with a **horizontal timeline** so that an RxJS pipeline can show both:

```text
top -> bottom   = dataflow through sources, operators, and sinks
left -> right   = time
```

The goal is to make RxJS behavior visible:

```text
where values flow
when values arrive
which operators transform values
which operators suppress values
which operators delay values
which operators join values
which operators create inner streams
which operators cancel, queue, share, or replay work
```

---

## 1. Core idea

An RxJS pipeline such as:

```text
source$ -> operator -> operator -> sink
```

is drawn as:

```text
Source node
   |
   v
Operator node
   |
   v
Operator node
   |
   v
Sink node
```

Each stage owns a horizontal timeline lane.

```text
source$ row = values emitted by the source
map row     = values emitted after map
filter row  = values emitted after filter
delay row   = values emitted after delay
sink row    = values received by the subscriber / resulting Observable
```

The most important rule:

```text
Every source value keeps a visible lineage from source to sink.
```

Even when a value does not produce an output at some stage or time, the diagram should still make that absence visible.

---

## 2. Dataflow vocabulary mapped to RxJS

| Dataflow term | RxJS-DFD meaning |
|---|---|
| **Node** | Source, operator, join, split/share, higher-order operator, or sink |
| **Port** | Input or output side of a node |
| **Arc** | Observable/subscription connection between stages |
| **Token** | A visible RxJS notification, usually `next(value)` |
| **Activation / firing** | A node reacts because an upstream notification arrived |
| **Graph** | The whole RxJS pipeline topology |

RxJS-specific refinement:

```text
A token is more precisely a notification-token.
```

RxJS has three notification kinds:

```text
next(value)
error(err)
complete()
```

---

## 3. Node types

| Node type | Visual role | RxJS examples |
|---|---|---|
| **Source node** | Top of the diagram; emits values; output only | `timer`, `interval`, `fromEvent`, `of`, `ajax`, custom Observable |
| **Operator node** | Middle stage; consumes upstream values and may emit downstream values | `map`, `filter`, `scan`, `delay`, `debounceTime` |
| **Join node** | Multiple input sources, one output | `zip`, `merge`, `combineLatest`, `withLatestFrom`, `forkJoin` |
| **Split/share node** | One upstream execution, multiple downstream consumers | `share`, `shareReplay`, `Subject` |
| **Higher-order node** | Creates dynamic inner stream lanes | `mergeMap`, `switchMap`, `concatMap`, `exhaustMap` |
| **Sink node** | Bottom of diagram; receives values; input only | `subscribe`, render, log, side effect, resulting Observable view |

Important rule:

```text
An operator is both a sink and a source.
```

It is a **sink** relative to its upstream stage because it consumes input notifications.  
It is a **source** relative to its downstream stage because it emits new notifications.

---

## 4. Port rules

```text
source node        = output port only
sink node          = input port only
operator node      = input port + output port
join operator      = multiple input ports + one output port
split/share node   = one input port + multiple output ports
higher-order node  = outer input port + dynamic inner input/output lanes
```

Example join:

```text
numbers$ ----\
              zip --> output$
letters$ ----/
```

`zip` has two input ports and one output port.

---

## 5. Arc rules

An arc is the connection through which notification-tokens travel.

In RxJS-DFD:

```text
horizontal lane = stream over time
vertical arrow  = value moves to next stage at the same timestamp
diagonal arrow  = value moves to next stage at a shifted timestamp
branch arrow    = shared/split flow
join arrow      = multi-input coordination
```

RxJS qualification:

```text
Observable definition + subscription = active runtime arc
```

Before subscription, the pipeline is only a description. After subscription, notifications flow through the runtime arcs.

---

## 6. Time axis

The timeline is always horizontal:

```text
0ms -> 500ms -> 1000ms -> 1500ms -> ...
```

Use `timer(0, period)` when the first emission should appear at `0ms`.

```ts
timer(0, 500).pipe(take(5))
```

Use `interval(period)` when the first emission should appear after the first period.

```ts
interval(500).pipe(take(5))
```

This distinction matters because RxJS-DFD diagrams are time-accurate.

---

## 7. Token symbols

| Symbol | Meaning |
|---|---|
| Labeled circle, for example `(4,c)` | Actual `next(value)` emission |
| Empty circle `○` | No output at that stage/time, but lineage is still shown |
| Dashed empty circle `◌` | Value was consumed but suppressed/dropped, for example by `filter` |
| Cancel marker `x` or crossed circle | Pending emission or inner stream was cancelled |
| Replay-highlighted circle | Value replayed from memory to a late subscriber |
| `C` marker | `complete()` notification |
| `E` marker | `error(err)` notification |

Important distinction:

```text
Token identity and token value are not the same thing.
```

Example:

```text
source lineage: source1#2 + source2#2
zip value:      (2,c)
map value:      (4,c)
```

The token lineage stays the same, even though the displayed value changes.

---

## 8. Value lineage rule

Every source value gets a visible path through the whole diagram.

Example with `delay(500)`:

```text
source at 0ms: 0
map at 0ms:    0
sink at 0ms:   ○
sink at 500ms: 0
```

The empty circle at `0ms` means:

```text
The value exists in the pipeline, but no sink output exists at this time yet.
```

The labeled `0` at `500ms` means:

```text
The delayed value has now reached the sink/resulting Observable.
```

---

## 9. Arrow symbols

| Arrow | Meaning |
|---|---|
| `↓` vertical arrow | Same-time propagation |
| `↘` diagonal arrow | Time shift introduced by an operator |
| `→` horizontal lane arrow | Stream continues over time |
| `↘ +500ms` | Value moved right by 500ms |
| dashed arrow | Potential, pending, inactive, or pre-subscription path |
| cut/cancel marker | Active inner stream or pending emission is cancelled |
| fan-out | One upstream execution is shared/split into multiple consumers |
| fan-in | Multiple input streams are coordinated by a join operator |

---

## 10. Immediate operator rule

Operators that do not change time use vertical arrows.

Example:

```ts
map(x => x * 2)
```

```text
source row:  1 at 500ms
             |
             v
map row:     2 at 500ms
```

The value changes. The timestamp does not.

Rule:

```text
vertical arrow = same logical time
```

---

## 11. Transformation rule

Transforming operators emit a new value at the same timestamp unless combined with a scheduler or time operator.

Examples:

```text
map
pluck
pairwise
scan
reduce, when it finally emits
```

For `map`:

```text
input token:  x at t
output token: f(x) at t
```

---

## 12. Filtering / suppression rule

For `filter`, the input token reaches the operator, but may not pass.

Example:

```ts
filter(x => x % 2 === 0)
```

```text
source: 0  1  2  3  4
filter: 0  ◌  2  ◌  4
```

Rule:

```text
passed token  = normal labeled circle
suppressed token = dashed or empty circle at the operator output row
```

A suppressed value may continue as an empty lineage marker if the diagram is tracking every source value down to the sink.

---

## 13. Time-shifting operator rule

Time operators are shown with diagonal arrows to the right.

Example:

```ts
delay(500)
```

```text
map row:    2 at 500ms
              \ +500ms
               v
delay row:  2 at 1000ms
```

Rule:

```text
A time-shifting operator draws an arrow from the input timeline position
into the operator output row at the shifted timeline position.
```

This is essential for operators such as:

```text
delay
debounceTime
throttleTime
auditTime
sampleTime
timeout
bufferTime
windowTime
```

But each time operator has a different policy.

---

## 14. Time operator policies

### 14.1 `delay(duration)`

Each input token is shifted right by the same duration.

```text
input:   0 ---- 1 ---- 2
output:      0 ---- 1 ---- 2
```

Rule:

```text
Draw one diagonal arrow from input token at t to output token at t + duration.
```

---

### 14.2 `debounceTime(duration)`

A value starts a waiting window. A newer value before the window closes cancels the previous pending value.

```text
input:   a -- b ---- c --------
output:            b ---- c ----
```

Rule:

```text
Draw a candidate arrow to t + duration.
If a newer token arrives before that time, mark the candidate as cancelled.
Only the last token after silence reaches the output row.
```

---

### 14.3 `throttleTime(duration)`

The first accepted value passes, then a time window suppresses later values.

```text
input:   a -- b -- c ---- d
output:  a ------------ d
```

Rule:

```text
Draw the accepted token.
Draw a rightward suppression window.
Tokens inside the window become suppressed circles unless trailing behavior is explicitly enabled.
```

---

### 14.4 `auditTime(duration)`

A value starts a window. The latest value seen inside the window emits at the end.

```text
input:   a -- b -- c ---- d
output:            c ---- d
```

Rule:

```text
Draw a window span.
Inside the window, update the remembered latest token.
At window end, emit the latest remembered token.
```

---

### 14.5 `sampleTime(duration)`

A periodic clock samples the latest remembered source value.

```text
source:  a -- b ----- c
clock:   |----|----|----|
output:       b    c
```

Rule:

```text
Draw clock ticks as vertical sample markers.
At each tick, emit the latest remembered source token if one exists.
```

---

## 15. Stateful operator rule

A stateful operator remembers information between tokens.

Examples:

```text
scan
reduce
distinctUntilChanged
buffer
window
combineLatest
zip
shareReplay
switchMap
concatMap
mergeMap
exhaustMap
```

Rule:

```text
A stateful node may emit a value and update internal memory at the same timestamp.
```

Optional state annotation:

```text
state: S0 -> S1 -> S2 -> S3
```

Example with `scan`:

```text
source: 1   2   3
scan:   1   3   6
state:  1   3   6
```

---

## 16. Join operator rules

Join operators are visualized as multiple input ports feeding one output row.

### 16.1 `merge`

Policy:

```text
fire when any input emits
```

Rule:

```text
Any input token may flow down to the output row.
Output order follows actual emission time.
```

---

### 16.2 `zip`

Policy:

```text
pair nth token from each source
```

Example:

```text
numbers$: 0 ---- 1 ---- 2
letters$: a -- b -- c
zip:      (0,a) (1,b) (2,c)
```

Rule:

```text
Output time = when the slowest required nth token has arrived.
Faster source tokens wait inside zip's internal queues.
```

---

### 16.3 `combineLatest`

Policy:

```text
wait until all sources emitted once, then fire when any source emits
```

Example:

```text
a$:   1 -------- 2
b$:      x -- y
out:     (1,x) (1,y) (2,y)
```

Rule:

```text
Before all inputs are primed: no output.
After all inputs are primed: any input token creates an output using latest remembered values.
```

---

### 16.4 `withLatestFrom`

Policy:

```text
primary source controls output
```

Example:

```text
primary$:   p1 ---- p2 ---- p3
state$:        s1 ------ s2
out:              (p2,s1) (p3,s2)
```

Rule:

```text
Secondary inputs update memory only.
Only the primary source emits output.
```

---

### 16.5 `forkJoin`

Policy:

```text
wait for all sources to complete, then emit final values once
```

Rule:

```text
Show remembered final value per source.
Output appears only at the completion coordination point.
```

---

## 17. Higher-order operator rules

Higher-order operators create dynamic inner streams.

Common form:

```text
outer token
   |
   v
flattening node creates inner lane
```

Each inner Observable should be visible as a temporary lane or nested flow.

---

### 17.1 `mergeMap`

Policy:

```text
allow overlap
```

Rule:

```text
Each outer token creates an inner lane.
All active inner lanes may emit concurrently.
Outputs interleave by actual time.
```

---

### 17.2 `switchMap`

Policy:

```text
only latest
```

Rule:

```text
A new outer token cancels the previous inner lane.
Cancelled future inner emissions are marked with cancellation symbols.
Only the latest active inner stream may continue.
```

---

### 17.3 `concatMap`

Policy:

```text
queue
```

Rule:

```text
If an inner stream is active, later outer tokens wait in a queue.
The next inner stream starts only after the previous inner stream completes.
```

---

### 17.4 `exhaustMap`

Policy:

```text
ignore while busy
```

Rule:

```text
The first accepted outer token starts an inner stream.
Outer tokens arriving while the inner stream is active are ignored.
The next accepted outer token appears only after the current inner stream completes.
```

---

## 18. Sharing and replay rules

### 18.1 `share`

Policy:

```text
one upstream execution, multiple downstream consumers
```

Diagram form:

```text
source$
   |
   v
share
 /   \
sinkA sinkB
```

Rule:

```text
Use a fan-out marker to show that consumers share the same upstream execution.
```

---

### 18.2 `shareReplay(1)`

Policy:

```text
shared execution + remembered latest output token
```

Late subscriber example:

```text
source output:   A ---- B ---- C
sinkA:           A ---- B ---- C
sinkB starts:              ^
sinkB receives:            B ---- C
```

Rule:

```text
Late subscriber lane is dashed before subscription.
At subscription time, show replayed value with a replay marker.
Then live values continue normally.
```

---

## 19. Cancellation and unsubscription

Cancellation is not a value. It is a teardown of an active runtime arc.

Rule:

```text
Cancellation cuts an active inner lane or pending emission path.
Future planned emissions from that cancelled lane are shown as cancelled, not emitted.
```

Most important operator:

```text
switchMap
```

Example:

```text
outer A starts inner A
outer B arrives
inner A is cancelled
inner B starts
```

---

## 20. Completion and error

RxJS-DFD should show terminal notifications when they matter.

| Marker | Meaning |
|---|---|
| `C` | `complete()` |
| `E` | `error(err)` |
| stopped lane | no more notifications possible |
| recovery branch | `catchError` replaces a failed stream with another stream |

Rule:

```text
After error or complete reaches a stream boundary, no later next-values can appear on that arc,
unless an operator replaces or recovers the stream before termination reaches the sink.
```

---

## 21. Formal minimal grammar

```text
Diagram =
  TimeAxis
  + Stage[]

Stage =
  NodeBox
  + TimelineLane

NodeBox =
  Source
  | Operator
  | Join
  | SplitShare
  | HigherOrder
  | Sink

TimelineLane =
  ArcBoundaryAfterNode

Token = {
  id: source identity or lineage identity,
  time: timestamp,
  value: displayed value,
  kind: next | error | complete,
  status: emitted | empty | suppressed | buffered | cancelled | replayed
}

Arrow =
  verticalSameTime
  | diagonalTimeShift
  | horizontalStreamProgress
  | joinConvergence
  | splitFanout
  | cancellation
```

---

## 22. Minimal legend

```text
Time axis
----------->
left to right = time

Pipeline axis
|
v
top to bottom = operator order

Labeled circle
(4,c)
actual next(value)

Empty circle
○
no output at this stage/time

Dashed empty circle
◌
value consumed but suppressed/dropped

Vertical arrow
↓
same-time propagation

Diagonal arrow
↘ +500ms
time shift introduced by an operator

Horizontal lane
----------->
stream continues over time

Cancelled marker
x
inner stream or pending emission cancelled

Replay marker
highlighted circle
value replayed from memory to late subscriber

Complete marker
C
complete notification

Error marker
E
error notification
```

---

## 23. Canonical example: `map + delay`

Pipeline:

```ts
const result$ = timer(0, 500).pipe(
  take(5),
  map(x => x * 2),
  delay(500)
);
```

Timeline:

```text
source$ : 0      1      2      3      4
map     : 0      2      4      6      8
delay   :    0      2      4      6      8
sink    : ○  0      2      4      6      8
```

Interpretation:

```text
0 at source 0ms flows down to map at 0ms.
map emits 0 at 0ms.
delay shifts that 0 to 500ms.
sink has an empty circle at 0ms and receives 0 at 500ms.
```

---

## 24. Canonical request phrase

Use this phrase when requesting future diagrams:

```text
Create an RxJS-DFD Notation v0.1 diagram for this pipeline.
Use top-to-bottom dataflow and left-to-right time.
Show every source value lineage down to the sink.
Use vertical arrows for same-time propagation,
diagonal arrows for time shifts,
empty circles for no output,
and cancellation/replay markers where needed.
```

---

## 25. Final definition

**RxJS-DFD Notation v0.1** is a reusable visual notation for RxJS pipelines.

It displays sources at the top, sinks at the bottom, operators between them, and a horizontal timeline across all stages.

Each source value keeps a visible lineage through the pipeline.

Operators may transform, suppress, delay, combine, split, queue, cancel, replay, complete, or error notification-tokens.

```text
straight down = same-time dataflow
diagonal right = time movement
empty circle = no output made visible
circle with value = emitted notification-token
```

RxJS-DFD makes the RxJS machine visible:

```text
values move over time
operators rewire the movement
subscription activates the graph
notifications drive execution
```
