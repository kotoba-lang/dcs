# dcs-clj (分散制御システム)

[![CI](https://github.com/kotoba-lang/dcs/actions/workflows/ci.yml/badge.svg)](https://github.com/kotoba-lang/dcs/actions/workflows/ci.yml)

Handle a **Distributed Control System** — plant/area hierarchy, I/O tags,
PID control loops, and ISA-18.2-style alarms — as plain EDN data, in
portable Clojure. Every core namespace is `.cljc` with **zero third-party
runtime deps**, so it runs on the JVM, ClojureScript, and Clojure-on-WASM
hosts (SCI). A control system config is data you can `assoc`, diff, store,
or generate; the library adds structural validation, a pure PID algorithm,
an ISA-18.2 alarm state machine, and a pure scan-cycle engine around it.

Sibling of the other reusable domain kernels in this org
([ddl](https://github.com/kotoba-lang/ddl),
[dmn](https://github.com/kotoba-lang/dmn),
[bpmn](https://github.com/kotoba-lang/bpmn)).

## Scope

This is a **basic process control (BPC)** model — regulatory control loops,
process alarms, and the tag registry they operate on. It is not a Safety
Instrumented System (SIS): interlocks/trips with independent SIL ratings are
a distinct, separately-certified concern and out of scope here. It does not
claim compatibility with any commercial DCS vendor's proprietary
configuration format; it is an open, EDN-first substrate for that same
domain.

## The model: a DCS system as EDN (`dcs.model`)

Areas, tags, loops and alarms are id-keyed maps for O(1) lookup:

```clojure
{:dcs/areas  {"area-1" {:dcs/id "area-1" :dcs/name "Reactor"}}
 :dcs/tags   {"TIC-101.PV" {:dcs/id "TIC-101.PV" :dcs/kind :ai
                            :dcs/units :degC :dcs/range [0 500]}}
 :dcs/loops  {"TIC-101" {:dcs/id "TIC-101" :dcs/pv-tag "TIC-101.PV"
                         :dcs/output-tag "TIC-101.OUT" :dcs/mode :auto
                         :dcs/setpoint 350.0
                         :dcs/tuning {:kp 2.0 :ki 0.1 :kd 0.0}
                         :dcs/output-limits [0.0 100.0]}}
 :dcs/alarms {"TIC-101.HI" {:dcs/id "TIC-101.HI" :dcs/tag "TIC-101.PV"
                            :dcs/type :hi :dcs/setpoint 420.0
                            :dcs/priority :high :dcs/deadband 2.0}}}
```

A threading-friendly builder:

```clojure
(require '[dcs.model :as m])

(-> (m/system)
    (m/add-area (m/area "area-1" {:name "Reactor"}))
    (m/add-tag  (m/tag "TIC-101.PV" :ai {:units :degC :range [0 500]}))
    (m/add-tag  (m/tag "TIC-101.OUT" :ao {:range [0.0 100.0]}))
    (m/add-loop (m/ctrl-loop "TIC-101" {:pv-tag "TIC-101.PV" :output-tag "TIC-101.OUT"
                                        :mode :auto :setpoint 350.0
                                        :tuning {:kp 2.0 :ki 0.1 :kd 0.0}
                                        :output-limits [0.0 100.0]}))
    (m/add-alarm (m/alarm "TIC-101.HI" {:tag "TIC-101.PV" :type :hi :setpoint 420.0
                                        :priority :high :deadband 2.0})))
```

## Validation (`dcs.validate`)

Structural checks — never throws, returns a vector of problems:

```clojure
(require '[dcs.validate :as v])

(v/validate system)
;; => [{:dcs/error :unknown-pv-tag :dcs/loop "TIC-101" :dcs/tag "TIC-101.PV"} ...]
(v/valid? system) ; => boolean
```

Checks: dangling pv-tag/output-tag references, wrong tag kind (pv-tag must
be `:ai`, output-tag must be `:ao`), setpoint/output-limits outside the
tag's `:dcs/range`, negative tuning gains, dangling alarm tag references,
alarm setpoint outside the tag's range, negative deadband, and
`:hi`/`:hi-hi` (or `:lo`/`:lo-lo`) threshold ordering across alarms sharing
a tag.

## PID (`dcs.pid`)

A pure, positional-form PID with clamped anti-windup and bumpless
auto/manual transfer — no I/O, no host loop:

```clojure
(require '[dcs.pid :as pid])

(pid/step {:kp 2.0 :ki 0.1 :kd 0.0} (pid/init-state) pv sp dt [0.0 100.0])
;; => {:dcs/output 42.3 :dcs/state' {...}}

;; at the :manual -> :auto edge, start the output where the operator left it:
(pid/transfer-to-auto tuning pv sp last-manual-output)
```

## Alarms (`dcs.alarm`)

An ISA-18.2-inspired state machine (`:normal -> :unacked -> :acked ->
:normal`, with a `:rtn-unacked` branch when the condition clears before
acknowledgment), plus deadband hysteresis on `:hi`/`:hi-hi`/`:lo`/`:lo-lo`:

```clojure
(require '[dcs.alarm :as alarm])

(alarm/step alarm-cfg value prior-state) ; => {:dcs/state' s :dcs/event e-or-nil}
(alarm/ack :unacked)                     ; => :acked
```

## Scan-cycle engine (`dcs.execute` + `dcs.ports`)

`dcs.ports` defines the host-injected `IFieldIO` protocol only (read/write a
tag by id); the host supplies the implementation (a real fieldbus driver,
an OPC UA client, a simulator). `dcs.execute/scan` is pure over that
protocol — it performs no I/O of its own:

```clojure
(require '[dcs.execute :as exec])

(exec/scan system state io dt)
;; => {:dcs/state' state' :dcs/events [{:dcs/alarm "TIC-101.HI" :dcs/from :normal :dcs/to :unacked}]}
```

Each scan advances every `:auto` loop's PID (writing its output tag) and
every alarm's state machine; `:manual` and `:cascade` loops are skipped
(their output/setpoint is driven externally).

## Dry-run runner (`dcs.runner`, JVM only)

A conservative, host-side runner for offline commissioning checks: validate
a system, then simulate N scan cycles entirely in memory against a
caller-supplied PV trajectory. No fieldbus, no network, no live process I/O.

```clojure
(require '[dcs.runner :as runner])

(runner/dry-run system
                 {:n-cycles 10 :dt 1.0
                  :initial-values {"TIC-101.PV" 300.0}
                  :pv-overrides [nil nil {"TIC-101.PV" 425.0} ...]})
;; => {:dcs/valid? true :dcs/problems [] :dcs/trace [{:dcs/cycle 0 :dcs/tags {...} :dcs/events [...]} ...]}
```

Returns immediately with `:dcs/valid? false` and no simulation if
`dcs.validate/validate` finds any structural problems.

## Why a shared library (org placement)

Per kotoba-lang's role/scope taxonomy, reusable domain models live in
`kotoba-lang`. dcs-clj carries no plant-specific configuration and no real
fieldbus/network bindings (those are host-injected via `dcs.ports`) — it is
the dependency, not a deployment.

## Test

```sh
clojure -M:test
```
