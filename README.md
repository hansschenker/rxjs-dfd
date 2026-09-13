# RxJS-DFD

**RxJS-DFD** means **RxJS Dataflow Diagram**.

It is a reusable visual notation for drawing RxJS pipelines as **dataflow diagrams with timeline semantics**.

```text
top -> bottom = dataflow through source, operators, and sink
left -> right = time
```

The goal is to make RxJS behavior visible: where values flow, when they arrive, which operators transform or suppress them, where time is shifted, where values are joined, and where cancellation, sharing, replay, completion, or error occurs.

## Main idea

An RxJS pipeline such as:

```text
source$ -> operator -> operator -> sink
```

is drawn vertically:

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

Each stage also owns a horizontal timeline lane:

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

Even when a value does not produce an output at some stage or time, the diagram should make that absence visible with an empty circle.

## Why this notation exists

RxJS is often difficult to understand from code alone because the execution behavior is distributed across operators. RxJS-DFD combines:

- a **dataflow graph** for structure
- a **timeline** for runtime behavior
- **token lineage** for following values through the pipeline
- explicit markers for time shifts, suppression, cancellation, replay, completion, and error

This makes it easier to reason about RxJS as values and notifications moving over time.

## Core visual rules

```text
straight down     = same-time propagation
diagonal right    = time movement introduced by an operator
circle with value = emitted next(value)
empty circle      = no output at that stage/time
dashed circle     = consumed but suppressed/dropped value
cancel marker     = pending emission or inner stream cancelled
C                 = complete()
E                 = error(err)
```

## Canonical example

```ts
const result$ = timer(0, 500).pipe(
  take(5),
  map(x => x * 2),
  delay(500)
);
```

Timeline sketch:

```text
source$ : 0      1      2      3      4
map     : 0      2      4      6      8
delay   :    0      2      4      6      8
sink    : ○  0      2      4      6      8
```

The first empty circle at the sink means: the source value exists in the pipeline, but the delayed output has not reached the sink yet. The delayed `0` then appears at the sink 500ms later.

## Specification

See the full notation document:

- [RxJS-DFD Notation v0.1](./RxJS-DFD-Notation-v0.1.md)

## Contributors

**Main contributor to the formal formulation:** GPT / OpenAI ChatGPT

**Concept direction, naming, validation, and RxJS learning context:** Hans Schenker

The notation was developed through an iterative discussion about mapping classic dataflow-programming vocabulary—nodes, ports, arcs, and tokens—to RxJS sources, operators, sinks, Observable connections, and notifications over time.

## Status

Version: **v0.1**

This is an early notation specification intended for future RxJS pipeline visualization, RxJS teaching material, and possible integration with RSL / RxJS visual tooling.
