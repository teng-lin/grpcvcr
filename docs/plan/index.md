# grpcvcr 0.2–0.3 Refactor Plan

Status: converged for 0.2; the v3 schema and the Python 3.10 support question
remain open (see [Decision gates and risks](#decision-gates-and-risks))  
Last updated: 2026-08-30 · Owner: TBD · Next review: before the `0.1.2` tag

!!! warning "Not user documentation"

    This is an internal design plan for unreleased 0.2/0.3 work. Shipped
    behavior is described in Concepts and API Reference. Where the two
    disagree, the shipped documentation is authoritative for the current
    release — except where this plan explicitly records that the shipped
    documentation is wrong today (see [Documentation defects](#documentation-defects-in-the-current-release)).

This document defines the implementation plan for refactoring `grpcvcr` into a
safe, deterministic, and lifecycle-faithful record/replay library. The plan was
reviewed iteratively from four perspectives: gRPC runtime behavior,
security/data-format design, public API/release compatibility, and internal
consistency against the current codebase.

The immediate target is a safe `grpc.aio` integration for a downstream pilot
consumer. The design remains general-purpose, preserves a compatibility path
where one is possible, and deliberately moves faithful request-streaming and
bidirectional streaming to a later schema and release.

## Goals

- Replay without constructing a live gRPC channel, resolving a host, or
  obtaining credentials.
- Record and replay unary-unary and unary-stream calls faithfully for sync and
  `grpc.aio` clients.
- Preserve streaming laziness, metadata, status, deadlines, callbacks, and
  cancellation behavior at the public gRPC API boundary.
- Sanitize protobuf and metadata content before it can enter a cassette,
  diagnostic, temporary file, or library log.
- Match deterministic, sanitized request projections and consume repeated
  interactions in a defined order.
- Replace Python import-path coupling with stub-provided serializers and
  deserializers.
- Introduce a validated, versioned, atomically written cassette format.
- Preserve existing public entry points where the strict security model permits
  it, and state every place it does not.
- Keep the implementation compatible with Python 3.10 for downstream consumers
  that require it, while treating upstream Python 3.10 support as a maintainer
  decision.

## Non-goals for 0.2

- Recording or replaying new client-streaming or bidirectional-streaming calls.
  Only the frozen v1 adapter replays pre-existing ones.
- Exact HTTP/2 scheduling, flow-control, or transport timing reproduction.
- Sandboxing application-provided sanitizers, credential providers, or hooks.
- Transparently routing request-streaming `NEW_EPISODES` calls by their full
  request body.
- Silently converting unframed v1 client-streaming or bidi recordings.

## Release scope

| Release | Published | Scope |
| --- | --- | --- |
| `0.1.2` | PyPI | Ships the correctness fixes already merged on main: scoped target recording, a partially repaired coverage gate (the pytest-plugin exclusion survives; pull request 4 finishes it), replayed-error fidelity, and cassette portability. A tag push after pull request 1 lands the publish hardening; it consumes no pull request of its own. |
| `0.2` | PyPI | Cassette v2, sync and async unary-unary/unary-stream, strict security profile, compatibility facades, v1 playback adapter, migration tooling, pytest and release hardening. |
| `0.3` | PyPI | Cassette v3, client-streaming and bidi, after a causal-event prototype validates the design. |

There is no `0.2` beta. The downstream pilot installs the pull request 3 branch
by commit, which pins more precisely than a pre-release and consumes no version.

`0.1.2` is not a prerequisite for rollback. `v0.1.0` and `v0.1.1` are tagged and
published, so `0.1.1` is already pinnable. `0.1.2` exists only to ship the twenty
merged commits that would otherwise wait for `0.2`, three of which change replay
behavior for existing cassettes. Dropping it is defensible and has no technical
consequence beyond leaving those fixes unshipped; it is a one-row change to this
table.

Note that `.github/workflows/release.yml` triggers on any `v*` tag and runs
build → publish → GitHub release, and `hatch-vcs` derives the version from that
tag. A `v0.1.2` tag is therefore a PyPI release, not a marker. A rollback marker
without a release requires a non-`v` tag such as `baseline-0.1.x`.

Release notes for `0.2` need a home before it ships. `CHANGELOG.md` currently has
only an `[Unreleased]` section despite two published tags, while the project
metadata points `Changelog` at GitHub releases. Pull request 5 picks one.

`grpcvcr` is pre-1.0. Semver 2.0.0 permits any change under `0.y.z`, so `0.2` may
remove client-streaming and bidi recording, replace the cassette format, change
the default matcher, and rename fixtures without a major bump and with no
required deprecation cycle. Folding v3 into `0.2` would therefore carry no semver
cost. v3 stays separate for schedule reasons only: it is 25–35 engineer-days
gated on prototypes that do not yet exist, and folding it in blocks the
pilot-ready async milestone behind them.

## Breaking changes in 0.2

This section is normative. `0.2` is a breaking release and its release notes must
lead with this list.

### Client-streaming and bidi recording is removed

`RecordingStreamUnaryInterceptor` and `RecordingStreamStreamInterceptor`
(`grpcvcr.interceptors`) record and replay client-streaming and bidirectional
calls in 0.1.x, and are covered by `tests/test_interceptor_paths.py`,
`tests/test_interceptor_paths_async.py`, and `tests/test_streaming_errors.py`.

Under a v2 profile those shapes raise `UnsupportedRpcShapeError`. The frozen v1
adapter continues to replay existing v1 recordings of them. This is the single
largest behavioral removal in the release.

`Channel.stream_unary` and `Channel.stream_stream` still return a working
multicallable object; construction never raises. Generated stubs build every
multicallable in `Stub.__init__`, so raising at factory time would break stub
construction for any service that merely declares an unsupported method.
`UnsupportedRpcShapeError` is raised from the multicallable's `__call__`, before
the request iterator is touched and before any transport factory, invocation
preparer, or credential provider runs.

`UnsupportedRpcShapeError` is new in 0.2. It subclasses `GrpcvcrError` and is
exported from the package root, so existing `except GrpcvcrError` handlers
continue to catch it.

`MessageTooDeepError`, `NonCanonicalPayloadError`, `UnresolvableAnyPayloadError`,
and `ProfileMismatchError` are likewise new in 0.2, subclass `GrpcvcrError`, and
are exported from the package root. The first three additionally subclass
`SerializationError`, so existing handlers around cassette load and save keep
working.

### A `grpcvcr` console script is added

`0.2` adds `[project.scripts] grpcvcr = "grpcvcr.__main__:main"`; `pyproject.toml`
declares no `[project.scripts]` today. Subcommands: `profile-hash` (print the
canonical profile document and its `config_sha256`), `validate` (validate a
cassette against its schema and profile), and `migrate` (v1 → v2 dry-run and
conversion).

The CLI is public surface and is snapshotted by pull request 1's baseline like
any other export. Phase 5 wires the console script; because `profile-hash` is
described under [Profile identity](#profile-identity) and is needed to debug a
`ProfileMismatchError` as soon as profiles exist, phase 2 lands it as a module
entry point first.

### The `grpcvcr.interceptors` subpackage is replaced

`src/grpcvcr/interceptors/__init__.py` exports five public names:
`RecordingUnaryUnaryInterceptor`, `RecordingUnaryStreamInterceptor`,
`RecordingStreamUnaryInterceptor`, `RecordingStreamStreamInterceptor`, and
`create_interceptors`. The v2 routing architecture replaces interception
entirely.

Pull request 1 snapshots all five. Pull request 4 decides, per name, whether it
becomes a facade over the routing engine or is removed with a documented
deprecation. The async interceptors in `interceptors/aio.py` are not re-exported
from the subpackage's `__all__` and are therefore not part of the formal public
surface.

### Exception attributes carrying raw data are removed or retyped

`NoMatchingInteractionError` is exported from the package root and today carries
`.request` — the raw serialized request bytes — and `.available`, every recorded
interaction in the cassette. Both are exactly the exfiltration path the
[Security boundary](#security-boundary) forbids.

Under the `protobuf-safe` profile these attributes are removed. `.method`
survives. The equivalent ruling for the other exceptions:

| Exception | Attribute | Strict-profile disposition |
| --- | --- | --- |
| `NoMatchingInteractionError` | `.request`, `.available` | Removed. Replaced by `.diagnostic`, a `SafeMismatchReport`. |
| `NoMatchingInteractionError` | `.method` | Retained. |
| `RecordingDisabledError` | `.method` | Retained. |
| `CassetteWriteError` | `.path`, `.cause` | `.path` retained. `.cause` removed; the message no longer interpolates the OS error. |
| `SerializationError` | `.cause` | Removed. |

`CassetteWriteError` is exported but never raised anywhere in `src/` or `tests/`;
`CassetteSerializer.save` raises `SerializationError` on write failure
(`src/grpcvcr/serialization.py:475`). Pull request 4 resolves it as wired-up or
removed rather than carrying a dead export through the compatibility snapshot.

These attributes are removed from the class unconditionally in 0.2, and
`NoMatchingInteractionError` gains `.diagnostic`. `legacy-raw` raises
`LegacyNoMatchingInteractionError`, a subclass that re-adds `.request` and
`.available`, so `except NoMatchingInteractionError` continues to catch both and
only code that actually reads the raw attributes must change. Every instance of a
given class then has a fixed attribute set, rather than one whose shape depends
on the active profile.

Attribute removal is necessary but not sufficient. Python populates `__context__`
implicitly on any exception raised inside an `except` block, and
`raise … from None` sets `__cause__ = None` and `__suppress_context__ = True`
while leaving `__context__` fully populated — the default traceback formatter
hides it, `e.__context__` and third-party chain-walking reporters do not.

Strict-profile exceptions therefore explicitly clear the chain at the raise site:

```python
try:
    ...
except Exception:
    err = CassetteWriteError(path)
    err.__cause__ = None
    err.__context__ = None
    err.__suppress_context__ = True
    raise err from None
```

A security test asserts, for every strict-profile exception, that
`__cause__ is None` **and** `__context__ is None`, and that no canary appears in
`traceback.format_exception(e)` or in `repr(e.__dict__)`. Asserting on the
rendered default traceback alone would pass while the data is still reachable.

Normative for every exception raised under `protobuf-safe`, existing or future:
the message is a constant plus values drawn only from the safe set — method path,
shape, FQN, field path, ordinal, consumption state, status code, digest, limit
name and configured value, profile id and digest, and the caller-supplied
cassette path. No payload bytes, no decoded field values, no metadata values, no
OS error text, and no filesystem path the caller did not supply. This covers the
exceptions introduced by this plan — `UnsupportedRpcShapeError`,
`MessageTooDeepError`, `UnresolvableAnyPayloadError`, `NonCanonicalPayloadError`,
and `ProfileMismatchError` — whose natural implementations would each interpolate
exactly what must not escape. A new exception type joins the strict surface only
with a test asserting canary absence from its message, `repr`, `__dict__`, and
formatted traceback.

### Matching semantics change

`DEFAULT_MATCHER` is `MethodMatcher()` today — method path only — and is the
default for `Cassette.match_on`, `recorded_channel`, `async_recorded_channel`,
and the `grpcvcr_cassette` fixture. The strict selector is strictly narrower, so
cassettes that match today on method alone will miss.

`CustomMatcher`, `RequestMatcher`, `MetadataMatcher`, and `TargetMatcher` keep
their signatures. Under a naive port they would keep running in strict mode and
receive `InteractionRequest` objects whose body is the sanitized projection
rather than wire bytes, failing in two opposite directions:

| Matcher | Naive-port failure |
| --- | --- |
| `CustomMatcher` | Silently **misses**: a matcher that deserializes `get_body_bytes()` and compares a now-redacted field never matches. |
| `RequestMatcher` | Silently **over-matches**: it compares whole bodies, and under deny-by-default two genuinely different requests differing only in dropped fields project to identical bytes, so it returns the response recorded for a different request. This is strictly worse than a miss, and `RequestMatcher` is the matcher most often composed with `MethodMatcher`. |
| `MetadataMatcher` | Silently misses on any key outside the allowlist, which records nothing by default. |
| `TargetMatcher` | Silently misses whenever `target_persistence` is false, its default: the cassette stores `target: None` while the live request carries one. |
| `AllMatcher` | Inherits whichever failure its members have. |

That port is rejected. Strict mode uses a distinct ABC, `StrictMatcher`, whose
`matches` receives a `SafeRequestProjection`. Passing a v1 `Matcher` — including
`MethodMatcher`, `RequestMatcher`, `CustomMatcher`, `MetadataMatcher`,
`TargetMatcher`, and any `__and__` composition of them — to a strict session
raises at configuration time, naming the `StrictMatcher` equivalent. The v1
`Matcher` hierarchy remains fully functional under `legacy-raw`, where bodies are
still wire bytes and today's semantics are unchanged.

`StrictMetadataMatcher` against a key outside the metadata allowlist, and
`StrictTargetMatcher` against a session with `target_persistence: false`, are
likewise configuration errors raised at session construction rather than on first
mismatch.

### Cassette surface changes

`Cassette.interactions` returns v1 `Interaction` objects. Backed by a v2
cassette under the strict profile it returns immutable views, not `Interaction`.
`Cassette.record_interaction` is a raw path and is available only under
`legacy-raw`.

Three more public `Cassette` methods break, and all three are unavoidable:

| Method | Break |
| --- | --- |
| `get_response(method, request_body, metadata)` | Returns `Interaction`, so it inherits the immutable-view ruling above. Its documented `Raises:` contract is `NoMatchingInteractionError`, whose attributes are removed above — and this method is the call site that passes `request_body` and `self.interactions` into it. It is also a stateless find-and-return, incompatible with the atomic reserve/consume lifecycle. |
| `find_interaction(InteractionRequest)` | Same stateless-lookup problem: it cannot reserve, so it cannot participate in ordered consumption. |
| `save()` | Gains new failure modes under the storage transaction — source-hash conflict, symlinked destination, lock acquisition — on a method that today raises only `CassetteWriteError`. |

`Cassette("test.yaml")` string-path coercion, `use_cassette`,
`Matcher.__and__` composition, and the `grpcvcr.serialization` names
(`InteractionRequest`, `InteractionResponse`, `StreamingInteractionResponse`,
`Interaction`, `CassetteData`, `CassetteSerializer`) are all in the pull request 1
snapshot.

`CassetteSerializer` dispatches on file extension to JSON or YAML today, and
`Cassette.path` documents both. **Schema v2 is YAML-only.** Opening a `.json`
path in a v2 record-capable mode is an error naming the restriction; the v1
adapter continues to read `.json` for playback.

### pytest plugin surface changes

- `grpcvcr_channel` is removed. It raises `NotImplementedError` unconditionally
  today and is undocumented in `docs/guides/pytest.md`, so removal is preferred
  over making it functional.
- The `grpcvcr_cassette` fixture's default `match_on` changes with
  `DEFAULT_MATCHER`, per [Matching semantics change](#matching-semantics-change).
- If the CI-detection decision resolves toward implementing it, the precedence
  order in `grpcvcr_record_mode` changes and CI stops defaulting to recording.

The surface pull request 1 must snapshot is six fixtures
(`grpcvcr_cassette_dir`, `grpcvcr_record_mode`, `grpcvcr_cassette`,
`grpcvcr_channel`, `grpcvcr_channel_factory`, `grpcvcr_async_channel_factory`),
two CLI options (`--grpcvcr-record`, `--grpcvcr-cassette-dir`), and one marker
with both positional and keyword forms. `grpcvcr_record_mode` and
`grpcvcr_async_channel_factory` are public but undocumented.

### Shipped all-shape documentation becomes false

`README.md:32`, `docs/index.md:32`, `docs/concepts/index.md:28`,
`docs/concepts/streaming.md:3`, and `CHANGELOG.md:13` all advertise support for
all four RPC types, with worked client-streaming and bidi examples. The exact
wording differs by file — "supports all four gRPC call types" in `docs/concepts/`,
"All RPC types" in `README.md`, `docs/index.md`, and `CHANGELOG.md` — so a sweep
matching one string will miss three files. True today, false in 0.2.

## Documentation defects in the current release

These are shipped bugs in `0.1.x`, not `0.2` breaking changes, and they are
listed separately so they do not enter the `0.2` breaking-change release notes.
Pull request 1 fixes them because they are wrong *today*.

| File | Claim | Status |
| --- | --- | --- |
| `docs/guides/ci-testing.md` | "grpcvcr automatically detects CI environments and sets `RecordMode.NONE` by default … CI is detected via the `CI` environment variable" | **False today.** No module under `src/grpcvcr/` reads any environment variable; `grpcvcr_record_mode` falls through to `NEW_EPISODES`. CI defaults to *recording*, so following this guide means live calls and credentials written into cassettes from CI. |
| `CHANGELOG.md:22` | "Automatic `RecordMode.NONE` in CI environments" | Same defect. |
| `docs/guides/ci-testing.md:13-19` | Worked example using fixtures `cassette` and `grpc_target` | **Neither fixture exists.** The plugin provides `grpcvcr_cassette` and no target fixture at all, so the example cannot run. |

The CI-detection claim must be resolved by decision, not by deletion alone:
either implement `CI` environment detection in 0.2, or remove the claim from the
guide and the changelog. Pull request 1 removes the claim; implementing detection
is tracked as a separate 0.2 decision. The fix covers the whole guide, not the
one sentence: `docs/guides/ci-testing.md` also carries a
"**RecordingDisabledError**: Attempted to record in CI" troubleshooting entry and
a "Force Recording in CI" section, both predicated on the same nonexistent
detection.

## Target architecture

```text
Generated stub
      │
      ▼
RoutingChannel / AsyncRoutingChannel
      │
      ├── encode request once
      ├── produce safe match projection
      ├── reserve cassette interaction
      │
      ├── HIT ─────────────► ReplayCall
      │                        ├── public call ABC
      │                        ├── metadata/status
      │                        └── lazy messages
      │
      └── LIVE ────────────► DeferredLiveCall
                               ├── shared transport initialization
                               ├── per-call live preparation
                               ├── delegated real call
                               └── terminal finalization
```

`RoutingChannel` and `AsyncRoutingChannel` are new types introduced in 0.2.

The v2 engine directly implements the public gRPC channel, multicallable, and
shape-specific call interfaces. It does not depend on private gRPC call classes.

Client-interceptor compatibility differs by stack and is stated explicitly:

- Sync: `grpc.intercept_channel` is public. A `RoutingChannel` is a valid
  `grpc.Channel` and may be wrapped by it, so user interceptors compose without
  grpcvcr reimplementing anything.
- Aio: gRPC has no public interception helper. `grpc.aio` applies interceptors
  inside its own channel implementation via `Intercepted*Call` classes, of which
  only `InterceptedUnaryUnaryCall` is exported; the unary-stream, stream-unary,
  and stream-stream variants are private.

Therefore `AsyncRoutingChannel` owns its own interception chain for the shapes it
supports. It accepts `interceptors: Sequence[grpc.aio.ClientInterceptor]`,
partitions them by shape as `grpc.aio._channel.Channel` does, and drives the
`continuation(client_call_details, request) -> Call` contract itself, including:
a `continuation` that may be invoked at most once, an interceptor that returns
either a call object or a plain response, `ClientCallDetails` replacement, and
the response-iterator wrapper an interceptor sees for unary-stream. This chain is
grpcvcr code with its own conformance tests against the behavior of `grpc.aio`
with the same interceptors on a real channel; it is not an import of gRPC
internals.

Existing `RecordingChannel`, `AsyncRecordingChannel`, `.channel`, `.cassette`,
and `.target` APIs remain compatibility facades over the new engine.

The compatibility obligation is every name in `grpcvcr.__all__` (21 exports at
`0.1.x`; 22 in 0.2 once `UnsupportedRpcShapeError` joins) and
`grpcvcr.interceptors.__all__` (5 exports), **plus** the module-level names in
`grpcvcr.matchers` and `grpcvcr.serialization` that the public API exposes
transitively. Those two modules and `grpcvcr.cassette` declare no `__all__`, so an
`__all__`-only definition would exclude names this document already treats as
breaking changes — `DEFAULT_MATCHER`, `find_matching_interaction`, and the
`serialization` types listed under
[Cassette surface changes](#cassette-surface-changes). Pull request 1 snapshots
the full enumerated list and pull request 4 resolves each name as facade,
deprecation, or documented removal.

The existing synchronous `async_recorded_channel` helper remains available
through a deprecation cycle. It never awaits `channel.close()` today
(`src/grpcvcr/channel.py:236-239`), unlike `recorded_channel`, which calls
`recording.close()`; the deprecation shim must either close the channel or
document the leak. A new proper async context manager performs awaited close and
finalization.

## Glossary

| Term | Definition |
| --- | --- |
| Shape | One of `unary_unary`, `unary_stream`, `stream_unary`, `stream_stream`. |
| `CallDescriptor` | Immutable per-invocation facts handed to the live-invocation preparer: method path, shape, request/response FQNs, request metadata, and remaining deadline. Carries no request payload. |
| `LiveInvocation` | The preparer's return value: additional request metadata, optional per-call credentials, and an opaque application context object. |
| `RealCall` | The underlying `grpc`/`grpc.aio` call object returned by the delegated transport, once it exists. |
| `SafeOutcome` | The sanitized terminal result of a call — status code, details (present only under `status_details: keep`, otherwise `None`), and sanitized trailing metadata — passed to completion hooks. Never carries payload bytes. |
| `SafeRequestProjection` | The deterministic, sanitized, validated protobuf serialization of a request, used for matching and persistence. Never the bytes sent on the wire. |
| `PersistableEvent` | A validated cassette event (`client_message`, `server_initial_metadata`, `server_message`, `server_trailing_metadata`, `terminal_status`) constructed only from sanitized inputs. |
| `SafeMismatchReport` | The structured diagnostic attached to a strict `NoMatchingInteractionError`: method, shape, safe FQNs, candidate ordinal, consumption state, and safe digests. |
| `MethodDescriptorRegistry` | The stub-provided map from method path to request/response protobuf descriptors and to serializers/deserializers, plus the `google.protobuf.descriptor_pool.DescriptorPool` those descriptors were resolved from. Required in strict mode; replaces cassette-stored Python import paths. The pool resolves packed `Any` types and validates `Any`-rooted policy rules; it defaults to `descriptor_pool.Default()` and may be replaced with an explicitly constructed pool to bound which types are resolvable. |
| Opaque-context finalization | The single guaranteed invocation of the completion/failure hook with the `LiveInvocation` context and a `SafeOutcome`, after which the context is released. Runs exactly once per live call, including on cancellation and startup failure. |
| Generation | One session's in-memory interaction list. `ALL` starts an empty generation rather than loading the file, so the file is replaced, not merged. |
| Global strict-sequence playback | The cassette's interactions are consumed in recorded ordinal order; interaction *n* must be the *n*th invocation, and a mismatch is an error. |
| First-unused matching | The first still-`unused` interaction whose selector matches is consumed, regardless of ordinal. Permits reordered and interleaved calls. |
| Sidecar writer lock | An exclusive advisory OS lock on a `<cassette>.lock` file adjacent to the cassette, held for the duration of a writable session. |
| Source hash | SHA-256 of the cassette file bytes, read under the sidecar writer lock at session entry and re-read under the same held lock immediately before commit. It detects a writer that did not take the lock — a `0.1.x` process, an editor, or a branch switch — not a second locked session, which is refused at entry. |
| Causal client-message watermark | On each v3 server event, the ordinal of the highest client message the server had observed when it produced that event. Preserves request/response causality without claiming wall-clock timing. |

The two recording profiles are named exactly once, here, and used verbatim
elsewhere: `protobuf-safe` (the strict profile) and `legacy-raw` (the permissive
profile). "Strict" is an adjective meaning "under the `protobuf-safe` profile".

## Channel and call conformance

The routing implementation must cover the documented public surfaces for the
shapes supported in 0.2.

| Shape | Sync surface | `grpc.aio` surface |
| --- | --- | --- |
| Unary-unary | `__call__`, `with_call`, `future`; the returned object implements `grpc.Call` and `grpc.Future` | `__call__` returns a `grpc.aio.UnaryUnaryCall` |
| Unary-stream | `__call__` only (`grpc.UnaryStreamMultiCallable` has no `with_call`/`future`); the returned object implements `grpc.Call`, `grpc.Future`, and `grpc.RpcContext`, is an iterator, and is itself a `grpc.RpcError` subclass | `__call__` returns a `grpc.aio.UnaryStreamCall` |

Every multicallable `__call__` accepts `(request, timeout=None, metadata=None,
credentials=None, wait_for_ready=None, compression=None)`; the aio variants are
keyword-only after `request`. All four channel factory methods accept
`(method, request_serializer=None, response_deserializer=None,
_registered_method=False)`. `_registered_method` is accepted and ignored on every
supported grpcio version; generated stubs have passed it since grpcio 1.63 and
the supported floor predates it.

Required call members:

- Sync `grpc.Call`: `code`, `details`, `initial_metadata`, `trailing_metadata`,
  `is_active`, `time_remaining`, `cancel`, `add_callback`.
- Sync `grpc.Future`: `result(timeout)`, `exception(timeout)`,
  `traceback(timeout)`, `cancel`, `cancelled`, `running`, `done`,
  `add_done_callback`. Required on **both** sync shapes: gRPC returns
  `_MultiThreadedRendezvous` from `unary_stream.__call__` and from
  `unary_unary.future()`, and that type subclasses `grpc.Future` in both cases.
  `result()` on a streaming call is near-meaningless but must exist.
- Sync unary-stream: `__iter__`, `__next__`, and `next()`.
- gRPC's sync call objects also subclass `grpc.RpcError`, so
  `isinstance(call, grpc.RpcError)` is `True` on a real channel. grpcvcr does
  not reproduce that by identity — it raises a separate `_RecordedRpcError`
  because `grpc._channel._InactiveRpcError` is private — so the conformance
  suite asserts the member set and documents the `isinstance` deviation rather
  than inheriting it.
- Aio `grpc.aio.Call`: `initial_metadata`, `trailing_metadata`, `code`,
  `details`, `cancel`, `cancelled`, `done`, `time_remaining`,
  `add_done_callback`, `wait_for_connection`.
- Aio unary-unary: `__await__`. Aio unary-stream: `__aiter__` and `read()`.
- `debug_error_string()` is not on any ABC but is present on every real call and
  error object, and applications call it. Its signature differs by object, not
  by stack:
    - Synchronous on sync call objects (`grpc._channel._MultiThreadedRendezvous`)
      and on `grpc.aio.AioRpcError`.
    - A coroutine only on `grpc.aio` *call* objects (`grpc.aio.Call`).
    - Absent from the `grpc.RpcError` base class; grpcvcr's replayed sync error
      type must define it explicitly, as gRPC's own concrete rendezvous types do.

    Strict mode returns a constant, non-identifying string; it never carries a
    target, peer, or recorded diagnostic text.
- `grpc.aio.AioRpcError.code()`, `details()`, `initial_metadata()`, and
  `trailing_metadata()` are synchronous, unlike the same-named coroutines on
  `grpc.aio.Call`. A replayed error object must preserve that asymmetry; making
  them coroutines breaks every `except AioRpcError as e: e.code()` handler.

The channels also implement:

- Sync `subscribe`, `unsubscribe`, idempotent `close`, and connectivity
  callbacks.
- Aio `channel_ready`, `get_state`, `wait_for_state_change`, async context
  management, and idempotent awaited `close(grace)`.
- Virtual `READY` behavior during playback without creating a live channel.

Aio unary-stream response consumption matches `grpc.aio` exactly:

- `read()` returns `grpc.aio.EOF` at end of stream, never `None`.
- After OK termination, `read()` returns `grpc.aio.EOF` idempotently.
- After non-OK termination, every `read()` raises the same `AioRpcError`.
- After local cancellation, `read()` raises `asyncio.CancelledError`.
- `__aiter__` returns one cached async generator per call; repeated `async for`
  over a single call resumes, it does not restart. The current
  `_AsyncFakeStreamingCall.__aiter__` returns a fresh generator each time, so a
  second loop replays from message zero — that is a live bug, fixed in pull
  request 3.
- Mixing `read()` and `__aiter__` on one call raises `grpc.aio.UsageError` with
  gRPC's message. This applies to unary-stream in 0.2, not only to the read/write
  shapes deferred to 0.3.

Metadata typing at the public boundary:

- Aio `initial_metadata()`, `trailing_metadata()`, and the metadata carried by a
  raised `grpc.aio.AioRpcError` are `grpc.aio.Metadata` instances, not tuples.
  `Metadata` is public and exported; returning a tuple breaks `md["key"]` and
  `get_all`.
- Sync `initial_metadata()` and `trailing_metadata()` are tuples of
  `(key, value)` pairs, with `bytes` values for `-bin` keys.
- Under the strict profile's empty-by-default metadata policy, both surfaces
  return empty containers on replay unless a key is allowlisted. Playback
  therefore does not reproduce metadata the recording session chose not to
  persist; the mismatch report names the policy, not the missing values.

Playback introduces no artificial latency and does not reproduce recorded
wall-clock timing. It preserves the public messages, sanitized metadata, status,
and details, but claims no HTTP/2 flow-control fidelity. Unary-stream playback
stays pull-based: messages materialize on iteration, not on call construction.

## Async live-call lifecycle

A permitted aio live miss begins at multicallable invocation, matching real
`grpc.aio` behavior.

`__call__` is a synchronous method that returns an awaitable call object,
matching `grpc.aio`. It captures an absolute monotonic deadline, constructs a
shape-specific `DeferredLiveCall`, and schedules its controller task on the loop
captured at channel construction:

- When `asyncio.get_running_loop()` is that loop, `loop.create_task`.
- Otherwise `asyncio.run_coroutine_threadsafe`, which is the only thread-safe
  path and matches gRPC's tolerance of invocation from a non-loop thread.
- If neither is possible, `__call__` raises `grpc.aio.UsageError` rather than
  returning a call that will never start.

The controller task never propagates an exception as its own result. Every
failure is routed into `terminal`, and the task carries a done callback that
consumes any residual exception, so an abandoned call produces no
"Task exception was never retrieved" noise. Errors surface only when the
application awaits, reads, or queries the call — as gRPC does.

Each call follows this state model:

```text
NEW ──► STARTING ──► DELEGATED ──► DONE          (absorbing)
           │                        ▲
           └────────────────────────┘
             startup failure, or deadline
             expiry before delegation

{NEW, STARTING, DELEGATED} ──► CANCELLED         (absorbing)
             local cancel()
```

Edges:

- `NEW → STARTING`: controller task begins transport single-flight.
- `STARTING → DELEGATED`: the real call exists and is published.
- `STARTING → DONE`: transport factory or invocation preparer failed, or the
  deadline expired before delegation. Terminal status is local.
- `DELEGATED → DONE`: the delegated call reached a status, OK or not.
- `{NEW, STARTING, DELEGATED} → CANCELLED`: local `cancel()`. From `DELEGATED`
  the wrapper also cancels the delegated call.
- `DONE` and `CANCELLED` are absorbing; a later `cancel()` returns `False`.

The call owns two separate synchronization points:

- `delegate_ready: Future[RealCall]` resolves once the real call exists.
- `terminal: Future[SafeOutcome]` resolves exactly once after logical RPC
  completion or local cancellation and after opaque-context finalization.

State invariants:

- Publication of the delegated call and observation of the cancel flag occur
  inside one critical section on the controller task. The controller acquires the
  state lock, re-reads the state, and either transitions `STARTING → DELEGATED`
  and publishes, or — if the state is already `CANCELLED` or `DONE` — cancels and
  drains the just-created real call, finalizes its context, and publishes
  nothing. A real call is never created and then forgotten.
- `terminal` is resolved exactly once, by whichever of controller, canceller, or
  deadline timer wins the lock.
- Context finalization runs exactly once, after `terminal` resolves, on every
  path including `STARTING → DONE` and `STARTING → CANCELLED`.

Operation behavior is defined as follows:

- `read`, `__anext__`, `initial_metadata`, and `wait_for_connection` race
  `delegate_ready` against `terminal` and act on whichever resolves first: on
  `delegate_ready` they delegate; on `terminal` they raise the terminal outcome
  (`AioRpcError` for a status, `asyncio.CancelledError` for local cancellation)
  without ever touching a delegate.
- On any terminal transition that skips delegation, `delegate_ready` is resolved
  exceptionally in the same critical section that resolves `terminal`. Neither
  future is ever left pending after a terminal transition, so no operation can
  outlive the call.
- `wait_for_connection()` returns as soon as the call is delegated and the
  delegated call reports connection, and raises the terminal error if the call
  terminated first. It never waits on `terminal` for a healthy call.
- Unary await delegates to the real unary call.
- `code`, `details`, and trailing metadata wait for the logical terminal state.
  `initial_metadata` does not.
- Done callbacks register against `terminal` and receive the wrapper call.
- `done`, `cancelled`, and `time_remaining` are nonmaterializing state queries.

Transport initialization is a session-level single-flight task. Per-call
cancellation never cancels transport initialization required by another call.
The invocation preparer runs once per live call.

Channel close follows `grpc.aio` exactly:

- `close(grace=None)` refuses new invocations, then cancels every active call
  immediately. It does not wait for `terminal`.
- `close(grace=N)` refuses new invocations, waits up to N seconds for active
  calls to reach `terminal`, then cancels whatever remains.
- `__aexit__` is `await self.close(None)`, matching `grpc.aio`.
- Close is idempotent; a second call returns immediately.

Close waits for active calls themselves only under `close(grace=N)`, and only for
that grace period. In every case it then waits for opaque-context finalization of
the calls it cancelled. Finalization is bounded: each finalizer is awaited under
an internal timeout, and a timed-out finalizer is reported through the completion
hook rather than blocking close. Cassette persistence happens after all
finalizers settle.

### Deadlines

The deadline starts at multicallable invocation: `deadline = time.monotonic() +
timeout` when `timeout` is not `None`, otherwise there is no deadline.

- `time_remaining()` returns `max(0.0, deadline - time.monotonic())` in every
  state, and `None` when the call has no deadline. It never materializes
  transport and never awaits.
- The remainder passed to the delegated call is recomputed from
  `time.monotonic()` at the moment of delegation, after transport single-flight
  and invocation preparation have completed.
- If that remainder is not strictly positive, the wrapper terminates locally with
  `DEADLINE_EXCEEDED` and starts no real RPC. A non-positive `timeout` is never
  handed to gRPC.
- A locally synthesized expiry surfaces as `grpc.aio.AioRpcError` /
  `grpc.RpcError` with `StatusCode.DEADLINE_EXCEEDED`, a constant details string,
  and empty metadata, so callers cannot distinguish it from a transport-produced
  expiry by type or code.
- An independent deadline timer resolves `terminal` if it fires while the call is
  still `NEW` or `STARTING`, using the same critical section as cancellation.

## Transport and per-call authentication

Transport creation and per-call authorization are separate extension points:

```python
SyncTransportFactory = Callable[[], grpc.Channel]

AsyncTransportFactory = Callable[
    [], grpc.aio.Channel | Awaitable[grpc.aio.Channel]
]

AsyncLiveInvocationPreparer = Callable[
    [CallDescriptor], Awaitable[LiveInvocation]
]
```

A completion/failure hook receives the opaque context and a `SafeOutcome`. This
lets an application own credential generation, invalidation, and
`UNAUTHENTICATED` retry without obtaining a credential before cassette routing.

On a cassette hit, none of the transport, preparation, or completion hooks are
invoked.

Per-call parameters supplied by the application are handled explicitly:

- `timeout` establishes the absolute monotonic deadline and is not forwarded
  verbatim.
- `metadata` is forwarded to the delegated call, concatenated after the
  preparer's additional metadata, with the application's entries last.
- `credentials` from the application and `LiveInvocation.credentials` are
  combined with `grpc.composite_call_credentials` and the composite is passed to
  the delegated call. gRPC accepts one `CallCredentials` per call, so a silent
  drop of either source is not acceptable.
- `wait_for_ready` and `compression` are forwarded unchanged.
- None of `credentials`, `wait_for_ready`, or `compression` enters the match
  projection, a cassette, a diagnostic, or a log. `credentials` is
  secret-bearing and is never rendered, hashed, or `repr`-ed.
- On a cassette hit these parameters are accepted and ignored; `timeout` still
  governs `time_remaining()` on the replay call.

Strict transport factories must return unintercepted/base channels. Request-
mutating live interceptors are unsupported in strict 0.2. A routing-layer live
request transform may run before serialization when the outgoing request must
change. Projection-only transforms never affect live bytes. Metadata-only live
behavior belongs in the invocation preparer.

## Request and response data separation

Strict mode produces two separate request artifacts.

### Ephemeral live bytes

```text
application request
  → original serializer, called exactly once
  → raw_live_bytes
  → unchanged bytes sent to live transport
```

`raw_live_bytes` never enter matchers, cassette builders, diagnostics, errors, or
library logs. Delegated live calls use identity serialization over these already
encoded bytes. Raw payloads are held only inside a `RawPayload` holder with
`__slots__`, a constant `__repr__`/`__str__`, and no `__eq__`; the library never
binds raw bytes to a bare local in a frame that can raise, and `del`s the holder
before any `raise` in the enclosing scope. Every module that touches raw payloads
sets `__tracebackhide__ = True` at module scope so pytest omits its frames and
their locals from rendered tracebacks.

This narrows but does not close the frame-locals channel. The library makes no
memory-scrubbing claim: `bytes` are immutable and unzeroable in CPython, so raw
payloads remain readable via `gc.get_objects()`, a core dump, a `sys.settrace`
hook, `pytest --pdb`, or a debugger until they are collected. Running a
strict-mode recording under a post-mortem debugger, a locals-rendering crash
reporter, or `--showlocals` is outside the guarantee.

### Safe projection

```text
raw_live_bytes
  → parse as registered request FQN
  → clone
  → trusted application projection transform
  → mandatory final redactor
  → recursive unknown-field discard
  → same-FQN validation
  → canonicalization
  → deterministic protobuf serialization
  → serialize/reparse/reserialize fixpoint check
  → structural validation
  → SafeRequestProjection
```

Sanitization completes before live delegation. Any of the following sends nothing
and persists nothing: missing sanitizer, unknown descriptor, parse failure, wrong
output type, sanitizer exception, FQN change, `RecursionError` or depth-limit
breach, fixpoint-check failure, or validation failure.

The redactor walks with an explicit depth counter bounded by `max_message_depth`
(default 100, matching the protobuf wire parser's own nesting limit) rather than
relying on the interpreter's recursion limit; exceeding it raises
`MessageTooDeepError` before any mutation is applied, and the partially redacted
clone is discarded. Recursive message *types* are permitted; recursive
*instances* cannot exist.

Structural validation operates on the emitted bytes, not the message object:
reparse with the registered descriptor, assert zero unknown fields, assert the
parsed FQN is unchanged, reserialize canonically, and assert byte equality with
the emitted artifact. Anything else is a descriptor-pool disagreement and fails
the interaction.

### Canonical serialization

`SerializeToString(deterministic=True)` guarantees map-entry ordering within one
protobuf build and nothing more; it is not canonical across implementations,
versions, or languages. `protobuf-wire-deterministic` therefore names a form this
library defines, of which the protobuf flag is one input:

| Requirement | Rule |
| --- | --- |
| Map ordering | `deterministic=True`, required, propagated to every nested message. |
| Field ordering | Ascending field number, including extensions. Guaranteed by the runtime; asserted by the fixpoint check. |
| Unknown fields | None present. The discard step must precede serialization; unknown-field order is wire-order and is not deterministic. |
| Float / double NaN | Canonicalized to quiet NaN `0x7ff8000000000000` / `0x7fc00000` before serialization. Non-canonical NaN payload bits are not preserved. |
| Infinities, `-0.0` | Preserved verbatim; both implementations serialize `-0.0` as present. |
| Fixpoint | After serialization, reparse with the registered descriptor, assert zero unknown fields, reserialize, and assert byte equality. A failure aborts the interaction. |

Matching compares the canonical bytes. The canonical bytes are a cache of message
identity, not the definition of it: a cassette whose payload bytes fail the
fixpoint check on the reader's protobuf build is rejected with
`NonCanonicalPayloadError` and a re-record instruction, rather than silently
failing to match. CI runs the byte-equality suite against both `upb` and
`PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python`, and against the minimum and
current protobuf floors.

### Responses

Responses use the same separation. The wrapper observes raw response bytes,
parses the registered response FQN, calls the original deserializer once for the
application value, and sanitizes a separate parsed clone immediately. The
application receives the original value; only a `PersistableEvent` reaches the
cassette builder.

Strict mode requires a `MethodDescriptorRegistry`. The `legacy-raw` profile may
retain arbitrary serializer output, but it makes no credential-safety claim.

## Security boundary

The strict threat model covers secrets in:

- Request and response protobufs.
- Nested, repeated, map, oneof, extension, and `Any` fields.
- Unknown protobuf fields.
- Request, initial, and trailing metadata.
- Binary metadata and rich-status trailers.
- Status details.
- Targets.
- Library-generated exceptions, mismatch diagnostics, and logs.
- Cassette, temporary, quarantine, and migration files.

Application-provided transforms, invocation preparers, and hooks are trusted code
outside the exfiltration guarantee. The library cannot prevent such a callback
from logging or transmitting its own input. Strict callback failures persist
nothing, expose a constant-message exception without a raw chained cause, and
cause the library to log no callback input.

Also outside the boundary, stated explicitly rather than by omission:

- **The live transport.** Unredacted request bytes and real credentials reach the
  application-chosen target under application-chosen channel credentials. Strict
  mode governs what is *persisted*, not what is *sent*.
- **Process memory.** No scrubbing is performed or possible; see the raw-payload
  note above.
- **The host test runner.** Frame-locals rendering (`pytest -l`, `--pdb`), crash
  reporters, `faulthandler`, and core dumps can surface raw payloads from library
  frames.
- **Structural metadata.** Even a fully redacted cassette records method, shape,
  event count, event ordering, payload byte lengths, metadata key presence, and
  status codes. These are not redacted and can identify a request. Callers for
  whom message counts or payload sizes are themselves sensitive should not commit
  cassettes to a shared repository.

### Redaction policy

The strict redactor is deny-by-default. A field is persisted only if a policy
rule names it; every other field is cleared before serialization. Policy rules
are expressed as fully qualified field paths against the registered FQN.

| Policy verb | Effect |
| --- | --- |
| `keep` | Persist the field value verbatim. |
| `keep_shape` | Persist a fixed placeholder of the field's type; the original value never reaches the cassette. |
| `drop` (default) | `ClearField` the field. |

| Construct | Deny-by-default resolution |
| --- | --- |
| Message field | Recurse; a message with no `keep` descendant is cleared entirely. |
| Real oneof | `ClearField(oneof_name)` unless the *selected* member is `keep`. Never redact by assigning a sibling member. |
| proto2 presence / proto3 `optional` | `ClearField(field_name)`. Assigning the default value is forbidden: proto2 and synthetic-oneof presence survive it. |
| Extension | Denied unless the rule names the full extension name. Extensions whose descriptor is absent from the pool arrive as unknown fields and are removed by the discard step. |
| Map | The whole map is cleared unless a rule names it. `keep` on a map keeps keys and values; there is no per-entry policy, because entry keys are data. |
| `Struct` / `Value` / `ListValue` | `keep` or `drop` as a unit only. Per-key policy is not offered: `Struct` keys are runtime data, not schema. |
| Unknown fields | Always removed. |

Policy resolution is closed over the descriptor: a rule naming a path absent from
the registered descriptor is a configuration error, not a no-op.

The redactor must handle proto2 presence, proto3 optional fields, oneofs,
extensions, nested/repeated messages, maps, wrappers, `Any`, `Struct`, `Value`,
`Timestamp`, `Duration`, `FieldMask`, `Empty`, `NullValue`, enums, bytes, numeric
edge cases, NaN/infinity, and unknown fields. `FieldMask` in particular persists
schema paths that a `keep` on the containing message would silently expose.

Replacement values are constant-shape, not length-preserving: a redacted `string`
becomes a fixed sentinel, `bytes` a fixed-length constant, numerics their type's
zero. Length-preserving redaction is forbidden — the persisted length and the
resulting deterministic-serialization length are themselves a disclosure, and
both reach the match key.

Map keys are never individually redacted; a map that is not explicitly `keep` is
cleared whole. Where `keep_shape` is applied to a map, two entries collapsing to
the same replacement key fails the interaction closed rather than dropping an
entry.

### `Any` handling

`Any.value` is opaque bytes; no traversal, discard, or field policy reaches
inside it. An `Any` carrying a message with an unknown field containing a bearer
token would otherwise pass unknown-field discard, pass same-FQN validation, and
land in the cassette verbatim. The redactor resolves `Any` explicitly:

1. Strip `type_url` to its FQN and look it up in the configured
   `MethodDescriptorRegistry` pool.
2. **Type absent from the pool:** clear `value` and retain `type_url`. Under a
   `keep` rule that names the `Any` field, the interaction instead fails closed
   with `UnresolvableAnyPayloadError`; opting into lossy `type_url`-only
   persistence requires `allow_opaque_any: true` on the field rule.
3. **Type present:** unpack into a message of that type, run the full pipeline
   recursively (the application transform is *not* re-run; redactor, discard, and
   validation are), repack, and require the repacked `type_url` to be
   byte-identical to the original.
4. Unpacking is depth-limited by `max_message_depth`. A nested `Any` counts
   against the same budget.
5. The same rules apply to any `bytes` field declared as carrying a serialized
   message; the library makes no attempt to detect such fields and persists them
   only under an explicit rule.

Policy paths for `Any` contents are rooted at the **packed** type, not at the
enclosing message: a rule is written
`google.protobuf.Any@package.Packed/field.subfield` and applies wherever a
payload of `package.Packed` is unpacked, at any nesting depth. Rules rooted at
the enclosing request FQN stop at the `Any` field itself and cannot reach inside
it.

An `Any` field with no packed-type rules resolves to step 2 (`value` cleared,
`type_url` retained) without unpacking. Step 3 runs only when at least one rule
is rooted at the resolved packed type.

Resolution uses the registry's pool. A registry constructed without an explicit
pool uses the default pool, which contains every proto module the process has
imported — narrow it deliberately if the set of resolvable packed types is part
of your threat model.

### Metadata policy

- Records nothing by default.
- Requires an explicit key allowlist and a value transformer. Allowlist keys are
  ASCII-lowercased at configuration load; runtime keys are ASCII-lowercased
  before comparison. A non-lowercasable or syntactically invalid key is a
  configuration error.
- Enforces the gRPC key grammar `^[0-9a-z_.\-]+$` on both write and read; a
  cassette carrying an invalid key is rejected at load, not at replay.
- Derives text/binary from the `-bin` suffix, never from the Python type of the
  value. `-bin` values are `bytes`, are passed to a separate binary transformer,
  and are persisted base64-encoded with an explicit `binary` tag. Text values
  must be ASCII `0x20`–`0x7e` after transformation; anything else fails closed.
- Denies the reserved set unconditionally, ahead of the allowlist and with no
  override: any `grpc-` prefixed key, any `:` prefixed pseudo-header,
  `content-type`, `te`, and `user-agent`. These keys are reserved by the gRPC
  wire protocol and are synthesized by the transport below the client API:
  `grpc-timeout` derives from the deadline, `grpc-encoding` from the
  `compression` parameter, `user-agent` and `content-type` from the channel.
  They do not appear in application request metadata, and gRPC rejects an
  application attempt to set them rather than honoring it. The deny-list exists
  so that a hand-edited or migrated cassette cannot smuggle one back in.
- Stores entries as an ordered tagged list preserving duplicate keys and their
  interleaving. The transformer receives `(key, occurrence_index, value)` so a
  secret split across duplicate entries is still seen per occurrence.
- Rejects matching on an excluded key. `MetadataMatcher` against an excluded key
  is a configuration error.
- Omits target persistence unless explicitly enabled. `TargetMatcher` configured
  against a session with `target_persistence: false` is a configuration error,
  not a silent miss.
- Status details are dropped by default. `terminal_status.details` is persisted
  as `null` unless a policy rule sets `status_details: keep`, which is an
  explicit acknowledgement that the server's free-text detail string may echo
  request content. There is no partial or truncating mode: truncation still
  discloses a prefix.
- `grpc-status-details-bin` is never persisted under any setting. It carries a
  `google.rpc.Status` whose `details` is a repeated `Any`, which the field policy
  cannot reach through a metadata value. Replay reconstructs status from
  `terminal_status` only, so a replayed rich-status trailer is absent rather than
  reconstructed; applications depending on rich status must record under
  `legacy-raw`.

### Profile identity

The sanitizer profile ID, semantic version, and non-secret configuration hash are
stored in the cassette. Playback rejects an incompatible configured projection.

`config_sha256` is the lowercase hex SHA-256 of the UTF-8 **canonical profile
document**: JSON with sorted keys, no insignificant whitespace
(`separators=(",", ":")`), `ensure_ascii=false`, and no trailing newline. The
document contains exactly:

| Key | Contents |
| --- | --- |
| `profile_id`, `profile_version` | As in the envelope. |
| `field_policy` | Sorted list of `[field_path, verb]` pairs, including implicit `drop` only where a rule was written. |
| `metadata_allowlist` | Sorted lowercased keys, with their text/binary tag. |
| `target_persistence` | Boolean. |

`config_sha256` covers only inputs that shaped the persisted bytes. Limits and
matcher selection are reader-side policy: they change what a reader will *accept*
or *select*, never what a writer *produced*, and are therefore excluded. A reader
may tighten or relax its limits without invalidating existing cassettes.

Secrets, file paths, environment values, and callables are excluded.
`grpcvcr profile-hash` prints both the canonical document and its digest so a
user can diff a rejection.

The hash covers declarative configuration only. It cannot cover the
application-provided projection transform or invocation preparer, which are
opaque callables: editing a transform's body leaves `config_sha256` unchanged and
a cassette recorded under the old behavior will still load. Detecting that is the
application's responsibility — bump `profile_version` when a transform's
semantics change. On mismatch, strict playback fails with `ProfileMismatchError`
reporting both digests; `legacy-raw` warns.

## Cassette formats and compatibility

### Schema v1

- `0.1.2` is the final unrestricted v1 writer.
- In 0.2, v1 is playback-only.
- Opening v1 with a record-capable mode raises a migrate/re-record error.
- V1 is never silently rewritten on close or on a `NEW_EPISODES` miss.
- Cassette-specified Python module paths are never imported.
- Migration defaults to dropping legacy metadata.
- Unary and unary-stream migration requires method descriptors and passes through
  the complete sanitizer and validator.
- Client-stream and bidi request boundaries cannot be recovered; migration
  classifies them as `re-record required`.

Migration is an explicit, separately invoked operation that writes a new v2 file;
it is never triggered by opening a cassette. It provides dry-run reporting with
`lossless`, `degraded`, and `re-record required` classifications, and refuses
unsupported future schemas.

### Schema v2

Schema v2 supports unary-unary and unary-stream, and is YAML-only. A
representative envelope is:

```yaml
format: grpcvcr-cassette
schema_version: 2
required_features: []      # reader MUST reject any element it does not implement
optional_features: []      # reader MAY ignore unknown elements
recording_profile:
  id: protobuf-safe
  version: 1
  config_sha256: "..."
interactions:
  - ordinal: 0
    method: /package.Service/Method
    shape: unary_stream
    request_fqn: package.Request
    response_fqn: package.Response
    events:
      - kind: client_message
        client_ordinal: 0
        payload:
          encoding: protobuf-wire-deterministic
          protobuf_type: package.Request
          protobuf_b64: "..."
      - kind: server_initial_metadata
        entries: []
      - kind: server_message
        server_ordinal: 0
        payload:
          encoding: protobuf-wire-deterministic
          protobuf_type: package.Response
          protobuf_b64: "..."
      - kind: server_trailing_metadata
        entries: []
      - kind: terminal_status
        code: OK
        details: null
```

Feature names are lowercase dotted identifiers in a registry owned by this
project (`streaming.client_ordinals`, `payload.any_recursive`, …). Both feature
lists are mandatory keys; absent or non-list values are a load error, so a v2
reader can never mistake a malformed envelope for a featureless one. A reader
rejects the cassette if any `required_features` element is outside its supported
set, naming the unsupported element and the minimum `grpcvcr` version that
implements it. `optional_features` conveys writer intent only and never changes
replay semantics.

Envelope keys are validated *after* the feature lists are read, in this order:
(1) `format`, `schema_version`, `required_features`, and `optional_features` must
be present and well-typed; (2) every `required_features` element must be in the
reader's supported set, or the cassette is rejected naming the element and the
minimum `grpcvcr` version implementing it; (3) any remaining top-level key must
be either a base-schema key or a key introduced by a feature the reader supports,
and is otherwise rejected. A feature's registry entry names the envelope keys it
introduces, so a reader that supports a feature accepts exactly the keys that
feature adds and no others. Adding a top-level key without a feature gate is a
schema-major change.

The validator enforces shape-specific cardinality and ordering, exactly one
terminal result, no events after termination, known status codes, canonical
base64, valid FQNs, size limits, profile compatibility, and rejection of unknown
event kinds and required features.

### Load-time hardening

Envelope parsing precedes validation and is itself adversarial input.

- Reject the file on `stat` size above `max_cassette_bytes` before reading it.
- Parse with a `yaml.SafeLoader` subclass that (a) raises on any anchor or alias
  node, and (b) raises on duplicate mapping keys. PyYAML's stock `SafeLoader`
  expands aliases and lets a duplicate key silently override an earlier one,
  which would let a shadowing `payload` key evade validation.
- Canonical base64 means: alphabet `A-Za-z0-9+/` with correct `=` padding, no
  whitespace, no line breaks, and zeroed padding bits. Enforce by decoding and
  re-encoding and requiring byte equality with the stored text —
  `binascii.a2b_base64(strict_mode=True)` is unavailable on the Python 3.10 floor
  and must not be the only check.

| Limit | Default | Enforced at |
| --- | ---: | --- |
| `max_cassette_bytes` | 32 MiB | Before read |
| `max_interactions` | 1 000 | Parse |
| `max_events_per_interaction` | 10 000 | Parse |
| `max_payload_bytes` | 4 MiB | Before base64 decode, from encoded length |
| `max_metadata_entries` | 64 | Parse |
| `max_metadata_value_bytes` | 8 KiB | Parse |
| `max_message_depth` | 100 | Redaction and reparse |

Limits come from the reader's configuration, never from the cassette, and are not
part of `config_sha256`. A cassette cannot raise the limits that gate it —
`max_cassette_bytes` in particular is enforced by `stat` before the file is
opened, so a value stored inside the file could never be consulted in time.

### Profiles

Two v2 profiles are defined:

- `legacy-raw` preserves arbitrary codec bytes and legacy matcher and replay
  behavior for supported shapes.
- `protobuf-safe` exposes immutable views, ordered consumption, secure metadata
  defaults, and policy-only persistence.

Only `legacy-raw` exposes mutable compatibility views or a direct raw
`record_interaction` path.

### Schema v3

Client-streaming and bidi use a new schema major after executable prototypes
cover ping-pong, server-first, concurrent reader/writer, early failure,
half-close, cancellation, and abandonment.

The provisional v3 event model contains independent client/server ordinals,
client half-close, initial/trailing metadata, terminal status, and a causal
client-message watermark for each server event. An older reader must reject
unknown required features rather than ignore them.

## Record modes and streaming routing

Mode behavior is defined exactly for the shapes 0.2 supports. Unsupported shapes
raise `UnsupportedRpcShapeError` in every mode, before mode logic runs.

| Mode and session condition | Existing match | Missing match |
| --- | --- | --- |
| `NONE` | Replay | Error; never live |
| `NEW_EPISODES` | Replay | Lazily use live transport and record |
| `ALL` | Always live | Always live |
| `ONCE`, cassette absent at session entry | Record | Record |
| `ONCE`, cassette existed at session entry | Replay | Error; never live |

Under v2, `ALL` starts an empty generation and records every invocation,
including repeated identical requests, replacing the file atomically only after
successful session completion. This changes 0.1.x behavior, where the existing
file is loaded regardless of mode (`src/grpcvcr/cassette.py:73-74`),
same-request interactions are replaced in place (`:117-120`), and the write is a
non-atomic `write_text` (`src/grpcvcr/serialization.py:473`) issued on close
regardless of outcome.

`ONCE` likewise changes: `Cassette.can_record` returns `True` for `ONCE`
unconditionally today (`src/grpcvcr/cassette.py:96-97`), so an existing cassette
does not currently force replay-only.

For future request-streaming support:

- `NONE` reserves a replay interaction and validates messages incrementally.
- `ALL` always uses genuine live streaming.
- `ONCE` selects an all-live or all-replay session at entry.
- `NEW_EPISODES` requires a pre-stream selector or interaction key capable of
  choosing before the request iterator is consumed.
- No default mode drains an iterator, starts live traffic on a hit, or falls back
  to live after replay output has been exposed.

## Matching and consumption

Strict interactions follow this runtime lifecycle:

```text
unused → reserved → consumed
```

- Reservation is atomic at invocation.
- Starting a replay consumes the interaction even if the caller later abandons
  it.
- Global strict-sequence playback is the `protobuf-safe` default.
- Generic users may explicitly choose first-unused matching.
- Under strict-sequence playback the ordinal selects the interaction and the key
  verifies it; a key mismatch is an error, never a search for a better candidate.
  Under first-unused matching the same key selects.
- Response FQN verifies the interaction in both regimes and is never a selection
  key.
- FQNs are optional under `legacy-raw` and mandatory in strict mode.
- Strict diagnostics contain only method, shape, safe FQNs, candidate ordinal,
  consumption state, and safe digests.
- Recording ordinal is assigned at invocation rather than completion.

The strict key comprises method, shape, request FQN, deterministic safe request
projection, and explicitly selected safe metadata. Target matching is opt-in.

Deny-by-default redaction makes projection collisions the normal case, not an
edge case: once identifying fields are dropped or replaced with constants,
distinct requests to the same method commonly project to identical canonical
bytes. The key therefore does not uniquely identify an interaction and is not
treated as if it did.

- Global strict-sequence playback is unaffected: ordinal selects, and the
  selector is a consistency assertion. This is why it is the `protobuf-safe`
  default.
- First-unused matching is refused on a cassette containing two interactions with
  an identical `(method, shape, request_fqn, projection, selected metadata)`
  tuple. The refusal names the colliding ordinals and points at strict-sequence
  playback or at a `keep` rule that would restore discrimination. Silently
  consuming the first of an ambiguous pair would return a response recorded for a
  different request.
- The writer detects collisions at record time and records the collision set in
  the mismatch diagnostic surface, so the ambiguity is reported when the cassette
  is written rather than on a later replay.

Caller-supplied cassette and lock paths may appear in strict diagnostics and
exception messages; they are caller-provided, not derived from recorded data. No
other filesystem path, including temporary-file paths, appears in any strict
exception.

## Cancellation and incomplete interactions

Schema v2 treats local client cancellation and abandonment as runtime-local
behavior:

- Locally cancelled or incomplete recordings are discarded.
- A remote terminal `StatusCode.CANCELLED` is recorded as an ordinary non-OK
  server result and remains replayable.
- No cassette-derived local cancellation state is claimed until schema v3.

Cancellation semantics match gRPC exactly, and the two stacks differ:

- `cancel()` returns `True` when the call had not yet reached a terminal state
  and `False` otherwise, in both stacks. A second `cancel()` returns `False` and
  changes nothing.
- After local cancellation: `cancelled()` is `True`, `done()` is `True`, `code()`
  is `StatusCode.CANCELLED`, `details()` is gRPC's local-cancellation string.
- Aio: a locally cancelled call raises `asyncio.CancelledError` from `await`,
  `read()`, and `__anext__` — never `AioRpcError`. This is the single most
  commonly mis-emulated aio behavior and has a dedicated conformance test.
- Sync: a locally cancelled unary future raises `grpc.FutureCancelledError` from
  `result()`, while iteration over a cancelled stream raises the `grpc.RpcError`
  carrying `CANCELLED`.
- A `ReplayCall` cancelled mid-stream still counts the interaction as consumed;
  the reservation is not returned to the pool.

Sync `close()` takes no grace parameter, aborts in-flight calls, discards
incomplete builders, and then saves completed interactions. Aio close is
described under [Async live-call lifecycle](#async-live-call-lifecycle);
cassette persistence happens after all finalizers settle.

## Storage transaction

Writable sessions use:

- Invocation-order interaction numbering.
- Atomic find/reserve/consume operations.
- Temporary files created in the **destination directory** with
  `O_CREAT|O_EXCL|O_NOFOLLOW` and mode `0600`, never in `$TMPDIR`: `os.replace`
  is atomic only within one filesystem.
- Rejection of a destination path that is a symlink, resolved under the acquired
  lock. `os.replace` overwrites the link itself, silently detaching a shared
  cassette.
- Explicit final mode: the committed file is `chmod`ed to the mode of the file it
  replaces, or to `0600` when creating. umask is deliberately not applied: the
  mode is already owner-only, so masking it can only produce an unreadable file.
  The temp file's `0600` is not inherited by the commit, so a checked-in `0644`
  fixture is not silently tightened.
- An exclusive sidecar writer lock held with `fcntl.flock` (POSIX) or
  `msvcrt.locking` (Windows) on an open descriptor, never an `O_EXCL` sentinel
  file: an advisory lock is released by the kernel on process death and needs no
  stale-lock heuristic.
- Source-hash re-read, comparison, serialization, validation of the serialized
  temporary artifact, flush, `fsync`, and `os.replace` all performed **inside**
  the held lock. Reading the hash outside the lock reintroduces the write-write
  race the lock exists to prevent.
- Directory `fsync` where supported.
- Temporary-file cleanup on every failure.
- Preservation of dirty state after a failed save.

Concurrent writers are rejected rather than merged. Readers observe either the
old complete cassette or the new complete cassette, never an intermediate file.

Platform caveats, stated rather than assumed: directory `fsync` is a no-op on
Windows, so a crash between `os.replace` and the next flush may lose the rename
while leaving both files intact — never a torn file. `os.replace` on Windows
raises `PermissionError` if a reader holds the destination open; the writer
retries with backoff and then fails, leaving the prior cassette intact. Advisory
locking over NFS depends on a working lock daemon and is not supported for
concurrent writers; single-writer use on NFS is fine.

## Implementation plan

One pull request per phase for phases 1–5. Each merges onto a green main and does
not depend on an unmerged successor. Phase 6 is a series, not a pull request, and
is sequenced after `0.2` ships. This is the only ordering in this document; there
is no separate PR list. Where a phase names an internal split, that split is a
review convenience and does not change the ordering.

| Phase | Title | Estimated effort | Approximate diff |
| --- | --- | --- | ---: |
| 1 | Public API baseline and release hardening | 5–8 engineer-days | +900 / −60 |
| 2 | Cassette v2, safe projection, and the storage transaction | 10–15 engineer-days | +4500 / −700 |
| 3 | Async routing engine and live-call lifecycle | 10–15 engineer-days | +3000 / −550 |
| 4 | Sync parity, compatibility facades, and the pytest plugin | 10–15 engineer-days | +2000 / −900 |
| 5 | Migration tooling, dependency matrix, typing, and documentation | 5–12 engineer-days | +1200 / −450 |
| 6 (series) | Schema v3 and request-streaming | 25–35 engineer-days | separate series |

The downstream pilot is not a `grpcvcr` pull request. It lands in the consumer's
repository and is tracked by the
[pilot's adoption gate](#appendix-downstream-pilot-notebooklm).

Each pull request must be independently reviewable. Regression tests land with
their fix or as explicitly tracked strict `xfail` contracts; the main branch is
never left with unexplained failing tests.

### 1. Public API baseline and release hardening

Contains the executable public-surface snapshot for every root export, `Cassette`
method, matcher, exception attribute, `grpcvcr.interceptors` export, pytest
fixture, CLI option, and marker form; the public `Channel`/`MultiCallable`/`Call`
conformance spike; a separate bidi routing and causal-event spike without
freezing its schema; hardened publishing so the built wheel is installed and
tested before the same artifact is published; `pytester` entry-point discovery
through that wheel; and correction of the false documentation claims listed under
[Documentation defects](#documentation-defects-in-the-current-release).

Put the upstream Python 3.10 support question to the maintainer here; record the
answer in Decision gates and risks before pull request 5 sets the CI matrix. Keep
replacement code compatible with Python 3.10 regardless.

Coherent because nothing here changes library behavior. It is a test, CI, and
documentation pull request that establishes the contract every later pull request
is measured against, and it is reviewable by reading the golden files.

Exit criteria:

- Routing feasibility is demonstrated for both stacks, including the aio
  interception chain: the spike reproduces `grpc.aio` interceptor dispatch for
  the supported shapes.
- The differential-test harness exists and runs against a real `grpc.aio`
  channel. The implementation it will be pointed at ships in pull request 3,
  which is where the comparison becomes an exit criterion.
- Public compatibility obligations are documented, name by name.
- The published documentation contains no claim that is false at `0.1.2`.

`0.1.2` is tagged after this pull request merges, not before: `release.yml`
publishes an artifact it never installs or tests today, so tagging earlier
publishes through the unhardened path.

### 2. Cassette v2, safe projection, and the storage transaction

Contains the v2 models, strict validator, profile metadata, and safe internal
constructors; the encode-once projection pipeline, method descriptor registry,
mandatory redactor, and metadata allowlist; the threat-model tests and
constant-message callback failures; atomic storage, exclusive writer locking,
source-hash conflict detection, and fault injection; cassette sessions, atomic
reservation, ordered consumption, and structured safe mismatch reports; the
frozen playback-only v1 adapter and the migration classifier.

Coherent because it is the entire persistence and security layer and touches no
gRPC call object. It can be reviewed and tested without a channel.

This is the one uncomfortably large merge in the sequence: roughly 1.5 times the
current library, landing at the existing `fail_under = 100` coverage gate in a
single review. Split it only at this seam if review capacity requires:

- 2a: v2 models, validator, and the storage transaction. Pure data and I/O.
- 2b: projection pipeline, redactor, metadata policy, session reservation and
  matcher, v1 playback adapter, and the migration classifier.

2b depends on 2a and on nothing else. No other split leaves both halves green.

Exit criteria:

- Raw data has no path into a strict persistable object.
- V2 round-trips and rejects invalid state machines.
- V1 never auto-upgrades.
- Storage fault injection leaves the prior cassette intact.

### 3. Async routing engine and live-call lifecycle

Contains `AsyncRoutingChannel`, async multicallables and the aio interception
chain, replay unary and lazy unary-stream calls with `read()`/`EOF`/cached
`__aiter__` semantics, `DeferredLiveCall` and its state model, transport
single-flight initialization, the per-call invocation preparer, absolute
deadlines, cancellation and startup races, terminal finalization, and
`UnsupportedRpcShapeError` for `stream_unary` and `stream_stream`.

Coherent because it is one state machine. Splitting replay from deferred live
calls, or unary from unary-stream, merges a channel that cannot complete a live
miss, so the intermediate states are not independently useful.

Exit criteria:

- First-message latency proves streams are not eagerly drained.
- Every deferred-call state and cancellation transition is covered, including the
  cancel/delegate race.
- Concurrent credential refresh and `UNAUTHENTICATED` retry behavior remains
  correct.
- The phase 1 differential harness, re-pointed at `AsyncRoutingChannel`, passes:
  the same interceptor stack run against a real `grpc.aio` channel and against
  `AsyncRoutingChannel` produces identical observable behavior.

The pilot-ready async milestone completes after this phase.

### 4. Sync parity, compatibility facades, and the pytest plugin

Contains sync routing, `with_call`, `future`, and lazy response iterators; the
`RecordingChannel`/`AsyncRecordingChannel` facades preserving `.channel`,
`.cassette`, `.target`, and existing context-manager behavior
(`RecordingChannel.__enter__`/`__exit__`, `AsyncRecordingChannel.__aenter__`/
`__aexit__`, `Cassette.__enter__`/`__exit__`), whose semantics change because
sync `close()` now aborts in-flight calls and aio `__aexit__` is
`await close(None)`; the `async_recorded_channel` deprecation and its
proper async replacement; pytest plugin hardening; removal of the plugin from
`[tool.coverage.run] omit` **and** removal of `-p no:grpcvcr` from the CI and
`Makefile` test invocations, without which the plugin is still never imported in
the run that measures coverage; and resolution of the unusable `grpcvcr_channel`
fixture. That fixture is undocumented in `docs/guides/pytest.md`, so deleting it
is low-risk and preferred over making it functional.

Coherent because it is the whole backwards-compatibility surface, verified
against the golden snapshot from pull request 1. The plugin belongs here because
its fixtures construct exactly the facades this pull request restores.

Exit criteria:

- Sync and aio behavior match the public conformance matrix.
- Every name in the pull request 1 snapshot is resolved as facade, deprecation,
  or documented removal.

### 5. Migration tooling, dependency matrix, typing, and documentation

Contains cassette `validate`, migration dry-run and reporting; minimum/current
`grpcio` and protobuf compatibility jobs; extension of the CI matrix to Python
3.11–3.14 with matching classifiers (the matrix is 3.11–3.13 today), testing
Python 3.10 in the downstream fork and advertising it upstream only if accepted
by the maintainer; tightened Pyright on replacement modules and
`pyright --verifytypes` after the public API stabilizes; removal of the unused
`[tool.mypy]` block and the `mypy`, `types-protobuf`, and `types-pyyaml` dev
dependencies, since only pyright runs in pre-commit and CI; a decision on
`CHANGELOG.md` versus GitHub releases as the release-note home; documentation
rewritten to the shapes actually supported; executable documentation examples
restored, or the dead `test-examples` Makefile target and the two
`--ignore=tests/test_examples.py` flags removed, since `tests/test_examples.py`
does not exist; `validation: anchors: warn` added to `mkdocs.yml`, because
`mkdocs build --strict` does not check intra-page fragments and this document now
carries several; and installation and testing of wheels and source distributions
before release.

Coherent because it is the release gate: configuration, tooling, and prose over a
frozen public API.

Exit criteria:

- A user can identify whether a v1 cassette is losslessly migratable, degraded,
  or must be re-recorded.

### 6. Schema v3 and request-streaming

Finalize the causal schema after the bidi prototypes pass; implement the
pre-stream routing selector contract; implement async iterator and explicit
`write`/`done_writing`/`read` modes; reject mixed iterator and read/write use as
gRPC does; implement sync client-streaming and bidi request/response iterator
wrappers; cover server-first, ping-pong, half-close, early failure, concurrent
reader/writer, cancellation, and abandoned calls. Published as `0.3`.

Not a single pull request. Sequenced after `0.2` ships, beginning with the causal
schema frozen against the prototypes from pull request 1.

## Test strategy

### No-live-side-effects tests

Replay tests patch the following to fail:

- Sync and aio channel constructors.
- Transport factories.
- Per-live-call invocation preparers.
- Credential providers.
- `socket.connect` and `getaddrinfo`.

Concurrent replay calls must pass without touching any of them.

### Security tests

Place canary secrets in every supported scalar and composite protobuf location,
including nested messages, repeated fields, map keys and values, oneofs,
extensions, `Any` (both resolvable and absent from the pool), unknown fields,
unknown fields nested inside an `Any` payload, and each request/response frame.

Also place secrets in request/initial/trailing metadata, binary metadata, status
details, rich status trailers, targets, and callback exception text.

Assert absence from:

- Cassette and temporary bytes.
- Migration output.
- Library exceptions and `repr` output.
- Mismatch diagnostics.
- Library logs.
- `pytest --showlocals` and `--pdb` traceback rendering.
- A `gc.get_objects()` sweep after the call completes, for the narrowed
  frame-locals claim only.

Decode persisted protobufs with descriptors and verify their structure; regex
scanning base64 text alone is insufficient. Verify that application messages are
never mutated. Verify deterministic, idempotent output across supported Python
and protobuf versions, and across `upb` and the pure-Python implementation.

### Schema and storage tests

- Unknown keys, events, and required features.
- YAML anchors, aliases, and duplicate mapping keys.
- Invalid and noncanonical base64.
- Invalid metadata typing, `-bin` handling, duplicate keys, and case folding.
- Illegal event ordering and cardinality.
- Missing or multiple terminal events.
- Oversized files and messages against each declared limit.
- Disk-full, permission, serializer, validator, rename, and directory-sync
  failures.
- Cross-filesystem `os.replace`, symlinked destinations, and committed file mode.
- Concurrent reservations, out-of-order completions, channel close with active
  calls, and source-hash conflicts.
- Readers observing only complete old or new files.

### Call-lifecycle tests

- Await, read, iteration, metadata, status, details, callbacks, and close.
- Every deferred-call state and cancellation transition, including
  `STARTING → DONE` and the cancel/delegate race.
- Deadline expiration before, during, and after transport materialization.
- Multiple operations racing on one call without duplicate startup.
- Concurrent calls sharing transport but not invocation preparation.
- Partial server streams followed by non-OK status.
- Local cancellation versus remote `StatusCode.CANCELLED`, asserting
  `asyncio.CancelledError` on aio and `grpc.FutureCancelledError` on sync.
- A call constructed and abandoned without being consumed, asserting no
  "Task exception was never retrieved" warning.
- `close(None)` and `close(grace)` with active calls.

### Compatibility and migration tests

- Existing root exports, signatures, attributes, exceptions, matchers, fixtures,
  and markers, against the pull request 1 golden snapshot.
- V1 unary playback without importing stored Python paths.
- No implicit v1 writing or conversion.
- Metadata dropped by default during migration.
- Refusal to migrate unresolved types or unframed request streams.
- Installed pytest entry-point discovery through a built wheel.

## Quality gates

This is the single normative gate list. Phase exit criteria assert only what is
specific to that phase.

Before the downstream pilot:

- Replay works with all transport, auth, DNS, and socket paths disabled.
- Strict sanitizer canaries pass across all required protobuf and metadata
  surfaces, including `Any` payloads and frame-locals rendering.
- Original live request and response objects remain unmodified.
- Matching derives from actual emitted request bytes, including custom serializer
  mutation tests.
- Canonical serialization is byte-stable across `upb`, pure-Python, and the
  protobuf floor.
- Repeated interactions consume in deterministic order.
- Unary-stream recording and replay remain pull-based and lazy.
- Deadline, callback, cancellation, and close races pass.

Before `0.2`:

- Full public conformance matrix for supported shapes passes.
- Unsupported shapes fail before transport or authentication.
- Every public production module, including the pytest plugin, is measured by
  coverage, with the plugin actually imported in the measuring run.
- Targeted mutation testing covers sanitizer, validator, storage transaction,
  record modes, and reservation logic.
- Strict typing passes on replacement modules.
- Minimum/current dependency combinations pass.
- The built distribution is installed and tested before publication.
- Documentation makes no unsupported all-shape or credential-safety claims, and
  every breaking change in
  [Breaking changes in 0.2](#breaking-changes-in-02) appears in the release
  notes.

Coverage and aggregate mutation percentages are supporting metrics, not proof of
security. The state-machine, persistable-constructor, and threat-model tests are
release gates in their own right.

## Appendix — downstream pilot: NotebookLM

This section is the pilot consumer's integration plan. Nothing in it constrains
the library design above; an upstream maintainer can skip it.

NotebookLM is the first consumer of the v2 async engine. It requires Python 3.10
and tests that in its own fork. It must move `BearerProvider.get()` out of the
channel factory and into the per-live-call invocation preparer, since no
transport, preparation, or completion hook runs on a cassette hit.

Its adoption gate — it retains its custom cassette seam as an oracle and fallback
until all of the following hold:

- The new engine covers more than the single recorded `GetProject` case.
- At least one real unary-stream interaction is recorded and replayed.
- Playback demonstrably constructs no channel and mints no bearer.
- Adversarial credential-leak tests pass.
- Web and Android public suites remain behaviorally equivalent.
- Android live E2E runs as a separate CI/nightly leg.
- Several Android nightly cycles complete without cassette or authentication
  regressions.

Only then should the custom seam be removed.

## Estimates and staffing

| Milestone | Estimated effort |
| --- | ---: |
| Async unary/unary-stream, pilot-ready (PRs 1–3) | 25–38 engineer-days |
| Upstream-quality `0.2` with sync parity (PRs 1–5) | 40–65 engineer-days |
| Full all-shape program through `0.3` (PRs 1–6) | 65–100 engineer-days |

The work benefits from one core runtime architect and one security/testing
engineer, with a downstream integration owner contributing integration cases.
Parallel staffing reduces test and integration latency but does not divide the
schedule linearly because the schema, call lifecycle, and security projection are
tightly coupled.

## Decision gates and risks

### Python 3.10

The downstream pilot requires Python 3.10, while upstream currently requires 3.11
and Python 3.10 reaches end of life in October 2026. Replacement code remains
3.10-compatible and the pilot's CI tests it. The upstream maintainer must decide
whether `0.2` advertises temporary Python 3.10 support or the consumer carries
that packaging change downstream.

Advertising it upstream touches six sites that must move together:
`requires-python`, the classifier block, `[tool.ruff] target-version`,
`[tool.pyright] pythonVersion`, `[tool.mypy] python_version` (if the block
survives pull request 5), and the CI matrix.

### CI record-mode detection

`docs/guides/ci-testing.md` and `CHANGELOG.md` claim automatic `RecordMode.NONE`
in CI. No such code exists, so CI currently defaults to recording. Pull request 1
removes the claim. Whether `0.2` implements the detection is an open decision;
implementing it is a behavior change for anyone whose CI currently records.

### Publishing this plan to the public documentation site

`mkdocs.yml` lists this page under `Development`, and `.readthedocs.yaml` builds
from that config with `fail_on_warning: true`, so this page publishes to
`grpcvcr.readthedocs.io`. The body is written in library terms, but
[the appendix](#appendix-downstream-pilot-notebooklm) names `BearerProvider.get()`,
the `GetProject` case, the Web and Android public suites, and the Android nightly
cadence.

The "Not user documentation" banner addresses reader confusion, not disclosure.
The maintainer must decide explicitly: publish as-is, drop the nav entry and keep
the plan in-repo, or split the appendix into a separate unpublished file. This
should be a decision, not a side effect of a nav edit.

### gRPC public API drift

The conformance suite, rather than inheritance alone, defines compatibility. No
private gRPC classes are permitted in the v2 implementation. The aio interception
chain is the largest exposure to upstream drift, because it reimplements behavior
gRPC keeps private; its differential test against a real `grpc.aio` channel is
the control.

### Pre-intercepted channels

Pre-intercepted channels can change request objects after the safe projection is
created and therefore violate matching and security assumptions. Reject them
where detectable and document the factory contract as mandatory.

### Credential retry behavior

Credential generation and invalidation remain application-owned through opaque
per-invocation context and completion hooks.

### Client-streaming and bidi routing

Full-body matching is unavailable before request consumption. Schema v3 and
`NEW_EPISODES` require a pre-stream selector rather than hiding eager buffering or
speculative network behavior. This decision must not be relaxed for convenience
without a separately named policy that clearly states its semantic trade-offs.

The current implementation does the opposite today — `requests = [r async for r
in request_iterator]` in `src/grpcvcr/interceptors/aio.py` and
`list(request_iterator)` in `sync.py` — destroying laziness, half-close timing,
and any generator with side effects. Convenience must not reintroduce it.
