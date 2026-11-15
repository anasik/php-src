# Generic closure function-JIT regression benchmark

`workloads/jit_closure_churn.php` creates and invokes 10,000 closures from one
generic factory, changing the captured value each time. One invocation warms the
specialization before timing. Each sample runs in a fresh PHP process.

```sh
sapi/cli/php -n \
  -d opcache.enable_cli=1 \
  -d opcache.file_update_protection=0 \
  -d opcache.jit=function \
  -d opcache.jit_buffer_size=16M \
  bench-harness/workloads/jit_closure_churn.php 10000
```

Use `opcache.jit=disable` for the interpreter control. Compare release builds
with identical configure/compiler settings. The previous implementation is
commit `d8ee975efe14bbc41113eeed9ccbd852dbc935b4`.

Measured on Linux x86-64 on 2026-11-15; median of seven fresh processes per
configuration, with a 16 MiB JIT buffer:

| Configuration | Median milliseconds | JIT bytes consumed after warmup | Request memory growth, bytes |
| --- | ---: | ---: | ---: |
| Previous implementation, function JIT | 2057.24 | 4,800,000 | 3,081,344 |
| Request-local closure cache, function JIT | 11.44 | 0 | 853,120 |
| JIT disabled, same revised build | 9.10 | 0 | 853,120 |

The cache removes repeated IR compilation and entry-table allocation for this
binding. It does not make this tiny closure faster than interpretation: lookup
and dispatch still cost time. Wall-clock values vary on the shared host and are
illustrative, not a general application speedup claim. The code-buffer and memory
figures were stable across the seven samples.

The cache identity includes the template handler table, lexical scope, method
binding, function flags and concrete signature. Different captured values and
closure-local static variables still belong to each closure. `bindTo()` and
`Closure::call()` use the new scope; equal scopes with different substituted
signatures remain separate. First-class method callables are also covered.

Cache entries retain an owned function prototype and signature, not the closure
object or its captured/bound objects. Entries are discarded at request shutdown.
Consequently, reuse is within a request; this does not establish cross-request
native-code reuse for request-owned specializations. Distinct specializations
still consume code and metadata, and buffer exhaustion still falls back to the VM.

The dispatch offset now uses `sizeof(zend_op)` and handler-pointer size instead
of a fixed 64-bit shift. Execution was tested on Linux x86-64. x86-32 runtime
testing was unavailable because the host lacks 32-bit C startup objects/libraries;
AArch64 and Windows runtime validation also remain outstanding.
