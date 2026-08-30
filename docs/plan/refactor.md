# grpcvcr 0.2 Refactor Plan

Status: review-converged and implementation-ready. `0.2` is the single release:
all four RPC shapes, one new cassette schema, no intermediate publication
(see [Resolved decisions and residual risks](#resolved-decisions-and-residual-risks)).  
Last updated: 2026-08-30 · Owner: upstream maintainer · Next review: at each phase exit

!!! warning "Not user documentation"

    This is an internal design plan for unreleased 0.2 work, kept in-repo and
    excluded from the published site by `exclude_docs: plan/**` in `mkdocs.yml`.
    Shipped behavior is described in Concepts and API Reference. Where the two
    disagree, the shipped documentation is authoritative for the current
    release — except where this plan explicitly records that the shipped
    documentation is wrong today (see [Documentation defects](#documentation-defects-in-the-current-release)).

This document defines the implementation plan for refactoring `grpcvcr` into a
safe, deterministic, and lifecycle-faithful record/replay library. The plan was
reviewed iteratively from four perspectives: gRPC runtime behavior,
security/data-format design, public API/release compatibility, and internal
consistency against the current codebase.

The immediate target is a safe `grpc.aio` integration for a downstream pilot
consumer. The design remains general-purpose and preserves a compatibility path
where one is possible. Request-streaming and bidirectional streaming ship in the
same release and the same schema as the unary shapes, so the library never
publishes a version that supports fewer shapes than 0.1.x does.

## Goals

- Replay without constructing a live gRPC channel, resolving a host, or
  obtaining credentials.
- Record and replay all four RPC shapes — unary-unary, unary-stream,
  stream-unary, and stream-stream — lifecycle faithfully for sync and `grpc.aio`
  clients; under the strict profile, message/value fidelity is explicitly limited
  to fields authorized by the recording policy.
- Preserve streaming laziness, sanitized metadata/status, deadlines, callbacks,
  and cancellation behavior at the public gRPC API boundary.
- Sanitize protobuf and metadata content before it can enter a cassette,
  diagnostic, temporary file, or library log.
- Match deterministic, sanitized request projections and consume repeated
  interactions in a defined order.
- Replace Python import-path coupling with stub-provided serializers and
  deserializers.
- Introduce a validated, versioned, atomically written cassette format.
- Preserve existing public entry points where the strict security model permits
  it, and state every place it does not.
- Keep replacement modules source-compatible with Python 3.10 for downstream
  forks that require it. Upstream 0.2 continues to require Python 3.11; it does
  not add support for a Python branch that reaches end of life in October 2026.

## Non-goals for 0.2

- Exact HTTP/2 scheduling, flow-control, or transport timing reproduction.
- Sandboxing application-provided sanitizers, credential providers, or hooks.
- Transparently routing request-streaming `NEW_EPISODES` calls by their full
  request body.
- Silently converting unframed v1 client-streaming or bidi recordings.

## Release scope

| Release | Published | Scope |
| --- | --- | --- |
| `0.2` | PyPI | Everything. Cassette v2 covering all four RPC shapes, sync and async routing, strict security profile, compatibility facades, v1 playback adapter, migration tooling, pytest and release hardening, and the correctness fixes already merged on main. |

`0.2` is the only release this plan produces. There is no `0.1.2`, no `0.3`, and
no beta. Two consequences follow, and both are accepted deliberately.

**The merged correctness fixes wait for `0.2`.** Two merged commits on `main` change replay behavior for cassettes users already hold: replayed-
error fidelity and cassette portability. A third, scoped target recording,
changes only cassettes written by `main`, because `v0.1.1` records no `target`
field at all — `TargetMatcher` and interaction-level `target` are unpublished
`0.2`-new surface, not v1 compatibility obligations. With no `0.1.2`, the last
published release stays `0.1.1` and those fixes ship only when `0.2` does —
69–107 engineer-days out. `0.1.1` therefore remains the final
published unrestricted v1 writer. If that wait becomes untenable, reinstating a
patch release is a one-row change to this table plus a tag; nothing else in the
plan depends on it.

**The single release ships behind the streaming work.** Folding the causal schema
and request-streaming into `0.2` means `0.2` cannot ship until the bidi
prototypes pass, so the pilot-ready async milestone (Phases 0–3) no longer
corresponds to anything published. The pilot must therefore install by commit —
see the rollout runbook — rather than pin a released version. Pre-1.0 semver
permits the combination at no versioning cost; the cost is entirely schedule.

The upside is real and is the reason the combination is defensible: the library
never publishes a version supporting fewer RPC shapes than 0.1.x, users never
perform a v2 → v3 cassette migration months apart, and there is one *new*
schema major to introduce, validate, and document, on top of the frozen v1
adapter that `0.2` supports either way.

The downstream pilot installs the Phase 3 branch by commit for early integration,
which pins more precisely than a pre-release and consumes no version. That
exploratory run is not release evidence: the five-night promotion gate below is
rerun against the final candidate identity tuple defined in the rollout runbook.

Because `hatch-vcs` derives the version from the last `v*` tag and no tag lands
until `v0.2.0`, every artifact built during the program self-reports
`0.1.2.dev<N>+g<sha>` — a version string for a release this plan says will never
exist, attached to a package that from Phase 3 contains the `0.2` engine. The
non-`v` `baseline-0.1.x` marker does not reset `<N>`. Consumers must pin the
commit SHA, not the version string: `dev<N>` is a commit count that collides
across branches, and version comparison ignores the `+g<sha>` local segment, so a
version specifier alone will not reliably reinstall across two candidates. The
rollout runbook's identity tuple is the reproducible handle; the reported version
is not one.

A package pin is not a data rollback: 0.1.x cannot read v2. Pilot and migration
runs therefore keep v1 and v2 in separate versioned paths, record the source
cassette hash, and never overwrite the v1 corpus. Rolling back means pinning
`0.1.1` **and** restoring the v1 path.

Note that `.github/workflows/release.yml` triggers on any `v*` tag and runs
build → publish → GitHub release, and `hatch-vcs` derives the version from that
tag. Any `v*` tag is therefore a PyPI release, not a marker. A rollback or
baseline marker without a release requires a non-`v` tag such as
`baseline-0.1.x`, which Phase 0 creates so the pre-refactor tree stays
addressable without consuming a version.

`CHANGELOG.md` is the canonical release-note source for `0.2`. Phase 6 creates
versioned sections for the prior tags, moves the complete breaking-change list
out of `[Unreleased]`, points project metadata at the file, and copies the same
text into the GitHub release rather than maintaining two independent versions.

### Rollout and rollback runbook

The release owner is the upstream maintainer; the downstream integration owner
owns the pilot evidence. Before migration, the pilot copies its v1 corpus to an
immutable backup, records SHA-256 values, and writes v2 only to a separate path.
Promotion requires every 0.2 quality gate and five consecutive pilot nights. The
Python 3.10 consumer cannot install the upstream `requires-python >=3.11` wheel,
so pilot identity is the tuple `(upstream release-tag commit SHA, upstream
wheel/sdist SHA-256 values, downstream packaging-patch SHA-256, downstream
Python-3.10 wheel SHA-256, consumer profile/config SHA-256)`. The downstream wheel
must be built from that exact upstream commit with only the recorded packaging
patch. A change to any tuple member resets the count; a byte-identical rebuild
does not. Upstream wheel/sdist installation is gated separately on Python 3.11+.
Before approval, the owners also rehearse rollback:
disable the candidate job, pin 0.1.1, restore the hashed v1 corpus, and complete
offline playback with transport, DNS, sockets, and authentication disabled.

Rollback triggers are any strict canary disclosure, unexpected `live_started`
event during playback, unreadable/misordered v2 interaction, authentication
regression, or commit yielding neither a valid old nor new cassette. Rollback is:
(1) disable 0.2 jobs, (2) pin 0.1.1, (3) restore the verified v1 path, and (4) run
playback with transport/DNS/auth disabled before re-enabling tests. PyPI yanking
may contain new installs but is not rollback. The release issue records owner,
artifact hashes, pilot run URLs, promotion approval, and any rollback execution.

`grpcvcr` is pre-1.0. Semver 2.0.0 permits any change under `0.y.z`, so `0.2` may
replace the cassette format, change the default matcher, rename fixtures, and
retire the interceptor subpackage without a major bump and with no required
deprecation cycle. That permission is what makes a single combined release
possible at no versioning cost.

## Breaking changes in 0.2

This section is normative. `0.2` is a breaking release and its release notes must
lead with this list.

### Client-streaming and bidi are re-implemented, not removed

`RecordingStreamUnaryInterceptor` and `RecordingStreamStreamInterceptor`
(`grpcvcr.interceptors`) record and replay client-streaming and bidirectional
calls in 0.1.x, and are covered by `tests/test_interceptor_paths.py`,
`tests/test_interceptor_paths_async.py`, and `tests/test_streaming_errors.py`.

`0.2` keeps both shapes working, through the routing engine and the causal event
model rather than through interceptors. The behavior changes even though the
capability does not, and the changes are the point of the refactor:

| 0.1.x behavior | 0.2 behavior |
| --- | --- |
| The request iterator is drained eagerly — `list(request_iterator)` in `interceptors/sync.py:166` and `:232`, `[r async for r in request_iterator]` in `interceptors/aio.py:318` and `:398` — before the call starts. | The iterator is consumed lazily, at the rate the call consumes it. Half-close timing is preserved and generator side effects fire in real order. |
| Requests are concatenated for matching, so a `NEW_EPISODES` miss is decided only after the whole stream is buffered. | Routing decides before the iterator is touched, via the pre-stream selector. No mode buffers a stream to choose a route. |
| Message boundaries are not recorded, so a recording cannot be replayed with correct framing. | Client message ordinals, an explicit half-close position, and a per-event client send count preserve framing and the observable request/response interleaving. |

Because message boundaries were never recorded, **v1 client-streaming and bidi
recordings cannot be migrated.** Migration classifies them `re-record required`,
and the frozen v1 adapter replays them only with their original eager, unframed
semantics, clearly labeled as legacy behavior.

`UnsupportedRpcShapeError` is **internal**, not root-exported. It subclasses
`GrpcvcrError` and exists only for the interim state of `main` between Phase 3
and Phase 5, where `stream_unary` and `stream_stream` are routed but not yet
implemented. In that interim it is raised from the multicallable's `__call__`,
before the request iterator is touched and before any transport factory,
invocation preparer, or credential provider runs.

The released `0.2` cannot raise it: `RpcShape` is a closed `Literal` over the
four standard shapes, and a registry gap is a `ConfigurationError`.
Root-exporting an error no released version can raise would create permanently
supported public surface with no reachable behavior — and, under
`fail_under = 100`, would require a test that fabricates an out-of-`Literal`
shape to cover a branch the type system says is unreachable. This follows the
pattern commit `9b09963` already established here for `ReplayedRpcError`: name
the error for when it is raised, and keep it internal.

**No** channel factory method raises. All four always return a working
multicallable object, and registry resolution is deferred to `__call__`.
Generated stubs build every multicallable in `Stub.__init__`, so raising at
factory time would break stub construction for any service that merely declares a
method the test never calls — an argument that applies to the unary shapes
exactly as it does to the streaming ones, which is why the rule is symmetric.

`MessageTooDeepError`, `NonCanonicalPayloadError`, `UnresolvableAnyPayloadError`,
and `ProfileMismatchError` are likewise new in 0.2, subclass `GrpcvcrError`, and
are exported from the package root. The first three additionally subclass
`SerializationError`, so existing handlers around cassette load and save keep
working.

`ConfigurationError` is also new, root-exported, and subclasses `GrpcvcrError`.
Every session-construction rejection named by this plan uses it rather than an
unrelated `ValueError`.

`FinalizationError` is new, root-exported, and subclasses `GrpcvcrError`. It is
raised by `close()` / normal context exit only after all finalizers settle and any
otherwise valid interactions have committed. It carries an immutable safe report
as parallel `.ordinals: tuple[int, ...]` and
`.reasons: tuple[SafeEventReason, ...]`, plus
`.commit_state == "committed"`; it never carries callback exceptions.

`TransportCloseError` is new, root-exported, and subclasses `GrpcvcrError`. It is
raised from terminal close only after other valid interactions commit, has the
constant `.reason == "transport_close"` and `.commit_state == "committed"`, and
safe `.count: int`, and never retains the channel, target, or raw close exception.

### A `grpcvcr` console script is added

`0.2` adds `[project.scripts] grpcvcr = "grpcvcr.__main__:main"`; `pyproject.toml`
declares no `[project.scripts]` today. Subcommands: `profile-hash` (print the
canonical profile document and its `config_sha256`), `validate` (validate a
cassette against its schema and profile), and `migrate` (v1 → v2 dry-run and
conversion).

The CLI is public surface and is included in Phase 1's proposed 0.2 snapshot like
any other export. Phase 6 wires the console script; because `profile-hash` is
described under [Profile identity](#profile-identity) and is needed to debug a
`ProfileMismatchError` as soon as profiles exist, Phase 2b lands it as a module
entry point first.

CLI behavior is stable public API:

| Exit | Meaning |
| ---: | --- |
| `0` | Valid input and completed operation. |
| `2` | Command-line usage or configuration error. |
| `3` | Invalid cassette, unsupported schema/feature, or profile mismatch. |
| `4` | Migration classified at least one interaction as `re-record required`; no output was written. |
| `5` | Safe I/O, lock, conflict, or commit failure. |

Human-readable diagnostics go to stderr and contain only the strict safe set.
`--json` emits a versioned object to stdout for automation. `validate` is
read-only. `migrate --dry-run` acquires no writer lock and creates no output or
temporary file. `migrate` refuses a source-equal, existing, or symlinked
destination unless a future separately reviewed `--replace` contract is added.

`profile-hash` requires `--config module:object` naming a `RecordingProfile` or
`SessionConfig`. `validate PATH` and `migrate SOURCE DEST` accept the same option;
`--registry module:object` supplies a registry only when the selected config has
none, and supplying both is an error. These are caller-authored imports, never
values discovered in cassette data. A bare `LEGACY_RAW` profile requires
`--registry` because its canonical document includes the snapshotted method codec
map; omission is exit 2, never an empty-map guess.

### The `grpcvcr.interceptors` subpackage is replaced

`src/grpcvcr/interceptors/__init__.py` exports five public names:
`RecordingUnaryUnaryInterceptor`, `RecordingUnaryStreamInterceptor`,
`RecordingStreamUnaryInterceptor`, `RecordingStreamStreamInterceptor`, and
`create_interceptors`. The v2 routing architecture replaces interception
entirely.

Phase 0 snapshots all five. All five are removed in 0.2. They cannot be honest
facades over a routing channel: an interceptor is installed only after a real
channel has already been constructed, and it does not receive the response
deserializer needed for import-free v1 replay. The release notes direct users to
`recorded_channel` / `aio_recorded_channel` and identify `0.1.1` as the last
published release with the interceptor API. The async interceptors in `interceptors/aio.py` are not
re-exported from the subpackage's `__all__` and are not a separate compatibility
surface.

### `async_recorded_channel` is replaced by `aio_recorded_channel`

The existing helper is a synchronous context manager around an aio channel and
cannot await close. It is removed in 0.2 rather than preserved as a shim that can
deadlock or return before persistence. `aio_recorded_channel` is an async context
manager with the same `path, target` positional pair and the new configuration
keywords. `AsyncRecordingChannel` remains available for callers already using
`async with`.

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
| `CassetteWriteError` | `.reason`, `.commit_state` | Added as safe enums; commit state is `not_committed` or `unknown`. |
| `SerializationError` | `.cause` | Removed. |

`CassetteWriteError` is exported but never raised anywhere in `src/` or `tests/`;
`CassetteSerializer.save` raises `SerializationError` on write failure
(`src/grpcvcr/serialization.py:475`). In 0.2 it is wired up for filesystem,
locking, conflict, flush, and commit failures. `SerializationError` is reserved
for parsing, schema validation, protobuf encoding, and canonicalization failures.

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

Strict-profile exceptions are raised through one chain-clearing helper. Clearing
inside the handler and immediately writing `raise err from None` is insufficient:
CPython repopulates `__context__` from the exception active at raise time. The
helper catches its own safe exception, clears the chain, and uses a bare re-raise;
this also works when the caller invoked grpcvcr from inside an unrelated `except`:

```python
from typing import NoReturn


def _raise_strict(err: BaseException) -> NoReturn:
    try:
        raise err from None
    except BaseException as caught:
        caught.__cause__ = None
        caught.__context__ = None
        caught.__suppress_context__ = True
        raise


safe_error: BaseException | None = None
try:
    ...
except Exception:  # deliberately do not bind the raw exception
    safe_error = CassetteWriteError(path)
if safe_error is not None:
    _raise_strict(safe_error)
```

A security test invokes each failure both normally and while handling a canary
exception, then asserts, for every strict-profile exception, that
`__cause__ is None` **and** `__context__ is None`, and that no canary appears in
`traceback.format_exception(e)` or in `repr(e.__dict__)`. Asserting on the
rendered default traceback alone would pass while the data is still reachable.

Normative for every exception raised under `protobuf-safe`, including replayed
`grpc.RpcError`, `grpc.aio.AioRpcError`, and local cancellation: it uses the
central strict raiser, and its message is a constant plus values drawn only from the safe set — method path,
shape, FQN, field path, ordinal, consumption state, status code, digest, limit
name and configured value, profile id and digest, reason enum, commit state, and
the caller-supplied cassette path. No payload bytes, no decoded field values, no metadata values, no
OS error text, and no filesystem path the caller did not supply. This covers the
exceptions introduced by this plan — `UnsupportedRpcShapeError`,
`MessageTooDeepError`, `UnresolvableAnyPayloadError`, `NonCanonicalPayloadError`,
`ProfileMismatchError`, `ConfigurationError`, `FinalizationError`, and
`TransportCloseError` — whose natural implementations would each interpolate
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

That port is rejected. Every strict selection begins with one unconditional base
selector `(method, shape, request_fqn, deterministic_safe_projection)`. Strict
mode then accepts an optional, distinct `StrictMatcher` ABC for **additive**
constraints. Its `selector_fragment(SafeRequestProjection) -> bytes` returns a
canonical fragment, and its non-empty `semantic_id` identifies that fragment's
meaning. `StrictAllMatcher` length-prefixes the ordered semantic IDs and fragments
of its children. `StrictMetadataMatcher` and `StrictTargetMatcher` are the built-in
additions; method and projection matchers do not exist because removing either
base invariant is not allowed.

Custom `StrictMatcher` subclasses are supported. They see only the sanitized
projection, must return deterministic bytes, and are included in ambiguity checks
exactly like built-ins. The fragment is never rendered; diagnostics may contain
only its SHA-256. `semantic_id` is caller-declared non-secret configuration and may
appear in errors. A missing/duplicate semantic ID, non-bytes fragment, or a
fragment that differs across two immediate evaluations is a configuration or
interaction failure. Passing a v1 `Matcher` — including `MethodMatcher`,
`RequestMatcher`, `CustomMatcher`, `MetadataMatcher`, `TargetMatcher`, and any
`__and__` composition — to a strict session raises at configuration time. The v1
`Matcher` hierarchy remains fully functional under `legacy-raw`, where bodies are
still wire bytes and today's semantics are unchanged.

The strict default has no additive matcher. Metadata and target participate only
through an explicit `StrictMetadataMatcher`, `StrictTargetMatcher`, or composition.

`StrictMetadataMatcher` against a key outside the metadata allowlist, and
`StrictTargetMatcher` against a session with `target_persistence: false`, are
likewise configuration errors raised at session construction rather than on first
mismatch.

### Strict replay returns policy-authorized response fields

Live calls still return the application's original deserialized response. A
strict cassette stores only the sanitized response projection, so replay returns
only fields authorized by `keep`/`keep_shape`; default-dropped fields have their
protobuf absent/default value. Tests requiring exact response values must keep
those fields explicitly. `legacy-raw` retains byte-for-byte codec replay but makes
no strict security claim.

### Cassette surface changes

`Cassette.interactions` returns v1 `Interaction` objects. Backed by a v2
cassette under the strict profile it returns immutable views, not `Interaction`.
`Cassette.record_interaction` is a raw path and is available only under
`legacy-raw`.

Four more public `Cassette` members break, and all four are unavoidable:

| Method | Break |
| --- | --- |
| `get_response(method, request_body, metadata)` | Returns `Interaction`, so it inherits the immutable-view ruling above. Its documented `Raises:` contract is `NoMatchingInteractionError`, whose attributes are removed above — and this method is the call site that passes `request_body` and `self.interactions` into it. It is also a stateless find-and-return, incompatible with the atomic reserve/consume lifecycle. |
| `find_interaction(InteractionRequest)` | Same stateless-lookup problem: it cannot reserve, so it cannot participate in ordered consumption. |
| `save()` | Becomes a terminal operation: it seals an unbound/sync session against new calls and commits, rather than acting as a reusable checkpoint. It also gains formal source-hash conflict, symlinked-destination, lock, and commit-state failures. The 0.1.x implementation actually raises `SerializationError` for writes despite exporting the unused `CassetteWriteError`; 0.2 corrects that defect and wires filesystem failures to `CassetteWriteError`. An open aio-bound session rejects synchronous `save()` before sealing and must be closed through awaited channel close. |
| `Cassette.__exit__` | An escaping application exception now aborts a still-open transaction instead of unconditionally saving it. For an aio-bound session, sync exit is valid only after the async channel has already completed its awaited close. A premature normal sync exit rejects before sealing; a premature exceptional exit preserves the body exception and leaves the outer async owner to abort. |

`Cassette("test.yaml")` string-path coercion, `use_cassette`,
`Matcher.__and__` composition, and the `grpcvcr.serialization` names
(`InteractionRequest`, `InteractionResponse`, `StreamingInteractionResponse`,
`Interaction`, `CassetteData`, `CassetteSerializer`) are all in the Phase 1
snapshot.

The ruling is exact: existing matcher and serialization-model names remain
functional and deprecated for v1 playback and explicit `legacy-raw` sessions.
`find_matching_interaction`, `get_response`, `find_interaction`, and
`record_interaction` remain callable only there; a strict session raises
`ConfigurationError` and directs callers to the internal atomic reservation API,
which is not public in 0.2. Strict `interactions` is a tuple of
`ImmutableInteractionView`. `CassetteSerializer` remains public and dispatches
v1/v2 reads, but only writes v2 through a session transaction.

`CassetteSerializer` dispatches on file extension to JSON or YAML today, and
`Cassette.path` documents both. **Schema v2 is YAML-only.** Opening a `.json`
path in a v2 record-capable mode is an error naming the restriction; the v1
adapter continues to read `.json` for playback.

### pytest plugin surface changes

- `grpcvcr_channel` is removed. It raises `NotImplementedError` unconditionally
  today and is undocumented in `docs/guides/pytest.md`, so removal is preferred
  over making it functional.
- The `grpcvcr_cassette` fixture defaults to `grpcvcr_config`; under the default
  strict profile that selects the strict method-plus-projection matcher.
  `DEFAULT_MATCHER` remains the legacy default only.
- No implicit CI detection is added. The precedence remains CLI override, marker,
  then `NEW_EPISODES`; CI that must never go live sets `--grpcvcr-record=none`
  explicitly. Phase 0 removes the false automatic-detection claim.

The Phase 0 baseline covers the six current fixtures
(`grpcvcr_cassette_dir`, `grpcvcr_record_mode`, `grpcvcr_cassette`,
`grpcvcr_channel`, `grpcvcr_channel_factory`, `grpcvcr_async_channel_factory`),
two CLI options (`--grpcvcr-record`, `--grpcvcr-cassette-dir`), and one marker
with both positional and keyword forms. The Phase 1 proposed 0.2 surface removes
`grpcvcr_channel`, adds `grpcvcr_config`, and lets the marker accept `config=`;
the factory fixtures consume that configuration. `grpcvcr_record_mode` and
`grpcvcr_async_channel_factory` are public but undocumented in 0.1.x and become
documented in 0.2.

### The base exception is renamed `GrpcvrError` → `GrpcvcrError`

The published `0.1.1` root-exports `GrpcvrError` — no `c` — and every grpcvcr
exception subclasses it. Commit `2354b78` renamed it to `GrpcvcrError` with no
alias anywhere in the tree. That rename was going to reach users in `0.1.2`; with
no `0.1.2`, `0.2` is the first release carrying it, so `except GrpcvrError`
breaks for every `0.1.1` user and `from grpcvcr import GrpcvrError` raises
`ImportError`.

`0.2` root-exports `GrpcvrError` as a deprecated alias of `GrpcvcrError`,
emitting `DeprecationWarning` on attribute access via a module `__getattr__`.
The alias is removed no earlier than `0.3`. Phase 0's baseline snapshot is taken
against the published `v0.1.1` surface, not against `main`, so this name is
captured; Phase 4 resolves it as a deprecation.

This is the highest-impact consequence of dropping `0.1.2`, and it is visible
only when the baseline is diffed against the tag rather than the working tree.

### Shipped JSON-storage documentation becomes false

`README.md:36`, `docs/index.md:36`, `docs/concepts/index.md:16`,
`docs/concepts/cassettes.md:7`, and `CHANGELOG.md:15` all advertise JSON as a
cassette storage format. Schema v2 is YAML-only, so from `0.2` a `.json` path is
an error in any record-capable mode; only the frozen v1 adapter still reads
`.json`, for playback. The exact wording differs by file — "YAML or JSON" in
`README.md`, `docs/index.md`, and `docs/concepts/`, versus "YAML and JSON
cassette formats" in `CHANGELOG.md` — so a sweep matching one string will miss
two files.

The shipped all-shape claims in `README.md:32`, `docs/index.md:32`,
`docs/concepts/index.md:28`, `docs/concepts/streaming.md:3`, and
`CHANGELOG.md:13` remain **true** in `0.2` and are not retracted. Their worked
examples are rewritten in Phase 6: the examples are built on the
`grpcvcr.interceptors` API, which `0.2` replaces, and on eager request-iterator
draining, which `0.2` replaces with lazy consumption.

## Documentation defects in the current release

These are shipped bugs in `0.1.x`, not `0.2` breaking changes, and they are
listed separately so they do not enter the `0.2` breaking-change release notes.
Phase 0 fixes them because they are wrong *today*.

| File | Claim | Status |
| --- | --- | --- |
| `docs/guides/ci-testing.md` | "grpcvcr automatically detects CI environments and sets `RecordMode.NONE` by default … CI is detected via the `CI` environment variable" | **False today.** No module under `src/grpcvcr/` reads any environment variable; `grpcvcr_record_mode` falls through to `NEW_EPISODES`. CI defaults to *recording*, so following this guide means live calls and credentials written into cassettes from CI. |
| `CHANGELOG.md:22` | "Automatic `RecordMode.NONE` in CI environments" | Same defect. |
| `docs/guides/ci-testing.md:13-19` | Worked example using fixtures `cassette` and `grpc_target` | **Neither fixture exists.** The plugin provides `grpcvcr_cassette` and no target fixture at all, so the example cannot run. |

Phase 0 removes the claim from the guide and changelog; the resolved 0.2 decision
is to require explicit `--grpcvcr-record=none`, not to add environment detection.
The fix covers the whole guide, not one sentence:
`docs/guides/ci-testing.md` also carries a
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
      ├── encode request once (unary-request shapes)
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
a continuation that may be invoked repeatedly, as grpcio permits; an interceptor
that returns either a call object or a plain response; `ClientCallDetails`
replacement; and the response-iterator wrapper an interceptor sees for
unary-stream. Each continuation invocation is a distinct routed RPC. A
short-circuit response returned without invoking the continuation is
application-owned and creates no cassette interaction. This chain is grpcvcr
code with its own conformance tests against the behavior of `grpc.aio` with the
same interceptors on a real channel; it is not an import of gRPC internals.
Encoding and safe projection happen at the routing continuation after all
preceding interceptor request and `ClientCallDetails` replacements, so
unary-request matching uses the bytes that would actually be emitted. For
stream-request shapes there is nothing to encode at continuation time: routing
uses the pre-stream selector and the request iterator is passed through
untouched. The stream interceptor contract is
`await continuation(client_call_details, request_iterator) -> Call`; in
reader/writer style grpcvcr supplies the same kind of bounded proxy async
generator `grpc.aio` does, and neither the chain nor the projection consumes it.
Repeated continuation invocations each perform their own reservation, and for
unary-request shapes their own single encode; a stream-request iterator is
consumed at most once across all invocations, so an interceptor that invokes the
continuation twice on one stream-request call gets an exhausted iterator on the
second, exactly as it does on a real `grpc.aio` channel.

Existing `RecordingChannel`, `AsyncRecordingChannel`, `.channel`, `.cassette`,
and `.target` APIs remain compatibility facades over the new engine.

The compatibility baseline enumerates every name in `grpcvcr.__all__` as
published at `v0.1.1` (20 exports, including `GrpcvrError` and excluding
`TargetMatcher`) and `grpcvcr.interceptors.__all__` (5 exports), **plus** the module-level names in
`grpcvcr.matchers` and `grpcvcr.serialization` that the public API exposes
transitively. Those two modules and `grpcvcr.cassette` declare no `__all__`, so an
`__all__`-only definition would exclude names this document already treats as
breaking changes — `DEFAULT_MATCHER`, `find_matching_interaction`, and the
`serialization` types listed under
[Cassette surface changes](#cassette-surface-changes). Phase 0 snapshots the full
enumerated 0.1.x list. Phase 1 records the proposed 0.2 list, and the final 0.2
snapshot is generated from the implementation and
must include all seven new root-exported errors — `UnsupportedRpcShapeError` is
internal — plus whichever configuration and
routing types the public-API section below marks as exported; no hand-maintained
numeric total is normative.

The existing synchronous `async_recorded_channel` helper is removed. It cannot
offer a correct deprecation shim: when used from its channel's running event-loop
thread, a synchronous `__exit__` cannot wait for `channel.close()` without
deadlocking. `aio_recorded_channel` is the proper async replacement and performs
awaited close and finalization. `AsyncRecordingChannel` remains available, so
callers that already use `async with` have a direct migration path.

## Glossary

| Term | Definition |
| --- | --- |
| Shape | One of `unary_unary`, `unary_stream`, `stream_unary`, `stream_stream`. |
| `CallDescriptor` | Immutable per-invocation facts handed to the live-invocation preparer: method path, shape, request/response FQNs, request metadata, and remaining deadline. Carries no request payload. |
| `LiveInvocation` | The public, user-constructible preparer return: additional request metadata, optional per-call credentials, and an opaque application context object. |
| `RealCall` | The underlying `grpc`/`grpc.aio` call object returned by the delegated transport, once it exists. |
| `SafeOutcome` | The sanitized terminal result of a call — status code, details (present only under `status_details: keep`, otherwise `None`), and sanitized trailing metadata — passed to completion hooks. Never carries payload bytes. |
| `SafeRequestProjection` | The immutable strict-selector input: method, shape, request FQN, deterministic sanitized protobuf bytes, sanitized selected metadata, and optionally authorized target. For `stream_unary` and `stream_stream`, `request_fqn` comes from the registry and `protobuf_bytes` is `b""`, because no client message has been read at routing time. Never contains the bytes sent on the wire. |
| Event grammar | The profile-independent set of v2 event kinds, their per-shape cardinality, and their ordering and send-count rules — the contract Phase 1's prototypes freeze and Phase 2a's validator encodes. |
| Pre-stream selector | The strict selector for a stream-request shape: the base tuple with `request_fqn` from the registry and `protobuf_bytes` empty, plus any additive `StrictMatcher` fragments. Computed at `__call__` from facts that exist before the first client message, so routing never touches the request iterator. |
| `PersistableEvent` | A validated cassette event (`client_message`, `client_half_close`, `server_initial_metadata`, `server_message`, `server_trailing_metadata`, `terminal_status`) constructed only from sanitized inputs. |
| `SafeMismatchReport` | The structured diagnostic attached to a strict `NoMatchingInteractionError`: method, shape, safe FQNs, candidate ordinal, consumption state, and safe digests. |
| `MethodDescriptorRegistry` | A mutable pre-session builder mapping method paths to shapes, protobuf classes, and optional codec IDs. `Cassette` snapshots it into a private immutable descriptor pool; there is no implicit process-global pool. Actual serializers/deserializers still come from the stub call. |
| Opaque-context finalization | The single controller-owned invocation of the completion/failure hook with the `LiveInvocation` context and a `SafeOutcome`, after which the context is released. Runs exactly once after a preparer returned a context, including later cancellation or transport startup failure; there is no hook when preparation itself failed. |
| `SafeEventSink` | Optional observability callback receiving immutable, safe-field-only lifecycle events. It is distinct from completion hooks; sink exceptions do not change state, while its documented inline latency can delay later work. |
| Generation | One session's in-memory interaction list. `ALL` starts an empty generation rather than loading the file, so the file is replaced, not merged. |
| Global strict-sequence playback | The cassette's interactions are consumed in recorded ordinal order; interaction *n* must be the *n*th invocation, and a mismatch is an error. |
| First-unused matching | The first still-`unused` interaction whose selector matches is consumed, regardless of ordinal. Permits reordered and interleaved calls. |
| Sidecar writer lock | An exclusive advisory OS lock on a `<cassette>.lock` file adjacent to the cassette, held for the duration of a writable session. |
| Source hash | SHA-256 of the cassette file bytes, read under the sidecar writer lock at session entry and re-read under the same held lock immediately before commit. It detects cooperative conflicts and most non-locking edits but cannot eliminate a final race with a process that ignores the advisory lock. |
| Client-message send count | On each server event, the number of client messages the client had submitted to the transport when it observed that event. A client-side approximation of causality; an upper bound on server consumption, not a claim about server state or wall-clock timing. |

The two recording profiles are named exactly once, here, and used verbatim
elsewhere: `protobuf-safe` (the strict profile) and `legacy-raw` (the permissive
profile). "Strict" is an adjective meaning "under the `protobuf-safe` profile".

## Normative public configuration

Phase 1 freezes these exported configuration types before implementation work
begins:

```python
from abc import ABC, abstractmethod
from collections.abc import Awaitable, Callable, Sequence
from dataclasses import dataclass
from typing import Literal, Protocol

import grpc
from google.protobuf.descriptor_pool import DescriptorPool
from google.protobuf.message import Message

RpcShape = Literal["unary_unary", "unary_stream", "stream_unary", "stream_stream"]
MetadataEntry = tuple[str, str | bytes]
ChannelOption = tuple[str, str | int]

@dataclass(frozen=True)
class CallDescriptor:
    method: str
    shape: RpcShape
    request_fqn: str | None
    response_fqn: str | None
    request_metadata: tuple[MetadataEntry, ...]
    time_remaining: float | None

@dataclass(frozen=True)
class LiveInvocation:
    additional_metadata: tuple[MetadataEntry, ...] = ()
    credentials: grpc.CallCredentials | None = None
    context: object | None = None

@dataclass(frozen=True)
class SafeOutcome:
    code: grpc.StatusCode
    details: str | None
    trailing_metadata: tuple[MetadataEntry, ...]

@dataclass(frozen=True)
class SafeRequestProjection:
    method: str
    shape: RpcShape
    request_fqn: str            # from the registry for stream-request shapes
    protobuf_bytes: bytes       # b"" for stream_unary and stream_stream
    metadata: tuple[MetadataEntry, ...]
    target: str | None

ProjectionTransform = Callable[[str, Message], Message]
MetadataTransformer = Callable[[str, int, str | bytes], str | bytes]

SafeEventKind = Literal[
    "replay_hit", "replay_miss", "live_started", "startup_failed",
    "interaction_discarded", "hook_failed", "hook_slow", "transport_close_failed",
    "source_conflict", "commit_succeeded", "commit_failed",
]
CommitState = Literal["not_committed", "committed", "unknown"]
SafeEventReason = Literal[
    "no_candidate", "transport_factory", "invocation_preparer",
    "delegate_creation", "hook_watchdog", "hook_exception",
    "local_cancel", "abandoned", "request_invalid", "response_invalid", "limit_exceeded",
    "transport_close", "path_rejected", "lock_open", "writer_busy",
    "temp_create", "temp_write", "permission_apply", "serialize", "validate",
    "flush", "source_changed", "replace", "directory_sync", "cleanup",
    "storage_io",
]

@dataclass(frozen=True)
class SafeEvent:
    kind: SafeEventKind
    sequence: int
    method: str | None = None
    shape: RpcShape | None = None
    request_fqn: str | None = None
    response_fqn: str | None = None
    ordinal: int | None = None
    status_code: grpc.StatusCode | None = None
    consumption_state: Literal["unused", "reserved", "consumed"] | None = None
    reason: SafeEventReason | None = None
    digest: str | None = None
    expected_digest: str | None = None
    observed_digest: str | None = None
    limit_name: str | None = None
    limit_value: int | None = None
    profile_id: str | None = None
    profile_digest: str | None = None
    cassette_path: str | None = None
    commit_state: CommitState | None = None

class SafeEventSink(Protocol):
    def __call__(self, event: SafeEvent, /) -> None: ...

class StrictMatcher(ABC):
    @property
    @abstractmethod
    def semantic_id(self) -> str: ...

    @abstractmethod
    def selector_fragment(self, request: SafeRequestProjection, /) -> bytes: ...

    def __and__(self, other: "StrictMatcher", /) -> "StrictAllMatcher": ...

class StrictMetadataMatcher(StrictMatcher):
    def __init__(self, *keys: str) -> None: ...
    @property
    def keys(self) -> tuple[str, ...]: ...
    @property
    def semantic_id(self) -> str: ...
    def selector_fragment(self, request: SafeRequestProjection, /) -> bytes: ...

class StrictTargetMatcher(StrictMatcher):
    def __init__(self) -> None: ...
    @property
    def semantic_id(self) -> str: ...
    def selector_fragment(self, request: SafeRequestProjection, /) -> bytes: ...

class StrictAllMatcher(StrictMatcher):
    def __init__(self, *members: StrictMatcher) -> None: ...
    @property
    def members(self) -> tuple[StrictMatcher, ...]: ...
    @property
    def semantic_id(self) -> str: ...
    def selector_fragment(self, request: SafeRequestProjection, /) -> bytes: ...

@dataclass(frozen=True)
class MetadataRule:
    key: str
    kind: Literal["text", "binary"]
    transformer_id: str
    transformer: MetadataTransformer

@dataclass(frozen=True)
class CassetteLimits:
    max_cassette_bytes: int = 32 * 1024 * 1024
    max_yaml_nodes: int = 100_000
    max_yaml_depth: int = 64
    max_scalar_chars: int = 8 * 1024 * 1024
    max_interactions: int = 1_000
    max_events_per_interaction: int = 10_000
    max_payload_bytes: int = 4 * 1024 * 1024
    max_protobuf_nodes: int = 100_000
    max_metadata_entries: int = 64
    max_metadata_value_bytes: int = 8 * 1024
    max_message_depth: int = 100

DEFAULT_LIMITS = CassetteLimits()

class MethodDescriptorRegistry:
    def __init__(self, *, pool: DescriptorPool | None = None) -> None: ...

    def register(
        self,
        method: str,
        *,
        shape: RpcShape,
        request_type: type[Message],
        response_type: type[Message],
        request_codec_id: str | None = None,
        response_codec_id: str | None = None,
    ) -> None: ...

    def register_legacy(
        self,
        method: str,
        *,
        shape: RpcShape,
        request_codec_id: str,
        response_codec_id: str,
    ) -> None: ...

@dataclass(frozen=True)
class FieldRule:
    path: str
    action: Literal["keep", "keep_shape", "drop"]
    allow_opaque_any: bool = False

@dataclass(frozen=True)
class RecordingProfile:
    id: Literal["protobuf-safe", "legacy-raw"]
    version: int
    field_policy: tuple[FieldRule, ...] = ()
    metadata_policy: tuple[MetadataRule, ...] = ()
    status_details: Literal["drop", "keep"] = "drop"
    target_persistence: bool = False
    projection_transform: ProjectionTransform | None = None
    projection_transform_id: str | None = None

PROTOBUF_SAFE = RecordingProfile(id="protobuf-safe", version=1)
LEGACY_RAW = RecordingProfile(
    id="legacy-raw", version=1,
    status_details="keep", target_persistence=True,
)

@dataclass(frozen=True)
class SessionConfig:
    profile: RecordingProfile = PROTOBUF_SAFE
    registry: MethodDescriptorRegistry | None = None
    matcher: StrictMatcher | Matcher | None = None
    consumption: Literal["strict_sequence", "first_unused"] | None = None
    limits: CassetteLimits = DEFAULT_LIMITS
    event_sink: SafeEventSink | None = None
```

The package root adds `aio_recorded_channel`, `RoutingChannel`,
`AsyncRoutingChannel`, `SessionConfig`, `RecordingProfile`, `FieldRule`,
`MetadataRule`, `CassetteLimits`, `MethodDescriptorRegistry`, `CallDescriptor`,
`LiveInvocation`, `SafeOutcome`, `SafeRequestProjection`, `SafeEvent`,
`SafeEventSink`, `RpcShape`, `MetadataEntry`, `ChannelOption`, `SafeEventKind`, `SafeEventReason`,
`CommitState`, `ProjectionTransform`, `MetadataTransformer`, `DEFAULT_LIMITS`, all six
transport/preparer/completion callback aliases, `StrictMatcher`, `StrictMetadataMatcher`,
`StrictTargetMatcher`, and `StrictAllMatcher`, plus the seven root-exported
errors named under breaking changes and the deprecated `GrpcvrError` alias.
`UnsupportedRpcShapeError` is internal and is not among them. The root
removes `async_recorded_channel`. `PROTOBUF_SAFE` and `LEGACY_RAW` are exported
immutable profile constants. This list, combined with retained 0.1.x root names,
is the normative 0.2 root surface; Phase 1 snapshots it directly.

`MetadataRule` lowercases and validates `key`; `kind` must agree with its `-bin`
suffix. Its transformer receives `(lowercase_key, occurrence_index, value)`, must
return `str` for text or `bytes` for binary, and is trusted code. `CassetteLimits`
rejects booleans, non-integers, and non-positive values at construction.
Configuration dataclasses are immutable. Supplying a projection or metadata
transformer without its non-empty non-secret semantic ID is a configuration error.
`ProjectionTransform` receives `(protobuf_fqn, cloned_message)` for either
direction and must return a message of the same FQN.

Built-in strict matchers are immutable. `StrictMetadataMatcher(*keys)` requires at
least one allowlisted key, ASCII-lowercases/deduplicates/sorts them, and exposes
that tuple. Its semantic ID is `strict-metadata-v1(<comma-joined-keys>)`; its
fragment is canonical UTF-8 JSON over the in-order duplicate-preserving entries
`[key, "text", value]` or `[key, "binary", canonical_base64]`, with sorted object
keys where applicable and no whitespace. `StrictTargetMatcher()` uses fixed
semantic ID `strict-target-v1` and its fragment is the authorized target's UTF-8
bytes. `StrictAllMatcher(*members)` requires at least two members,
flattens nested compositions, preserves constructor order, and rejects duplicate
semantic IDs. Its semantic ID is `strict-all-v1:<sha256>` over the same
length-prefixed ordered child semantic IDs; its fragment
concatenates each child semantic-ID UTF-8 value and child fragment, each preceded
by an unsigned eight-byte big-endian length. `a & b` is exactly
`StrictAllMatcher(a, b)`. Custom matcher IDs are governed by the contract above.

`SessionConfig()` selects `protobuf-safe`, global strict-sequence consumption,
and the unconditional strict base selector with no additive matcher. Its empty
field and metadata policies safely drop all protobuf fields and metadata.
`legacy-raw` is explicit opt-in and selects the existing `DEFAULT_MATCHER` only
when `matcher` is omitted. Passing a v1 `Matcher` to `protobuf-safe`, or a
`StrictMatcher` to `legacy-raw`, fails at session construction.

`registry=None` means an empty registry; it never wraps the mutable process-global
default pool. An actual strict call, new legacy-v2 call, or migration therefore
supplies an explicit registry. **No** multicallable factory resolves the registry
at construction. All four factories defer resolution to `__call__`, after v1
schema dispatch, because generated stubs build every multicallable in
`Stub.__init__` and a bounded registry must not break construction of a stub for
a service that merely declares a method the test never calls — an argument that
applies to the unary shapes exactly as it does to the streaming ones. A missing
method is a `ConfigurationError` raised from `__call__`, before the request
iterator is touched and before any transport factory, invocation preparer, or
credential provider runs. The frozen v1 raw adapter is the only call path that
needs no registry.

Registry population is explicit and uses application-provided protobuf classes:

```python
registry.register(
    "/package.Service/Method",
    shape="unary_unary",
    request_type=package_pb2.Request,
    response_type=package_pb2.Response,
    request_codec_id="application.grpc-protobuf-v1",  # optional in strict mode
    response_codec_id="application.grpc-protobuf-v1",
)

registry.register_legacy(
    "/package.Service/LegacyMethod",
    shape="unary_unary",
    request_codec_id="application.custom-codec-v3",
    response_codec_id="application.custom-codec-v3",
)
```

Registration is locked and returns `None`. Re-registering a byte-for-byte
equivalent entry is idempotent; a conflicting method, shape, type, or one-sided
codec pair raises `ConfigurationError`. `pool=` is only an additional lookup
source for named packed types; it is not retained as the live session pool.

At `Cassette` construction, grpcvcr takes one locked snapshot of the explicit
method/codec maps and copies the transitive descriptors needed by registered
types and every packed FQN named by the field policy into a new private
`DescriptorPool`. Missing/conflicting descriptors fail construction. Later
`registry.register*()` calls, imports into an explicitly supplied default pool,
and mutations used by another session cannot affect this session's method set,
`Any` resolution, descriptor digest, or legacy codec-map hash; they affect only a
future snapshot. The callable policy objects themselves remain trusted and are
identified by their required semantic IDs.

Those classes provide descriptors and the pool. The request serializer and
response deserializer passed by the generated stub to the channel factory remain
the actual codecs used for the call and are checked against the registered FQNs;
grpcvcr does not introspect them to discover types. Codec IDs are caller-managed,
non-secret semantic identifiers; grpcvcr cannot prove that a callable implements
the claimed codec. New `legacy-raw` v2 recording requires an explicit registry
entry with both direction IDs, stores the appropriate ID on every payload, and
requires an exact configured/stored ID match before invoking a stub deserializer.
The registry never invents codec IDs. Frozen v1 playback
has no IDs and retains its historical caller-supplied-codec behavior.

For every strict interaction, the registry also computes `descriptor_sha256` over
the deterministic, name-sorted, transitive `FileDescriptorProto` closure for the
method, request/response messages, extensions, and packed types authorized by the
policy, with `source_code_info` removed. The digest includes the method path and
shape. It is stored per interaction and must match the reader's recomputation
before parsing; the same FQN with an evolved field schema is therefore not treated
as compatible. Descriptor identity is validated separately rather than duplicated
inside the profile hash.

The migration CLI accepts
`--registry module:object`, an application-chosen import, and requires it when the
selected config has no explicit registry because a standalone cassette supplies
no trusted module imports. It never imports a module path read from a cassette.

The helper signatures add configuration without changing the existing positional
`path, target` pair:

```python
recorded_channel(
    path: str | Path, target: str, *,
    record_mode: RecordMode = RecordMode.NEW_EPISODES,
    config: SessionConfig | None = None,
    match_on: StrictMatcher | Matcher | None = None,
    transport_factory: SyncTransportFactory | None = None,
    invocation_preparer: SyncLiveInvocationPreparer | None = None,
    completion_hook: SyncCompletionHook | None = None,
    credentials: grpc.ChannelCredentials | None = None,
    options: Sequence[ChannelOption] | None = None,
) -> ContextManager[RoutingChannel]

aio_recorded_channel(
    path: str | Path, target: str, *,
    record_mode: RecordMode = RecordMode.NEW_EPISODES,
    config: SessionConfig | None = None,
    match_on: StrictMatcher | Matcher | None = None,
    transport_factory: AsyncTransportFactory | None = None,
    invocation_preparer: AsyncLiveInvocationPreparer | None = None,
    completion_hook: AsyncCompletionHook | None = None,
    credentials: grpc.ChannelCredentials | None = None,
    options: Sequence[ChannelOption] | None = None,
    interceptors: tuple[grpc.aio.ClientInterceptor, ...] = (),
) -> AsyncContextManager[AsyncRoutingChannel]
```

The four exported channel constructors freeze these exact signatures:

```python
RoutingChannel(
    cassette: Cassette, target: str, *,
    credentials: grpc.ChannelCredentials | None = None,
    options: Sequence[ChannelOption] | None = None,
    transport_factory: SyncTransportFactory | None = None,
    invocation_preparer: SyncLiveInvocationPreparer | None = None,
    completion_hook: SyncCompletionHook | None = None,
) -> None

AsyncRoutingChannel(
    cassette: Cassette, target: str, *,
    credentials: grpc.ChannelCredentials | None = None,
    options: Sequence[ChannelOption] | None = None,
    interceptors: tuple[grpc.aio.ClientInterceptor, ...] = (),
    transport_factory: AsyncTransportFactory | None = None,
    invocation_preparer: AsyncLiveInvocationPreparer | None = None,
    completion_hook: AsyncCompletionHook | None = None,
) -> None

RecordingChannel(  # compatibility facade; .channel is its RoutingChannel
    cassette: Cassette, target: str, *,
    credentials: grpc.ChannelCredentials | None = None,
    options: Sequence[ChannelOption] | None = None,
    transport_factory: SyncTransportFactory | None = None,
    invocation_preparer: SyncLiveInvocationPreparer | None = None,
    completion_hook: SyncCompletionHook | None = None,
) -> None

AsyncRecordingChannel(  # compatibility facade; .channel is AsyncRoutingChannel
    cassette: Cassette, target: str, *,
    credentials: grpc.ChannelCredentials | None = None,
    options: Sequence[ChannelOption] | None = None,
    interceptors: tuple[grpc.aio.ClientInterceptor, ...] = (),
    transport_factory: AsyncTransportFactory | None = None,
    invocation_preparer: AsyncLiveInvocationPreparer | None = None,
    completion_hook: AsyncCompletionHook | None = None,
) -> None
```

Attachment-local constructors accept no `record_mode`, `config`, or `match_on`;
session policy comes only from `cassette.config`. The compatibility facades retain
their existing `cassette, target, *, credentials, options` prefix and add only the
attachment-local runtime keywords above.

A cassette session is `UNBOUND`, `SYNC`, or `AIO`. The first channel attachment
binds the runtime stack; a mixed-stack attachment is a `ConfigurationError` before
calls or state changes. An aio session is additionally bound to the construction
event loop, and attachment from a second loop is rejected. Same-stack facades may
share the session; terminal close is session-wide and cancels/finalizes every
attached call before closing all owned transports.

Synchronous `Cassette.save()` and a synchronous `Cassette.__exit__` may terminate
only an `UNBOUND` or `SYNC` session. On an open `AIO` session, `save()` and a
normal sync exit raise a constant `ConfigurationError` **before** sealing or
cancelling anything and direct the caller to `await AsyncRoutingChannel.close()` /
exit `aio_recorded_channel`. If a body exception is already escaping a prematurely
nested sync cassette context, `__exit__` does not mask it or transition state; the
outer async channel receives that exception and performs awaited abort/cleanup.
`Cassette` does not add a misleading blocking `asave` shim in 0.2. After awaited
aio close has reached `COMMITTED` or `ABORTED`, an enclosing synchronous cassette
exit is a nonblocking observation of the stored result. Correct nesting is
therefore `with Cassette(...)` outside `async with AsyncRecordingChannel(...)`, so
the inner async owner exits first.

The existing `match_on=` keyword remains as a deprecated alias for
`config.matcher`. `config=None` means construct `SessionConfig()`; specifying
`match_on` then replaces only that default config's matcher. Supplying an explicit
`config` together with `match_on` is a configuration error. The alias does not
weaken profile separation: an old v1 `Matcher` supplied to the strict default is
rejected with instructions to use
`SessionConfig(profile=LEGACY_RAW, matcher=...)`. `Cassette` remains the
storage/session facade with exact constructor
`Cassette(path: str | Path, record_mode: RecordMode = RecordMode.NEW_EPISODES,
match_on: StrictMatcher | Matcher | None = None, *, config: SessionConfig | None =
None)`. `use_cassette` mirrors those parameters and returns
`ContextManager[Cassette]`. The `grpcvcr` pytest marker retains its existing
`match_on=` keyword and adds `config=` with the same mutual exclusion and profile
validation; positional marker forms are unchanged. The pytest plugin
adds `grpcvcr_config`, and its channel factories pass that object unchanged.

## Channel and call conformance

The routing implementation must cover the documented public surfaces for all four
RPC shapes. Verified against grpcio 1.76.0 `__abstractmethods__`. Phases 3 and 4
satisfy the unary rows; Phase 5 satisfies the streaming rows.

| Shape | Sync surface | `grpc.aio` surface |
| --- | --- | --- |
| Unary-unary | `__call__`, `with_call`, `future`; the returned object implements `grpc.Call` and `grpc.Future` | `__call__` returns a `grpc.aio.UnaryUnaryCall` |
| Unary-stream | `__call__` only (`grpc.UnaryStreamMultiCallable` has no `with_call`/`future`); the returned object implements `grpc.Call`, `grpc.Future`, and `grpc.RpcContext`, is an iterator, and is itself a `grpc.RpcError` subclass | `__call__` returns a `grpc.aio.UnaryStreamCall` |
| Stream-unary | `__call__`, `with_call`, `future` — the same three members as unary-unary. `__call__` blocks and returns the deserialized response; `with_call` returns `(response, call)`; `future` returns a call object implementing `grpc.Call` and `grpc.Future` | `__call__` returns a `grpc.aio.StreamUnaryCall` |
| Stream-stream | `__call__` only (`grpc.StreamStreamMultiCallable` has no `with_call`/`future`); the returned object has the same bases as the unary-stream one and is an iterator | `__call__` returns a `grpc.aio.StreamStreamCall` |

Multicallable `__call__` signatures differ by request arity and by stack:

- Unary-request shapes accept `(request, timeout=None, metadata=None,
  credentials=None, wait_for_ready=None, compression=None)`; the aio variants
  are keyword-only after `request`.
- Stream-request shapes accept `(request_iterator, timeout=None, metadata=None,
  credentials=None, wait_for_ready=None, compression=None)`. The parameter is
  named `request_iterator` and is matched by that keyword. On aio it defaults to
  `None`, which selects reader/writer style, and the remaining parameters are
  **not** keyword-only — grpcio 1.76.0 declares no `*` on
  `grpc.aio.StreamUnaryMultiCallable.__call__` or
  `grpc.aio.StreamStreamMultiCallable.__call__`. On sync it is required.
- Aio stream-request shapes accept either a synchronous `Iterable` or an
  `AsyncIterable`; grpcio's own consumer branches on `__aiter__`. grpcvcr
  accepts both.

All four channel factory methods accept
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
- Sync streaming responses (`unary_stream` and `stream_stream`): `__iter__`,
  `__next__`, and `next()`.
- Sync request streaming (`stream_unary` and `stream_stream`): the only request
  API is the iterator passed to `__call__`/`with_call`/`future`. gRPC starts one
  daemon consumption thread per call at invocation
  (`grpc._channel._consume_request_iterator`), serializes on that thread, and
  emits half-close when the iterator raises `StopIteration`. There is no sync
  `write()`/`done_writing()`; grpcvcr does not add one.
- gRPC's sync call objects also subclass `grpc.RpcError`, so
  `isinstance(call, grpc.RpcError)` is `True` on a real channel. The replayed
  sync call therefore multiply inherits the public `grpc.RpcError`, `grpc.Call`,
  `grpc.Future`, and `grpc.RpcContext` bases; no private rendezvous class is
  needed. On terminal stream failure, iteration raises that same call object.
- Aio `grpc.aio.Call`: `initial_metadata`, `trailing_metadata`, `code`,
  `details`, `cancel`, `cancelled`, `done`, `time_remaining`,
  `add_done_callback`, `wait_for_connection`.
- Aio unary-unary: `__await__`. Aio unary-stream: `__aiter__` and `read()`.
  Aio stream-unary: `__await__`, `write()`, and `done_writing()`. Aio
  stream-stream: `__aiter__`, `read()`, `write()`, and `done_writing()`.
  `write` and `done_writing` are coroutines; `done_writing` is idempotent.
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

Connectivity routing is deterministic:

| Session state | Initial connectivity behavior | Materialization/transition |
| --- | --- | --- |
| `NONE` or existing-cassette `ONCE` | Virtual `READY`; `channel_ready()` returns, `get_state()` is `READY`, and sync subscribers receive one initial `READY`. | Never materializes transport. |
| `NEW_EPISODES` before a live miss | The same virtual `READY`, including readiness calls made before any invocation. | Replay hits leave it virtual. The first routed live miss materializes transport, atomically switches connectivity to the real channel's current enum, and delegates all future waits/subscriptions. |
| `ALL` or absent-cassette `ONCE` | No virtual state. `get_state(try_to_connect=False)` reports the lazy real channel's initial `IDLE`; a readiness request or first call may materialize it. | Delegates from first materialization. |

At the `NEW_EPISODES` handoff, existing subscribers are transferred exactly once.
They are notified only if the real enum differs from their last virtual `READY`;
a virtual-`READY` to real-`READY` handoff alone does not satisfy
`wait_for_state_change(READY)`. `unsubscribe` removes the callback on whichever
side currently owns it. These states are a documented routing abstraction, not a
claim that a real channel produced the pre-handoff history, and the conformance
matrix tests every mode before and after materialization. When multiple same-stack
routing channels share a cassette session, each channel owns its connectivity
handoff and transport slot independently.

Aio streaming-response consumption — `unary_stream` and `stream_stream` alike,
since both inherit `_StreamResponseMixin` — matches the documented `grpc.aio` surface,
with any intentional timing deviation called out below:

- `read()` returns `grpc.aio.EOF` at end of stream, never `None`.
- After OK termination, `read()` returns `grpc.aio.EOF` idempotently.
- After non-OK termination, every `read()` raises an `AioRpcError` with equivalent
  status, details, and metadata. Object identity is not promised; grpcio versions
  may construct a fresh error for each failed read.
- After local cancellation, `read()` raises `asyncio.CancelledError`.
- `__aiter__` returns one cached async generator per call; repeated `async for`
  over a single call resumes, it does not restart. The current
  `_AsyncFakeStreamingCall.__aiter__` returns a fresh generator each time, so a
  second loop replays from message zero — that is a live bug, fixed in Phase 3.
- Mixing `read()` and `__aiter__` on one call raises `grpc.aio.UsageError` with
  gRPC's message, `The iterator and read/write APIs may not be mixed on a single
  RPC.` Response style is first-use-wins and applies to `unary_stream` and
  `stream_stream`.
- Request style is **not** first-use-wins: it is fixed at `__call__`. Passing a
  `request_iterator` selects iterator style, and the **first** `write()` or
  `done_writing()` on that call raises the same `UsageError` — nothing was
  "mixed". Passing `request_iterator=None` selects reader/writer style, and the
  call has no iterator API to conflict with. grpcvcr reproduces both directions
  including the asymmetry: the request-side error fires on first use, the
  response-side error on second, differing use.

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
wall-clock timing. `protobuf-safe` returns sanitized messages plus authorized
metadata/status details; only `keep` fields are value-faithful, and default-drop
fields replay as absent/default. `legacy-raw` preserves codec bytes. Neither
profile claims HTTP/2 flow-control fidelity. Unary-stream playback stays
pull-based: messages materialize on iteration, not on call construction.

## Async live-call lifecycle

A permitted aio live miss begins at multicallable invocation, matching real
`grpc.aio` behavior.

`__call__` is a synchronous method that returns an awaitable call object,
matching `grpc.aio`. It captures an absolute monotonic deadline, constructs a
shape-specific `DeferredLiveCall`, and schedules its controller task with
`loop.create_task` on the loop captured at channel construction. `grpc.aio`
channels are loop-affine: invocation from another loop or a thread with no
running loop raises `grpc.aio.UsageError`. grpcvcr does not claim unsupported
cross-thread tolerance or hide it behind `run_coroutine_threadsafe`.

The controller task never propagates an exception as its own result. Every
call-visible startup/delegate failure is routed into `logical_terminal`; hook
failure is reduced to a safe finalization result for close. The task carries a done
callback that consumes any residual internal exception, so an abandoned call
produces no "Task exception was never retrieved" noise. RPC errors surface when
the application awaits, reads, or queries the call; finalization errors surface at
the documented close boundary.

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

Request-stream state is orthogonal to the call states above. A stream-request
call additionally holds:

- `request_style`, fixed at `__call__` to iterator or reader/writer as described
  under [Channel and call conformance](#channel-and-call-conformance).
- `half_closed: bool`, set by iterator exhaustion or by `done_writing()`. It is
  **not** a call state: a half-closed `stream_stream` call stays `DELEGATED`
  until its response side terminates, and a call may half-close before
  delegation. Modelling half-close as an edge into `DONE` would break
  `stream_stream` entirely.
- A controller-owned request-pump task, present only in iterator style. It is
  created at `__call__` and pulls the first message **before** `delegate_ready`
  resolves, matching `grpc.aio._call._StreamRequestMixin`, where the poller task
  is created synchronously in `__init__` and the first `_write` blocks on
  `_metadata_sent`. Exactly one message is therefore pulled from the
  application's generator before the RPC has sent anything; deferring that pull
  until after delegation would change observable generator side-effect order,
  which Phase 5's exit criteria forbid. It never runs more than one
  pulled-but-unwritten message ahead, and after delegation it is paced by the
  delegate's write backpressure.

Iterator exhaustion half-closes; an iterator that raises is handled under
[Cancellation and incomplete interactions](#cancellation-and-incomplete-interactions).
The pump is cancelled by any terminal transition, and `finalization_done` waits
for it to settle in addition to the existing barriers.

The call owns four separate synchronization points and one controller owns every
context-bearing transition:

- `delegate_ready: Future[RealCall | NO_DELEGATE]` resolves with the real call or
  an internal non-exception sentinel when terminal state wins before publication.
- `preparation_settled: Future[LiveInvocation | NO_CONTEXT]` resolves only when
  the preparer has returned a context or has definitively failed/cancelled without
  one. `NO_CONTEXT` is an internal sentinel, never application data.
- `logical_terminal: Future[SafeOutcome]` resolves exactly once at logical RPC
  completion or local cancellation. Public status methods, done callbacks, and
  active-call removal use this future and may safely be called from a finalizer.
- `finalization_done: Future[SafeFinalizationResult]` resolves only after both
  `logical_terminal` and `preparation_settled`, and after the completion hook, if
  any, returns or raises. The result contains only success or a safe ordinal/reason
  failure report. Persistence and channel close wait for this future.

State invariants:

- The shielded controller, not the canceller or deadline timer, owns preparer
  settlement, delegate publication/draining, and context finalization. Cancellation
  may resolve `logical_terminal` immediately and requests cancellation of a
  per-call async preparer, but it does not cancel the controller. If trusted
  preparer code ignores cancellation and later returns `LiveInvocation`, the
  controller observes the already terminal call, starts no real RPC, invokes the
  completion hook exactly once with the cancellation `SafeOutcome`, and only then
  resolves `finalization_done`. If preparation raises or accepts cancellation, it
  publishes `NO_CONTEXT` and no completion hook runs.
- Publication of the delegated call and observation of the cancel flag occur
  inside one critical section on the controller task. The controller acquires the
  state lock, re-reads the state, and either transitions `STARTING → DELEGATED`
  and publishes, or — if the state is already `CANCELLED` or `DONE` — cancels and
  drains the just-created real call and publishes
  nothing. A real call is never created and then forgotten.
- `logical_terminal` is resolved exactly once, by whichever of controller,
  canceller, delegated completion, or deadline timer wins the lock.
- Context finalization starts exactly once after both barriers resolve, on every
  path including `STARTING → DONE` and `STARTING → CANCELLED`. The controller
  releases the opaque context and resolves `finalization_done` in `finally`; raw
  hook exceptions are chain-cleared, released, and never stored in a future.

Operation behavior is defined as follows:

- `read`, `__anext__`, `initial_metadata`, and `wait_for_connection` race
  `delegate_ready` against `logical_terminal` and act on whichever resolves
  first: on `delegate_ready` they delegate; on `logical_terminal` they raise the
  terminal outcome (`AioRpcError` for a status, `asyncio.CancelledError` for local
  cancellation) without ever touching a delegate.
- On any terminal transition that skips delegation, `delegate_ready` resolves with
  `NO_DELEGATE` in the same critical section as `logical_terminal`. An operation
  observing the sentinel reads/raises the safe outcome from `logical_terminal`.
  No unobserved internal future stores an exception, and neither future is left
  pending after a terminal transition.
- `wait_for_connection()` returns as soon as the call is delegated and the
  delegated call reports connection, and raises the terminal error if the call
  terminated first. It never waits on `logical_terminal` for a healthy call.
- Unary await delegates to the real unary call.
- `write(request)` **awaits** `delegate_ready` rather than racing it, matching
  `grpc.aio._interceptor._InterceptedStreamRequestMixin.write`. On delegation it
  writes through. If the call is already terminal — whether by status or by local
  cancellation — it raises `asyncio.InvalidStateError` with gRPC's
  `RPC already finished.`; after half-close it raises `asyncio.InvalidStateError`
  with `RPC is half closed after calling "done_writing".`. It never raises
  `AioRpcError` or `asyncio.CancelledError`, because gRPC does not.
- `done_writing()` awaits the same barrier, is idempotent, and is a no-op once
  the call is terminal. It raises `asyncio.InvalidStateError` only when the
  barrier itself was cancelled.
- Writes pending before delegation are bounded at one outstanding message, as
  gRPC's own `maxsize=1` proxy queue is, and a pending write is released — not
  left hanging — when the call reaches `logical_terminal`.
- `code`, `details`, and trailing metadata wait for the logical terminal state.
  `initial_metadata` does not.
- Done callbacks register against `logical_terminal` and receive the wrapper call.
- `done`, `cancelled`, and `time_remaining` are nonmaterializing state queries.

Each attached routing channel has one session-owned single-flight transport slot.
Per-call cancellation never cancels initialization required by another call on
that channel; different attached targets/factories do not accidentally share a
transport. The invocation preparer runs once per live call.

Channel close follows `grpc.aio` grace/cancellation semantics, then adds the
documented grpcvcr finalization and transaction work:

- `close(grace=None)` refuses new invocations, then cancels every active call
  immediately. It does not wait for natural logical completion.
- `close(grace=N)` refuses new invocations, waits up to N seconds for active
  calls to reach `logical_terminal`, then cancels whatever remains.
- On a normal context exit, `__aexit__` is `await self.close(None)`. On an
  exceptional exit it passes the exception into the abort-close path, cancels and
  finalizes calls, preserves the body exception, and does not commit while the
  session is still `OPEN`.
- Close attempts are shared. The first caller creates and owns `close_attempt`;
  every concurrent `close()` awaits that same future and observes the same safe
  result or failure. After a committed/aborted attempt, repeated close returns the
  stored result (or reconstructs the same highest-precedence chain-cleared close error) only
  after that attempt is complete. It never returns merely because another close
  is in progress.
- If an attempt ends with storage `commit_state="not_committed"` or `"unknown"`,
  all callers already joined to it receive that failure. A later, non-concurrent
  `close()` creates a new shared **commit-only** attempt from the retained canonical
  bytes and dirty state. It never repeats cancellation, hooks, or interaction
  finalization. Concurrent callers of that retry share its future too.

Close waits for active calls themselves only under `close(grace=N)`, and only for
that grace period. In every case it then waits for opaque-context finalization of
the calls it cancelled. Completion hooks are trusted code and finalization is not
falsely advertised as hard-bounded: an async hook that never returns can block
close just as a sync hook can. A watchdog may emit a constant-field `hook_slow`
event, but does not detach code that still owns the opaque context. A failed hook
emits `hook_failed`, resolves a safe failed finalization result, and discards that
interaction; neither event is reported recursively through the completion hook.
After every `finalization_done` and transport barrier settles, close closes owned
transports, commits other valid interactions, and then raises the applicable safe
close error. A storage error takes precedence for that attempt and retains safe
transport/finalization reports, which surface by the precedence above after a
later successful commit-only retry.

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
- Strict wrappers never re-raise a delegated transport exception object. They
  derive `SafeOutcome`, release the original, construct a safe public error, and
  raise it through `_raise_strict`; this prevents raw transport diagnostics or an
  existing exception chain from escaping.
- An independent deadline timer resolves `logical_terminal` if it fires while
  the call is still `NEW` or `STARTING`, using the same critical section as
  cancellation.

Replay uses the same absolute deadline. A non-positive timeout fails before any
recorded message is exposed. Unary replay reaches its recorded terminal outcome
on first await/result. Unary-stream replay remains pull-based: the deadline stays
active until the recorded terminal event is reached, so expiry before the first
read or between messages produces local `DEADLINE_EXCEEDED`; expiry after the
recorded terminal event has no effect. Playback adds no sleeps or recorded
latency. Tests use a fake monotonic clock rather than wall-clock timing.

This is an explicit replay-timing deviation, not a claim of exact background gRPC
completion: a real server stream may already have reached terminal while the
application pauses between reads, whereas grpcvcr does not materialize unread
cassette events in the background. The chosen policy charges that reader think
time to the replay deadline so lazy messages cannot be exposed after the caller's
budget. Public documentation and conformance tests name the deviation and do not
describe “no artificial latency” as deadline equivalence.

## Sync live-call lifecycle

Sync live misses use the same `NEW → STARTING → DELEGATED → DONE` /
`CANCELLED` states and the same separation among delegate readiness, logical
termination, and finalization. Synchronization uses a lock and condition rather
than asyncio futures.

- `unary_unary.future()`, `stream_unary.future()`, `unary_stream.__call__()`,
  and `stream_stream.__call__()` return deferred public call objects immediately
  and start their controller on a bounded, session-owned executor.
  `unary_unary.__call__()` and `stream_unary.__call__()` are defined as
  `future(...).result()`; `with_call()` waits for that same future and returns
  the `(response, call)` **pair** its shape's ABC requires —
  `_end_unary_response_blocking` returns `state.response, rendezvous`, so
  returning a bare call object would break every
  `response, call = stub.M.with_call(req)` caller.
- Sync request-stream consumption does **not** run on the bounded controller
  executor. Each `stream_unary`/`stream_stream` call gets its own daemon pump
  thread, as `grpc._channel._consume_request_iterator` does, because that thread
  blocks on trusted user code and on write backpressure. Sharing the bounded
  pool between controllers and pumps deadlocks once concurrent streaming calls
  exceed half the pool — controllers wait for messages a queued pump would have
  produced, and it hangs rather than erroring. Close joins every pump thread it
  cancelled.
- Each attached sync routing channel's transport factory is a session-owned single-flight
  `concurrent.futures.Future`. Cancelling one call never cancels shared transport
  initialization. A transport-factory failure completes every waiting call with
  a safe local error and allows a later invocation to retry with a new
  single-flight generation.
- Deadline, delegate publication, cancellation, callback registration, and
  finalization use one state lock. The `cancel()`/delegate race follows the async
  invariant: a call created after cancellation is cancelled and drained before
  publication.
- A condition-backed `preparation_settled` barrier has the same `LiveInvocation |
  NO_CONTEXT` meaning as aio. Cancelling a running thread cannot kill trusted
  preparer code; the controller waits for it, finalizes a late context exactly
  once with the already selected outcome, and close waits for the safe finalizer
  result.
- `result(timeout)` and metadata/status methods implement their documented
  per-operation wait timeout independently of the RPC deadline.
- The sync completion hook is trusted synchronous code and may block `close()`;
  Python cannot safely terminate it. Unlike the async hook, it has no fake
  timeout guarantee. This behavior is explicit in the hook contract.
- Sync `close()` refuses new calls, cancels active calls, waits for their
  finalizers and transport settlement, closes owned transports, shuts down the
  controller executor, and commits according to the transaction table. A
  condition-backed shared close attempt gives every
  concurrent caller the same completion/failure, and later storage retries are
  commit-only under the same rule as aio.

## Transport and per-call authentication

Transport creation and per-call authorization are separate extension points:

```python
SyncTransportFactory = Callable[[], grpc.Channel]

AsyncTransportFactory = Callable[
    [], grpc.aio.Channel | Awaitable[grpc.aio.Channel]
]

SyncLiveInvocationPreparer = Callable[
    [CallDescriptor], LiveInvocation
]

AsyncLiveInvocationPreparer = Callable[
    [CallDescriptor], Awaitable[LiveInvocation]
]

SyncCompletionHook = Callable[[object | None, SafeOutcome], None]
AsyncCompletionHook = Callable[[object | None, SafeOutcome], Awaitable[None]]
```

A completion/failure hook receives the opaque context and a `SafeOutcome`. This
lets an application own credential generation, invalidation, and
`UNAUTHENTICATED` retry without obtaining a credential before cassette routing.

On a cassette hit, none of the transport, preparation, or completion hooks are
invoked.

If no transport factory is supplied, grpcvcr builds a lazy default factory from
`target`, `credentials`, and `options`. A custom factory controls channel
credentials and options; supplying it together with `credentials` or `options` is
a configuration error rather than silently ignoring either source. Returning from
either factory transfers exclusive lifecycle ownership of that channel to the
grpcvcr session. Borrowed/shared transports are not supported in 0.2.

Every attached routing channel has a session-owned `transport_settled` barrier.
Closing while a sync/async factory is running requests cancellation where possible and
waits for settlement; trusted factory code may ignore cancellation and is not
advertised as hard-bounded. A factory that later returns still transfers ownership
and its channel is closed without starting a call. A factory that fails or accepts
cancellation settles as `NO_TRANSPORT`. No late channel can escape session
ownership.

Terminal close ordering is: refuse calls; apply grace/cancellation; settle every
preparer and completion hook; settle any in-progress transport factory; close each
owned transport exactly once in attachment order (`close()` for sync, awaited
`close(None)` for aio); shut down the sync controller executor; then commit valid
interactions. Transport close is trusted and may block. A close exception emits
`transport_close_failed`, is chain-cleared and released, does not discard already
complete interactions, and surfaces as `TransportCloseError` only after commit.
Failure precedence for one close attempt is storage error, then
`TransportCloseError`, then `FinalizationError`; every lower-priority safe event
still emits, and the shared/repeated close result preserves that precedence.

A completion
hook runs exactly once only after its preparer successfully returned a
`LiveInvocation`; failures before a context exists emit `startup_failed`, settle
as `NO_CONTEXT`, and have nothing to finalize.

Supplying a completion hook without an invocation preparer is a construction-time
`ConfigurationError`; grpcvcr does not fabricate application context. A preparer
without a completion hook is allowed: its additional metadata/credentials are
used, and its context is released at terminal settlement with an immediate
successful finalization result.

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
- On a cassette hit, `credentials`, `wait_for_ready`, and `compression` are
  accepted and ignored. `timeout` drives the replay deadline policy, and selected
  sanitized metadata remains part of matching.

Strict transport factories must return unintercepted/base channels. Request-
mutating live interceptors are unsupported in strict 0.2. A routing-layer live
request transform may run before serialization when the outgoing request must
change. Projection-only transforms never affect live bytes. Metadata-only live
behavior belongs in the invocation preparer.

## Request and response data separation

Strict mode produces two separate artifacts per request message.

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

This narrows but does not close the frame-locals channel. `__tracebackhide__` is
presentation hygiene, not a security boundary. The library makes no
memory-scrubbing claim: `bytes` are immutable and unzeroable in CPython, and may
remain reachable through a core dump, `sys.settrace`, `pytest --pdb`, or a
debugger until collected. Immutable bytes are generally not tracked by cyclic
GC, so `gc.get_objects()` is not an absence oracle and is not used as one.
Running a strict-mode recording under a post-mortem debugger or locals-rendering
crash reporter is outside the guarantee.

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

For the request-streaming shapes this pipeline runs once per client message, at
the rate the call consumes the iterator. A failure on message *k* aborts the
interaction and sends nothing further, but messages `0..k-1` were already
delegated and cannot be retracted — the same caveat stated for streamed responses
below. The interaction is discarded and never commits.

Before constructing a protobuf message, a descriptor-aware wire preflight checks
the byte limit, field-tag/node count, length bounds, and message nesting. It skips
unknown length-delimited fields as opaque bytes and recursively preflights known
message fields and unpacked `Any` values. This prevents a small wire payload with
millions of empty repeated messages from allocating an unbounded object graph.

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

### Deterministic serialization

`SerializeToString(deterministic=True)` guarantees map-entry ordering within one
protobuf build and nothing more; it is not canonical across implementations,
versions, or languages. `protobuf-python-deterministic-v1` therefore names a
Python-runtime form scoped to the supported compatibility matrix, of which the
protobuf flag is one input:

| Requirement | Rule |
| --- | --- |
| Map ordering | `deterministic=True`, required, propagated to every nested message. |
| Field ordering | Ascending field number, including extensions, as emitted by every runtime in the tested matrix. |
| Unknown fields | None present. The discard step must precede serialization; unknown-field order is wire-order and is not deterministic. |
| Float / double NaN | Canonicalized to quiet NaN `0x7ff8000000000000` / `0x7fc00000` before serialization. Non-canonical NaN payload bits are not preserved. |
| Infinities, `-0.0` | Preserved verbatim; both implementations serialize `-0.0` as present. |
| Fixpoint | After serialization, reparse with the registered descriptor, assert zero unknown fields, reserialize, and assert byte equality. A failure aborts the interaction. |

Matching compares the deterministic bytes. They are a cache of message
identity, not the definition of it: a cassette whose payload bytes fail the
fixpoint check on the reader's protobuf build is rejected with
`NonCanonicalPayloadError` and a re-record instruction, rather than silently
failing to match. CI runs the byte-equality suite against both `upb` and
`PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python`, and against the minimum and
current protobuf floors.

### Responses

Responses use the same separation. The wrapper observes raw response bytes,
parses the registered response FQN, calls the original deserializer once for the
application value, and sanitizes a separate parsed clone before exposing that
message. On success the application receives the original value; only a
`PersistableEvent` reaches the cassette builder. A response-side parse,
sanitization, or validation failure raises a chain-cleared safe error, discards
the interaction, and never exposes the failing message. Earlier streaming
messages may already have been observed and cannot be retracted. On later strict
replay the application receives the sanitized
persisted message, so policies must `keep` every response field whose value is
part of the test contract. This intentional security/fidelity trade-off is
documented in the public profile guide.

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

Application-provided interceptors, transforms, invocation preparers, hooks, and
event sinks are trusted code outside the exfiltration guarantee. The library
cannot prevent such a callback from logging or transmitting its own input.
Projection, preparer, and completion-hook failures discard the affected
interaction as defined by the transaction table and cause no library logging of
callback input. Projection/preparer failures surface immediately as a constant
chain-cleared error. Hook failures surface as chain-cleared `FinalizationError`
from normal close after other valid interactions commit; an already escaping
application exception has precedence. Event-sink failure is the deliberate
exception: it is swallowed and does not change an otherwise valid state
transition or commit.

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
  request/response and explicitly authorized packed FQNs, event count, event
  ordering, payload byte lengths, metadata key presence, and status codes. These
  are not redacted and can identify a request. Callers for whom types, message
  counts, or payload sizes are themselves sensitive should not commit cassettes
  to a shared repository.

### Safe observability

`SafeEvent` and `SafeEventSink` have the exact frozen constructors shown under
[Normative public configuration](#normative-public-configuration). Every event
populates `kind`, a session-monotonic `sequence`, `profile_id`, and
`profile_digest`; fields not named below are `None`.

| Kind | Emission point and cardinality | Additional populated fields |
| --- | --- | --- |
| `replay_hit` | Once per successful atomic reservation, before returning the replay call. | method, shape, request/response FQN, ordinal, `reserved` consumption state, safe selector digest |
| `replay_miss` | Once per lookup with no selectable candidate, before record-mode routing. | method, shape, request FQN, safe selector digest, reason |
| `live_started` | Once immediately after a real call object is created, whether it is later published, cancelled, or drained. | method, shape, request/response FQN, generation ordinal |
| `startup_failed` | Once when transport, preparation, or delegation fails before a real call exists. | method, shape, request/response FQN, generation ordinal, stage reason |
| `hook_slow` | At most once per finalizer when its watchdog threshold first elapses. | method, shape, ordinal, `hook_watchdog` reason |
| `hook_failed` | Once when a completion hook raises, before discarding its interaction. | method, shape, ordinal, `hook_exception` reason |
| `interaction_discarded` | Once when a builder becomes permanently ineligible to commit. | method, shape, ordinal, status code when safe, discard reason; configured limit name/value when the reason is `limit_exceeded` |
| `transport_close_failed` | Once per owned transport whose exactly-once close raises, before the commit attempt. | `transport_close` reason |
| `source_conflict` | Once per commit attempt whose final source hash differs, immediately before `commit_failed`. | caller path, `expected_digest`, `observed_digest`, `not_committed` |
| `commit_succeeded` | Once per successful replace-and-durability attempt. | caller path, resulting source digest, `committed` |
| `commit_failed` | Once per failed commit attempt, after `source_conflict` when applicable. | caller path, reason, `not_committed` or `unknown` |

Storage stages map exhaustively to safe reasons: destination/symlink/non-regular
rejection → `path_rejected`; sidecar open/create/security setup → `lock_open`;
nonblocking contention → `writer_busy`; temporary creation → `temp_create`;
temporary write/disk-full → `temp_write`; model encoding → `serialize`; artifact
validation → `validate`; mode/DACL application → `permission_apply`; flush or file
`fsync` → `flush`; final hash mismatch → `source_changed`; rename → `replace`;
directory sync → `directory_sync`; and a cleanup failure with no earlier primary
failure → `cleanup`. A cleanup failure secondary to another stage never replaces
the primary reason. `storage_io` is the safe fallback for an OS failure point not
otherwise classifiable; adding a new injected stage requires assigning it a
specific reason or explicitly testing this fallback before the public enum freezes.

Within a session, assignment and delivery are serialized in `sequence` order.
Events caused by one interaction obey the table order; events from concurrent
interactions may interleave only as their sequence numbers show. Delivery is
synchronous and inline, after releasing lifecycle and storage locks. Sink latency
therefore counts as trusted callback latency: it can delay later work or allow a
subsequently checked deadline to expire, but cannot roll back the state transition
that caused the event. A sink exception is swallowed without logging and never
recursively emits another event. Same-session API re-entry from the sink is
rejected with a constant `ConfigurationError`; the attempted nested event is
suppressed. Exact event-sequence tests cover every row, failures, concurrency, and
re-entry. Pilot playback requires zero `live_started` events in addition to
transport/DNS/auth paths being disabled.

### Redaction policy

The strict redactor is deny-by-default. A field is persisted only if a policy
rule names it; every other field is cleared before serialization. Policy rules
are expressed as fully qualified field paths against the registered FQN.

The verbs are category-sensitive; `keep` is exact for leaf/unit values but only a
container authorization for a regular message or `Any`. The complete transform is:

| Field category | `keep` | `keep_shape` | absent / `drop` |
| --- | --- | --- | --- |
| Presence-aware singular scalar, enum, string, or bytes (proto2, proto3 `optional`, or selected real-oneof member) | Preserve the exact value and presence. | Preserve presence with a type-fixed placeholder: numeric zero, the enum descriptor's first declared value, `false`, fixed non-empty string sentinel, or fixed non-empty bytes constant. | `ClearField`. |
| Implicit-presence proto3 singular scalar, enum, string, or bytes | Preserve the exact value; the API has no presence bit. | Preserve **serialized non-defaultness**, not presence: if the original would serialize, replace it with fixed non-default `1`/`1.0`, `true`, numeric enum value `1` (open proto3 enums permit an unnamed value), or the fixed non-empty string/bytes constant. An original default remains absent/default. | Reset to the language default. |
| Repeated scalar, enum, string, or bytes | Preserve all values and order. | Preserve element count and order, replacing every element with its type-fixed placeholder. | Clear the list. |
| Singular regular message, including wrappers, `Timestamp`, `Duration`, and `FieldMask` | Authorize the container, then recurse only through explicitly ruled descendants. If no descendant remains, clear the field. No future field is implicitly kept. | Preserve only fixed empty-message presence; descendants are not inspected. | Clear the field. |
| Repeated regular message | Preserve element count/order and recursively redact each element; only explicitly ruled descendants survive. | Preserve element count/order as fixed empty messages. | Clear the list. |
| `google.protobuf.Empty` | Preserve its exact presence (and repeated count/order). | Same fixed empty presence/count; there is no value to reveal. | Clear it. |
| Map | Preserve the entire map exactly; keys are runtime data and there is no per-entry policy. | Configuration error. Synthetic keys would invent semantics, while constant keys can collapse entries. | Clear the map. |
| `Struct`, `Value`, or `ListValue` | Preserve the selected value as an indivisible unit. | Configuration error. | Clear the value. |
| `Any` | Apply the separate matrix below. | Apply the separate matrix below. | Clear the entire `Any`. |
| Unknown field | Never allowed. | Never allowed. | Discard before serialization. |

A real oneof is first cleared with `ClearField(oneof_name)` unless its **selected**
member's resolved action authorizes it; grpcvcr never redacts by assigning a
sibling. Proto2 presence and proto3 `optional` use the presence-aware row above —
a drop uses `ClearField(field_name)`, never assignment of a default that would
retain presence. Ordinary proto3 scalars use the separate implicit-presence row;
the plan makes no nonexistent presence guarantee for them. Extensions are denied
unless a rule names the full extension name;
unresolved extensions arrive as unknown fields and are discarded.

Policy resolution is closed over the descriptor: a rule naming a path absent from
the registered descriptor is a configuration error, not a no-op.

The redactor must handle proto2 presence, proto3 optional fields, oneofs,
extensions, nested/repeated messages, maps, wrappers, `Any`, `Struct`, `Value`,
`Timestamp`, `Duration`, `FieldMask`, `Empty`, `NullValue`, enums, bytes, numeric
edge cases, NaN/infinity, and unknown fields. `FieldMask` in particular persists
schema paths that a `keep` on the containing message would silently expose.

Replacement values are constant-shape, not length-preserving: a redacted `string`
becomes a fixed non-empty sentinel and `bytes` a fixed non-empty constant.
Presence-aware numerics use zero and enums use the descriptor's first declared
value; proto2 enums need not declare numeric zero, so assigning zero
unconditionally is forbidden. Repeated numerics use zero and repeated enums use
the first declared value because repeated cardinality preserves their shape.
Implicit-presence proto3 values instead use the non-default constants in the table
when the original would serialize; this includes bit-distinct `-0.0`. Length-preserving redaction is forbidden — the persisted length and the
resulting deterministic-serialization length are themselves a disclosure, and
both reach the match key.

Map keys are never individually redacted. A map is either kept as one exact unit
or cleared; `keep_shape` is rejected while validating the profile, before calls.

### `Any` handling

`Any.value` is opaque bytes; no ordinary traversal, discard, or field policy
reaches inside it. An `Any` carrying a message with an unknown field containing a
bearer token would otherwise land in the cassette verbatim. The original
`type_url` is also untrusted protobuf data and is never persisted verbatim; its
arbitrary prefix could itself contain a secret. Behavior is the following
complete matrix:

| Enclosing rule | Packed type / packed rules | Result |
| --- | --- | --- |
| absent or `drop` | any | Clear the entire `Any`, including `type_url`. |
| `keep_shape` | any | Persist the fixed constant `type.googleapis.com/google.protobuf.Empty` with an empty value. |
| `keep` | resolvable, with packed-type rules | Unpack, recursively redact/discard/validate, and repack with canonical `type.googleapis.com/<FQN>`. |
| `keep` | absent or no packed-type rules | Raise `UnresolvableAnyPayloadError`, unless this field rule sets `allow_opaque_any: true`; that opt-in persists canonical `type.googleapis.com/<validated-FQN>` with an empty value. |

FQN extraction validates the suffix before lookup; a malformed suffix fails
closed. Unpacking is depth-limited by `max_message_depth`, with nested `Any`
values counting against the same budget. The application transform is not re-run
inside a packed value. The same explicit-policy rule applies to any `bytes` field
declared by the application as carrying a serialized message; grpcvcr does not
guess from its contents.

Policy paths for `Any` contents are rooted at the **packed** type, not at the
enclosing message: a rule is written
`google.protobuf.Any@package.Packed/field.subfield` and applies wherever a
payload of `package.Packed` is unpacked, at any nesting depth. Rules rooted at
the enclosing request FQN stop at the `Any` field itself and cannot reach inside
it.

An `Any` field with no enclosing field rule is dropped even when packed-type
rules exist; packed rules never implicitly authorize the enclosing field.

Resolution uses the session's private registry snapshot. Without an explicit
lookup pool, only registered descriptor closures are candidates. Supplying a pool
(including `descriptor_pool.Default()` explicitly) allows policy-named packed
types to be copied at snapshot time; later imports do not expand the session's
resolvable set. Construction fails if a policy-named packed FQN cannot be copied.

### Strict metadata policy

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
  grpcio versions differ in whether application attempts are rejected, accepted,
  or filtered before the server sees them; grpcvcr does not depend on that
  behavior. The deny-list prevents a hand-edited or migrated cassette from
  smuggling transport-reserved data back into replay.
- Stores entries as an ordered tagged list preserving duplicate keys and their
  interleaving. The transformer receives `(key, occurrence_index, value)` so a
  secret split across duplicate entries is still seen per occurrence.
- Rejects matching on an excluded key. `StrictMetadataMatcher` against an excluded
  key is a configuration error.
- Omits target persistence unless explicitly enabled. `StrictTargetMatcher` configured
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
`protobuf-safe` document contains exactly:

| Key | Contents |
| --- | --- |
| `profile_id`, `profile_version` | As in the envelope. |
| `field_policy` | Sorted normalized objects containing `field_path`, `verb`, and `allow_opaque_any`. |
| `metadata_allowlist` | Sorted lowercased keys with text/binary tag and caller-supplied non-secret `transformer_id`. |
| `status_details` | `drop` or `keep`. |
| `target_persistence` | Boolean. |
| `projection_transform_id` | Caller-supplied non-secret semantic ID, or `null` when absent. |

The `legacy-raw` canonical document instead contains exactly `profile_id`,
`profile_version`, fixed booleans `raw_metadata`, `raw_target`, and
`raw_status_details` (all `true`), plus `method_codecs`: the sorted list of method,
shape, request codec ID, and response codec ID from the explicit registry. This
makes the opaque bytes constructible and prevents playback under a differently
identified codec. Strict per-method descriptor identity is stored and validated
as `descriptor_sha256` on each interaction; it is intentionally not duplicated in
the profile-wide document.

`config_sha256` covers only inputs that shaped the persisted bytes. Limits and
matcher selection are reader-side policy: they change what a reader will *accept*
or *select*, never what a writer *produced*, and are therefore excluded. A reader
may tighten or relax its limits without invalidating existing cassettes.

Secrets, file paths, environment values, and callables are excluded. Every
opaque callable that can change persisted bytes requires a semantic ID in the
canonical document; changing its behavior requires changing that ID.
`grpcvcr profile-hash` prints both the canonical document and its digest so a
user can diff a rejection.

The hash covers normalized declarations and caller-managed semantic IDs, not
callable bytecode. Editing a projection or metadata transform without changing
its semantic ID is an application configuration error that grpcvcr cannot
detect. The invocation preparer never shapes persisted bytes and is excluded. On
mismatch, v2 playback fails with `ProfileMismatchError` reporting both digests.
The frozen v1 adapter has no profile digest and is the only warning-only path.

## Cassette formats and compatibility

### Schema v1

- `0.1.1` is the final published unrestricted v1 writer; no further v1-writing
  release is planned.
- In 0.2, v1 is playback-only.
- Two v1 dialects exist. The published `0.1.1` async recorder stores the method
  key as a YAML `!!binary` blob; the sync recorder stores text. Commit `262598b`
  normalized both to text on `main`, so cassettes recorded from `main` are a
  second v1 dialect that no release produced. The frozen adapter reads both and
  normalizes on load; a corpus fixture covers each. Migration classifies them
  identically.
- Direct v1 playback requires explicit `SessionConfig(profile=LEGACY_RAW)`;
  opening raw v1 under the strict default fails with a migrate-or-opt-in message
  before exposing interactions.
- Schema dispatch occurs before the v2 unsupported-shape guard. The adapter
  replays all four v1 shapes, using the serializer/deserializer supplied by the
  current stub and ignoring cassette-stored import paths.
- V1 client-streaming and bidi matching preserves the historical limitation: it
  eagerly drains and concatenates the request iterator because v1 stored no
  frame boundaries. This exception is confined to the frozen adapter and is
  called out in its API documentation.
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

`lossless` means every persisted response message and authorized metadata/status
value is byte-equivalent to the v1 replay result and the strict selector remains
unambiguous. `degraded` means migration is safe and replayable but policy drops or
replaces an application-visible value, drops metadata/status details, or creates
a projection collision that requires strict sequence. Unresolved descriptors and
unframed request streams are `re-record required`.

### Schema v2

Schema v2 supports all four RPC shapes and is YAML-only. It is the only new
schema this plan introduces: the causal event model that an earlier draft
deferred to a separate v3 is part of v2 from the first release, so no user ever
migrates a v2 cassette to v3. A representative envelope is:

```yaml
format: grpcvcr-cassette
schema_version: 2
required_features: []      # reader MUST reject any element it does not implement
optional_features: []      # reader MAY ignore unknown elements
extensions: {}             # feature data is namespaced by feature identifier
recording_profile:
  id: protobuf-safe
  version: 1
  config_sha256: "..."
interactions:
  - ordinal: 0
    method: /package.Service/Method
    shape: unary_stream
    target: null
    request_metadata: []
    request_fqn: package.Request
    response_fqn: package.Response
    descriptor_sha256: "..."
    events:
      - kind: client_message
        client_ordinal: 0
        payload:
          encoding: protobuf-python-deterministic-v1
          protobuf_type: package.Request
          protobuf_b64: "..."
      # client_half_close omitted: defaults to immediately after client_ordinal 0
      - kind: server_initial_metadata
        entries: []
      - kind: server_message
        # client_messages_sent omitted: defaults to 1, the preceding client_message count
        payload:
          encoding: protobuf-python-deterministic-v1
          protobuf_type: package.Response
          protobuf_b64: "..."
      - kind: server_trailing_metadata
        entries: []
      - kind: terminal_status
        code: OK
        details: null
```

The event grammar is uniform across all four shapes: a reader has one grammar
rather than a per-shape dialect, and no parser, validator, or replay path
branches on shape except for the cardinality table below.

Uniformity does not require writing determined values. `client_messages_sent` may
be omitted from a server event, and defaults to the number of `client_message`
events preceding it in the interaction. `client_half_close` may be omitted from
an interaction with exactly one `client_message` and a status the server produced
after receiving it, and defaults to immediately following that message. Both
defaults are shape-independent rules applied by the reader, so a `stream_stream`
cassette whose interleaving happens to match them is equally entitled to omit
them.

In practice the writer emits neither field for `unary_unary` or `unary_stream`,
emits `client_half_close` for `stream_unary` and `stream_stream`, and emits
`client_messages_sent` only on `stream_stream` server events where it differs
from the default. A unary cassette is then as compact as a per-shape dialect
would make it, and remains readable by the single grammar.

`client_messages_sent` is the number of `client_message` events the client had
submitted to the transport at the moment it observed this server event. It is a
client-side observation, **not** a statement about the server: a gRPC client
cannot observe which of its messages the server's handler had read, because
neither the gRPC API nor the HTTP/2 wire carries a message-consumption
acknowledgement. `WINDOW_UPDATE` reflects the receiving transport's buffer drain,
not the handler's `Read()`, and C-core does not surface it to Python in any case.
The value is therefore an *upper* bound on what the server could have consumed,
and it is sensitive to client task scheduling and transport buffering — two
recordings of the same deterministic server may differ.

It is recorded because it is the best available approximation of
request/response causality and is useful for diagnostics and for the opt-in
strict interleaving replay mode. It is **not** a replay gate by default; see
[Replay of `stream_stream` interleaving](#replay-of-stream_stream-interleaving).
The reproducible cross-direction fact is the position of `client_half_close` in
the event sequence, which corresponds to a real wire event (`END_STREAM` on the
request stream).

The sample shows the `protobuf-safe` payload variant. Under `legacy-raw`, each
payload instead has `encoding: opaque-base64`, a non-secret `codec_id`, and
`body_b64`; request/response FQNs and `descriptor_sha256` are optional. The payload variant is determined by the envelope's
`recording_profile.id` and may not vary within a cassette: `protobuf-safe`
requires `encoding: protobuf-python-deterministic-v1` on every payload,
`legacy-raw` requires `encoding: opaque-base64`. The v2 validator defines the
variants as a tagged union, rejects mixed or unknown payload keys within one
payload, and rejects any payload whose encoding disagrees with the envelope
profile. A `protobuf-safe` envelope carrying opaque payloads would advertise a
security claim for content that never passed the sanitizer, and is a load error
rather than a downgrade. The configured registry codec ID must equal the cassette value before the
stub-provided codec handles playback. `legacy-raw` never revives
cassette-provided Python imports. The event grammar is profile-independent, so
`legacy-raw` records all four shapes under the same causal event model; only the
payload encoding differs.

Unlike the strict profile, new `legacy-raw` v2 records the target, ordered request,
initial, and trailing metadata (including duplicate order and tagged/base64 binary
values), and status details verbatim. This is what makes `MetadataMatcher` retain its
historical meaning; `TargetMatcher` is `0.2`-new and carries the raw semantics
its implementation defines rather than a shipped one. Reserved transport metadata is
not filtered in this profile. It is explicit opt-in and makes no credential or
secret-safety claim.

`target` and `request_metadata` are interaction-level fields because they exist
before the first client message and do not vary across a request stream;
initial/trailing metadata remain ordered events. Strict mode writes `target: null` unless authorized and writes only
transformed allowlisted request entries. Legacy raw writes both verbatim as just
defined.

Feature names are lowercase dotted identifiers in a registry owned by this
project (`payload.any_recursive`, `transport.peer_identity`, …). Both feature
lists and the `extensions` mapping are mandatory keys; absent or wrongly typed
values are a load error, so a v2 reader can never mistake a malformed envelope
for a featureless one. Feature-specific data appears in one of two places, and nowhere else.

1. **Envelope-scoped data** lives below `extensions.<feature-name>` as a bounded
   namespaced subtree.
2. **Object-scoped data** — a field a feature adds to an interaction or an
   event — appears as a key on that object prefixed `x-<feature-name>.`, for
   example `x-transport.peer_identity` on an interaction or
   `x-payload.compression` on a `server_message`. A prefixed key is legal only if
   `<feature-name>` appears in exactly one feature list, mirroring the rule for
   `extensions` keys. A reader that supports the feature validates the key
   against that feature's schema; a reader that does not ignores it when the
   feature is optional, and has already rejected the cassette when it is
   required.

Unprefixed keys on any object must belong to the base schema; adding one is a
schema-major change. Object scoping exists because events carry no identity that
an envelope-scoped subtree could address, so without it every per-event and
per-interaction annotation would require a schema major — and per-event
annotation is precisely what streaming support is most likely to need.

A new **event kind** is always a `required_features` change and therefore rejects
the cassette on older readers even if the kind appears once. This is deliberate:
a reader that silently skipped an unknown event would replay a different exchange
than was recorded. New per-event *fields* on existing kinds are the additive path
that does not carry that cost, via an optional prefixed key.

A reader
rejects the cassette if any `required_features` element is outside its supported
set, naming the unsupported element and the current reader version. An old reader
cannot truthfully know which future grpcvcr release implements an unknown
feature. `optional_features` conveys writer intent only and never changes replay
semantics; an unknown optional feature's bounded namespaced subtree is ignored.

Envelope keys are validated *after* the feature lists are read, in this order:
(1) `format`, `schema_version`, both feature lists, and `extensions` must be
present and well-typed; (2) every required feature must be supported; (3)
top-level keys must belong to the base schema; (3b) every key prefixed `x-` on an
interaction or event must name a feature appearing in exactly one feature list;
(4) every `extensions` key must
appear in exactly one feature list; and (5) supported feature subtrees are
validated by that feature's schema while unknown optional subtrees are ignored.
Adding any other top-level key is a schema-major change.

The validator enforces shape-specific cardinality and ordering, exactly one
terminal result, no events after termination, known status codes, canonical
base64, valid FQNs, size limits, profile compatibility, and rejection of unknown
event kinds and required features.

Every interaction carries exactly one `server_initial_metadata` (possibly empty),
exactly one `server_trailing_metadata` (possibly empty), and exactly one
terminating `terminal_status`. Metadata events are synthesized as empty when the
live call exposes no entries, so the schema has no absent-versus-empty ambiguity.

`client_half_close` appears **at most** once. It is present exactly when the
client closed its request stream before the interaction terminated — the
application called `done_writing()`, or the request iterator was exhausted. It is
absent exactly when `terminal_status` arrived while the request stream was still
open. That distinction is client-observable and therefore recordable; forcing a
synthetic half-close that never happened would make the most common streaming
failure mode — a server rejecting a `stream_unary` call while the client is still
writing — unrepresentable.

Client messages are contiguously numbered from zero by `client_ordinal`, which
`client_messages_sent` and incremental request validation reference. Server
messages carry no ordinal; their order is their position in the event list, and
diagnostics name them by that position.

| Shape | `client_message` | `client_half_close` | `server_message` |
| --- | --- | --- | --- |
| `unary_unary` | Exactly one, or zero if the RPC terminated before the request was sent | Present iff a `client_message` is | Exactly one iff status is `OK`; none on non-OK |
| `unary_stream` | Exactly one, or zero if the RPC terminated before the request was sent | Present iff a `client_message` is | Zero or more; `OK` may legitimately have zero |
| `stream_unary` | Zero or more | Zero or one | Exactly one iff status is `OK`; none on non-OK |
| `stream_stream` | Zero or more | Zero or one | Zero or more, freely interleaved with client messages |

The grammar has no representation for a locally cancelled or abandoned call, and
needs none: such interactions are discarded rather than persisted, so no cassette
ever contains a partial exchange with no `terminal_status`. See
[Cancellation and incomplete interactions](#cancellation-and-incomplete-interactions).
A *remote* `CANCELLED` is an ordinary non-OK server status and is recorded like
any other, which commonly means an interaction with no `client_half_close`.

Additional ordering rules, which are what the streaming shapes actually need:

- No `client_message` may follow `client_half_close`. A recording whose events
  violate this is rejected at load, not at replay.
- `client_half_close` carries no ordinal. Its position in the event list
  determines which client messages preceded it, so an explicit back-reference
  would be derived data a hand-edit could contradict. An interaction may
  half-close with zero preceding `client_message` events; an empty request stream
  is legal for `stream_unary` and `stream_stream`.
- `client_messages_sent` is a count, not an ordinal, so a server-first exchange
  records `0` and needs no sentinel. It is non-decreasing across the server event
  sequence and never exceeds the number of `client_message` events preceding that
  event in the interaction. When omitted it defaults to exactly that number.
- `server_initial_metadata` precedes every `server_message`, the
  `server_trailing_metadata`, and the `terminal_status`. When the server sent no
  response headers — a trailers-only response, or a call that failed before
  headers arrived — the synthesized empty event is emitted immediately before
  `server_trailing_metadata`, so its position is determined by the recording
  rather than by writer choice.
- `server_trailing_metadata` follows every `server_message` and immediately
  precedes `terminal_status`. No event may fall between them.
- Recording `server_initial_metadata` at its true position requires the recorder
  to observe headers as they arrive, not lazily: the async engine awaits
  `initial_metadata()` on the delegated call in the controller task rather than
  only when the application asks. This changes no application-visible behavior,
  because `initial_metadata()` is already non-materializing and does not wait for
  the logical terminal.
- Interleaving is recorded as it occurred: for `stream_stream` the event list is
  a single ordered sequence containing both directions, not two concatenated
  runs. What replay guarantees about that order is defined in
  [Replay of `stream_stream` interleaving](#replay-of-stream_stream-interleaving).
- `terminal_status` terminates the interaction; no event may follow it. A server
  that terminates before the client half-closes is a legal early-failure
  recording, expressed by the absence of `client_half_close`.
- Client messages the application submits after the interaction's
  `terminal_status` are not persisted; the live call rejected them and they never
  reached the server. On replay, a write issued after the recorded terminal event
  raises the same error the live stack raises for a write to a terminated RPC —
  `AioRpcError` carrying the recorded status on `grpc.aio`, the corresponding
  `grpc.RpcError` on the sync stack — and is not treated as a request-stream
  mismatch. The recorded `client_message` count is therefore a lower bound on
  what the application wrote, and incremental validation stops at the terminal
  event.

### Replay of `stream_stream` interleaving

The recorded interleaving is preserved in the cassette because it is diagnostic
evidence of how the exchange actually ran. Replay does **not** promise to
reproduce it. A bidirectional interleaving is a function of both peers'
scheduling and of the application's own read/write ordering, and grpcvcr controls
only the server side of a replayed exchange; requiring the recorded interleaving
would mean blocking the application until its writes match a scheduling artifact
of the recording session, which deadlocks whenever they do not.

Replay guarantees exactly this, and the conformance suite asserts each part
separately:

1. **Response order and content.** `server_message` payloads are delivered to the
   reader in recorded order, one per successful read, with the recorded content.
2. **Response framing boundaries.** `server_initial_metadata` is observable
   before the first `server_message`; `server_trailing_metadata` and
   `terminal_status` are observable only after the last one.
3. **Request-stream validation.** The application's writes are validated in order
   against the recorded `client_message` sequence, and the half-close position is
   validated against the recorded one, per
   [Matching and consumption](#matching-and-consumption). A divergence is an
   error naming the ordinal, never a silent pass and never a search for a
   different interaction.
4. **The half-close boundary.** A `server_message` recorded after
   `client_half_close` is not delivered before the application half-closes. This
   is the one cross-direction ordering constraint that corresponds to a real wire
   event and is therefore reproducible. It is enforced by default, and it is what
   makes a recorded "collect-then-respond" server behave like one on replay
   rather than answering the first read immediately.

An opt-in `interleaving="strict"` session setting additionally gates each
`server_message` on its recorded `client_messages_sent` count. It is off by
default, its documentation states that it can block a replay whose application
writes fewer messages before reading than the recording did, and it converts that
block into a local `DEADLINE_EXCEEDED` at the call deadline rather than hanging
indefinitely.

The writer uses one deterministic YAML presentation: UTF-8, LF endings, final
newline, fixed schema key order, no anchors/aliases, no implicit binary tags, and
single-line canonical base64. Loading and saving an unchanged v2 model produces
identical bytes across the supported PyYAML matrix; the reader validates values,
not cosmetic whitespace from hand-authored files.

### Load-time hardening

Envelope parsing precedes validation and is itself adversarial input.
Size, bounded-read, duplicate-key, node/depth, and scalar limits apply to v1
playback/migration as well as v2. Before `json.loads`, the v1 JSON reader runs a
string/escape-aware lexical preflight over the bounded bytes to enforce nesting,
node, and scalar limits; `object_pairs_hook` then rejects duplicate keys. The
stdlib decoder is never the first component to inspect an unbounded structure.

- Open the cassette once with no-follow semantics where the platform supports
  them, `fstat` that descriptor, and reject it above `max_cassette_bytes`.
  Otherwise use `lstat` plus a post-open identity check. Read through a bounded
  reader capped at `max_cassette_bytes + 1`; a pathname `stat` followed by an
  unbounded reopen is forbidden because replacement can race the check.
- Parse with a `yaml.SafeLoader` subclass that (a) raises on any anchor or alias
  node, and (b) raises on duplicate mapping keys. PyYAML's stock `SafeLoader`
  expands aliases and lets a duplicate key silently override an earlier one,
  which would let a shadowing `payload` key evade validation.
- Enforce YAML node count, collection depth, and scalar-length limits during
  composition, before constructing Python containers. Schema-specific
  interaction/event limits are then enforced while constructing their lists,
  not after a fully materialized adversarial tree exists.
- Canonical base64 means: alphabet `A-Za-z0-9+/` with correct `=` padding, no
  whitespace, no line breaks, and zeroed padding bits. Enforce by decoding and
  re-encoding and requiring byte equality with the stored text —
  `binascii.a2b_base64(strict_mode=True)` is unavailable to the source-compatible
  Python 3.10 downstream fork and must not be the only check.

| Limit | Default | Enforced at |
| --- | ---: | --- |
| `max_cassette_bytes` | 32 MiB | Before read |
| `max_yaml_nodes` | 100 000 | YAML composition |
| `max_yaml_depth` | 64 | YAML composition |
| `max_scalar_chars` | 8 MiB | YAML composition |
| `max_interactions` | 1 000 | Parse |
| `max_events_per_interaction` | 10 000 | Parse |
| `max_payload_bytes` | 4 MiB | Before base64 decode, from encoded length |
| `max_protobuf_nodes` | 100 000 | Descriptor-aware wire preflight before message parse |
| `max_metadata_entries` | 64 | Parse |
| `max_metadata_value_bytes` | 8 KiB | Parse |
| `max_message_depth` | 100 | Redaction and reparse |

Limits come from the reader's configuration, never from the cassette, and are not
part of `config_sha256`. A cassette cannot raise the limits that gate it —
`max_cassette_bytes` in particular is enforced from the opened descriptor and a
bounded reader, so a value stored inside the file can never raise it.

### Profiles

Two v2 profiles are defined:

- `legacy-raw` preserves explicitly identified codec bytes, raw metadata/target/
  status details, and legacy matcher and replay behavior for all four shapes.
- `protobuf-safe` exposes immutable views, ordered consumption, secure metadata
  defaults, and policy-only persistence.

Only `legacy-raw` exposes mutable compatibility views or a direct raw
`record_interaction` path.

The field, metadata, status, target, and projection-transform knobs on
`RecordingProfile` belong to `protobuf-safe`. Constructing a `legacy-raw` profile
with values unequal to `LEGACY_RAW` (empty policies, status `keep`, target `true`,
and no transform) is a `ConfigurationError`; its raw behavior is fixed as above
rather than presenting controls it silently ignores.

### No schema v3

There is no schema v3. The causal event model that would have defined it is part
of v2, above. The `required_features` mechanism remains the forward-compatibility
path for additive changes, and a schema major is reserved for a change that
mechanism cannot express.

The event model is frozen only after executable prototypes cover ping-pong,
server-first, concurrent reader/writer, early failure, half-close, cancellation,
abandonment, a server that terminates mid-request-stream, and pre-stream
selection for a request-streaming shape. Because v2 now ships that model, those prototypes are a Phase 1
deliverable and a release gate rather than post-release work — this is the
schedule cost of the single-release decision, recorded in
[Release scope](#release-scope).

## Record modes and streaming routing

Mode behavior is defined for all four shapes. A v1 playback-only cassette under
explicit `LEGACY_RAW` is dispatched first and uses its explicitly documented
all-shape legacy adapter with the original eager, unframed semantics.

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
file is loaded regardless of mode (`src/grpcvcr/cassette.py:72-73`),
same-request interactions are replaced in place (`:120-123`), and the write is a
non-atomic `write_text` (`src/grpcvcr/serialization.py:473`) issued on close
regardless of outcome.

`ONCE` likewise changes: `Cassette.can_record` returns `True` for `ONCE`
unconditionally today (`src/grpcvcr/cassette.py:96`), so an existing cassette
does not currently force replay-only.

`NEW_EPISODES` rebuilds a generation in observed invocation order. At entry, the
loaded interactions form a read-only candidate sequence and the output generation
is empty. A replay hit is copied into the generation when reserved; a live miss
reserves the next generation position and fills it when complete. Under strict
sequence, a key mismatch at the next
candidate is a permitted live miss in `NEW_EPISODES` and does not consume that
candidate. On successful session exit, unused source interactions are appended in
their original relative order and every interaction is renumbered contiguously.
Thus source `[A, B]` plus calls `[A, C]` commits `[A, C, B]`, not `[A, B, C]`.
`NONE` and existing-cassette `ONCE` still treat the same mismatch as an error.

Request-streaming routing, which is in scope for this release:

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
  verifies it; a key mismatch is an error except for the explicitly defined
  `NEW_EPISODES` insertion rule, and is never a search for a better candidate.
  Under first-unused matching the same key selects.
- Response FQN verifies the interaction in both regimes and is never a selection
  key.
- FQNs are optional under `legacy-raw` and mandatory in strict mode.
- Strict diagnostics contain only method, shape, safe FQNs, candidate ordinal,
  consumption state, and safe digests.
- Generation position is reserved at invocation rather than completion; ordinals
  are validated and written as contiguous integers at commit.

The strict key comprises the unconditional base tuple `(method, shape,
request_fqn, deterministic_safe_projection)` followed by the ordered,
length-prefixed `(semantic_id, selector_fragment)` values from the configured
additive `StrictMatcher`. Thus selected safe metadata, target, or a custom
constraint participates only when its matcher contributes a fragment; none can
remove a base component.

For `unary_unary` and `unary_stream` the base tuple's fourth component is the
deterministic safe projection of the single request message. For `stream_unary`
and `stream_stream` no request message exists at invocation — that is the premise
of lazy consumption — so the base tuple is `(method, shape, request_fqn,
PRE_STREAM)`, where `PRE_STREAM` is a fixed sentinel byte string distinct from
any projection. This is the pre-stream selector: the request stream contributes
nothing to *selection*, and is *validated* after selection, incrementally.

Because the base tuple for a request-streaming shape discriminates only on method
and shape, `first_unused` matching on a cassette with two same-method `stream_*`
interactions is ambiguous by construction and is refused by the existing
ambiguity rule unless an additive `StrictMatcher` — metadata, target, or custom —
supplies discrimination from data available before the first write.
Strict-sequence playback, the `protobuf-safe` default, is unaffected: the ordinal
selects.

**Incremental request-stream validation.** Once a `stream_unary` or
`stream_stream` interaction is reserved, each application write is projected and
compared against the reserved interaction's `client_message` events in order:

- A projection unequal to the expected `client_ordinal`'s payload raises the
  strict `NoMatchingInteractionError`, carrying a `SafeMismatchReport` naming the
  method, shape, expected and observed client ordinal, and both safe digests. It
  is raised from `write()` on aio and from the request pump's next step on sync.
  This is deliberately a grpcvcr error rather than the `InvalidStateError` that
  gRPC conformance reserves for terminated-RPC writes: the two conditions are
  different and must be distinguishable by type. Selection is never revisited —
  the interaction is already reserved and stays consumed, and no search for a
  better candidate occurs.
- A write after the recorded `client_message` events are exhausted is an error
  naming the recorded count, unless the recorded `terminal_status` has already
  been reached, in which case the write raises the terminated-RPC error instead.
- A half-close before they are exhausted is an error naming the ordinal reached
  and the recorded count.
- Under `legacy-raw` the comparison is over codec bytes rather than projections;
  the ordering rules are identical.

Deny-by-default redaction makes projection collisions the normal case, not an
edge case: once identifying fields are dropped or replaced with constants,
distinct requests to the same method commonly project to identical deterministic
bytes. The key therefore does not uniquely identify an interaction and is not
treated as if it did.

- Global strict-sequence playback is unaffected: ordinal selects, and the
  selector is a consistency assertion. This is why it is the `protobuf-safe`
  default.
- First-unused matching is refused on a cassette containing two interactions with
  an identical complete key — base tuple plus every configured built-in or custom
  fragment, including target when selected. The refusal names the colliding
  ordinals and matcher semantic IDs, and points at strict-sequence
  playback or at a `keep` rule that would restore discrimination. Silently
  consuming the first of an ambiguous pair would return a response recorded for a
  different request.
- The writer detects collisions before commit. Strict-sequence sessions may write
  them because ordinal selection remains unambiguous. A `first_unused` session
  fails the commit with a safe report naming the colliding ordinals and retains
  dirty state; it never writes a cassette that its own configured reader rejects.

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
- Local cancellation is a runtime-local fact and is never persisted, so no
  cassette-derived local cancellation state is ever claimed. Schema v2 records a
  remote `CANCELLED` status and nothing more.
- An exception raised by the application's request iterator is a runtime-local
  fact, never a server result. gRPC reports it differently per stack and grpcvcr
  reproduces both: sync cancels the call and synthesizes a local
  `StatusCode.UNKNOWN` with details `Exception iterating requests!`, logging the
  original exception (`grpc._channel._consume_request_iterator`); aio cancels the
  call, so `await`/`read()`/`__anext__` raise `asyncio.CancelledError`, and the
  original exception is only logged. Neither is recorded. The sync case is
  indistinguishable at the public API from a remote `UNKNOWN`, so the interaction
  is discarded on an explicit local-origin flag, never by inspecting the status
  code.
- Cancelling a call mid-request-stream stops request consumption. On aio the
  request-pump task is cancelled, so the application's async generator is thrown
  into at its current suspension point; on sync the pump observes the cancelled
  state before its next send and returns without being killed mid-`next()`, so a
  generator blocked in user code still runs to its next yield. In neither stack
  is the application's iterator advanced after cancellation completes.
- After cancellation, aio `write()` and `done_writing()` raise
  `asyncio.InvalidStateError`, **not** `asyncio.CancelledError` — a pending write
  blocked on the delegation barrier is released with that error rather than left
  hanging. Applications wrapping `write()` in `except asyncio.CancelledError`
  will not catch it.
- A `ReplayCall` cancelled or abandoned mid-request-stream counts the interaction
  as consumed, exactly as a mid-response-stream cancellation does.
- A terminal status synthesized locally by grpcvcr rather than received from a
  server — deadline expiry before delegation, transport-factory failure,
  invocation-preparer failure — is runtime-local in exactly the sense local
  cancellation is, and is likewise discarded. Only statuses a server actually
  produced, `OK` or not, are persisted. An interaction whose `terminal_status`
  was never on the wire would replay a fabricated exchange, including a
  `client_message` for a request that was never sent, turning a timing accident
  during recording into a permanent fixture.

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

### Session commit and abort

The session transaction is a one-way state machine:

```text
OPEN ──close/save──► CLOSING ──success──► COMMITTED
  │                     │
  │                     ├──failure before replace──► DIRTY_NOT_COMMITTED
  │                     └──failure after replace───► DIRTY_UNKNOWN
  └──exception exit──► ABORTED

{DIRTY_NOT_COMMITTED, DIRTY_UNKNOWN} ──later commit-only close──►
    COMMITTED | DIRTY_NOT_COMMITTED | DIRTY_UNKNOWN
```

The first stack-appropriate `close()`, or public `Cassette.save()` on an
`UNBOUND`/`SYNC` session, seals the session against new calls; `save()` is a
terminal commit operation, not a checkpoint that leaves the context open. A
rejected sync operation on an open `AIO` session does not transition this state
machine. Concurrent callers share the in-progress attempt as specified in
the lifecycle sections. After `COMMITTED` or `ABORTED`, repeated save/close returns
the stored completion; a committed close with transport/finalizer failures
reconstructs its stored highest-precedence safe close error. Dirty states accept only a later commit-only
retry. No lifecycle work is rerun.

The transaction rows are normative:

| Event | Affected interaction | Session generation |
| --- | --- | --- |
| Normal context-manager exit or explicit `close()` / `save()` while `OPEN` | Completed interactions retained. | Seal and commit after finalization. |
| Application exception escapes while still `OPEN` | Any in-flight interaction discarded. | Transition to `ABORTED`; source cassette is not intentionally replaced. |
| Application exception after an explicit commit | No new work was permitted after sealing. | `COMMITTED` is irreversible; preserve the application exception and do not pretend to abort or replace again. |
| Application exception after a caught commit failure | No lifecycle work reruns. | Preserve the application exception, retain `DIRTY_NOT_COMMITTED` or `DIRTY_UNKNOWN`, and require a later explicit commit-only retry. |
| Remote non-OK terminal status | Complete and replayable. | Eligible to commit. |
| Local cancellation or abandonment | Discard. | Other completed interactions remain eligible. |
| Request-iterator exception (local `UNKNOWN` on sync, local cancel on aio) | Discard; never recorded as a server status. | Same caught/escaping rule as above. |
| Deadline expiry before delegation | Discard; no RPC started and no server status exists. | Other completed interactions remain eligible. |
| Transport factory failure | Discard; emit `startup_failed`. | Other completed interactions remain eligible. |
| Request sanitization/validation failure | No live RPC starts; discard. | If caller catches the safe error and the context exits normally, other completed interactions may commit. |
| Response sanitization failure | Discard; raw application response is never persisted. | Same caught/escaping rule as above. |
| Preparer failure before `LiveInvocation` exists | Discard; no completion hook. | Same caught/escaping rule as above. |
| Completion hook failure on normal close | Discard; emit `hook_failed`. | Commit other completed interactions, then raise chain-cleared `FinalizationError`. |
| Completion hook failure while an application exception is escaping | Discard; emit `hook_failed`. | The application exception has precedence; abort if still `OPEN` and do not mask it. |
| Owned transport close failure on normal close | Completed interactions remain valid; emit `transport_close_failed`. | Close every other owned transport, commit, then raise chain-cleared `TransportCloseError`. |
| Owned transport close failure while an application exception is escaping | Same exactly-once cleanup; emit the safe event. | The application exception has precedence; abort if still `OPEN` and do not mask it. |
| Storage failure before replace | No commit. | Dirty state retained for explicit retry. |
| Failure after replace | New complete file may already be visible; durability is unknown. | Raise with `commit_state="unknown"`; update the in-memory source hash and retain dirty state for an idempotent retry. |

The context manager passes exception information into the session close path; a
plain `finally: save()` implementation is forbidden. Remote RPC errors do not by
themselves count as application-body exceptions once observed by the caller. A
storage failure takes precedence over transport/finalization errors for that close
attempt; their safe reports are retained and the highest-precedence one surfaces
after a successful retry.

## Storage transaction

Writable sessions use:

- Invocation-order interaction numbering.
- Atomic find/reserve/consume operations.
- Temporary files created in the **destination directory** with
  `O_CREAT|O_EXCL` and mode `0600`, plus `O_NOFOLLOW` where available, never in
  `$TMPDIR`: `os.replace` is atomic only within one filesystem. The Windows
  helper uses handle-based reparse-point checks instead of pretending
  `O_NOFOLLOW` exists there.
- Rejection of a destination path that is a symlink, resolved under the acquired
  lock. `os.replace` overwrites the link itself, silently detaching a shared
  cassette.
- Explicit pre-replace permissions: on POSIX, the open temporary descriptor is
  `fchmod`ed to the mode of the regular file it replaces, or left `0600` when
  creating. On Windows, a ctypes-backed security helper copies and verifies the
  existing destination's protected DACL, or applies a protected DACL granting the
  current token user full control for a new file. Failure to apply or verify the
  platform policy fails before replace. No cross-platform claim equates POSIX mode
  bits with Windows ACLs.
- An exclusive sidecar writer lock held with `fcntl.flock` (POSIX) or
  `msvcrt.locking` (Windows) on an open descriptor, never an `O_EXCL` sentinel
  file: an advisory lock is released by the kernel on process death and needs no
  stale-lock heuristic. The sidecar itself is opened/created as a regular
  owner-only file with no-follow or Windows reparse-point checks; a symlinked or
  non-regular lock path is rejected.
- Lock acquisition is nonblocking (`LOCK_EX|LOCK_NB` on POSIX,
  `LK_NBLCK` on Windows). Contention immediately raises a chain-cleared
  `CassetteWriteError(reason="writer_busy", commit_state="not_committed")`; there
  is no hidden indefinite wait or configurable timeout in 0.2.
- Serialization, validation of the serialized temporary artifact, flush,
  `fsync`, a final source-hash comparison, and `os.replace` are all performed
  **inside** the held lock. The final hash check occurs immediately before
  replace. It detects cooperative-session conflicts and most non-cooperating
  edits, but advisory locking cannot eliminate the final race with a writer that
  deliberately ignores the lock; the documentation does not claim otherwise.
- Directory `fsync` where supported.
- Temporary-file cleanup on every failure where the temporary path still exists.
- Preservation of dirty state after a failed save, with explicit
  `commit_state="not_committed"` before replace and `commit_state="unknown"`
  after replace.

Concurrent writers are rejected rather than merged. Readers observe either the
old complete cassette or the new complete cassette, never an intermediate file.

Platform behavior is explicit:

| Platform | No-follow/path check | Writer lock | Committed permissions | Directory durability | Replace retry |
| --- | --- | --- | --- | --- | --- |
| Linux | `O_NOFOLLOW` plus descriptor `fstat` where available; identity fallback otherwise. | nonblocking `flock` | Preserve existing POSIX mode with pre-replace `fchmod`; new file `0600`. | Directory `fsync`; a failure after replace yields `unknown`. | Retry only `EINTR`; other errors fail. |
| macOS | Same POSIX contract, with runtime capability checks. | nonblocking `flock` | Same POSIX mode guarantee. | Directory `fsync` where the filesystem supports it; unsupported sync is reported as a documented durability limitation, not success. | Retry only `EINTR`. |
| Windows | Handle-based reparse-point and pre/post-open file-identity checks. | nonblocking `msvcrt.locking` | Preserve/verify existing protected DACL or apply/verify the new-file current-user DACL; no POSIX-mode promise. | No directory `fsync`; successful replace is complete but crash durability is weaker. | Bounded exponential retry only for sharing/access violations caused by open readers, then `not_committed`. |

A directory-sync failure after replace cannot restore the prior file; the visible
file is complete but crash durability is unknown. On Windows, a crash may lose the
rename while leaving a complete old or new file. Advisory locking over NFS depends
on a working lock daemon and is not supported for concurrent writers;
single-writer use on NFS is fine.

## Implementation plan

Each implementation phase is a separately named pull request and merges onto a
green main without depending on an unmerged successor. Phase labels are local to
this plan and are never GitHub pull-request numbers. Phase 2 has two intentional
pull requests, and Phase 5 is a series. Every phase lands before the single `0.2`
release; none of them publishes.

| Phase | Title | Estimated effort | Approximate diff |
| --- | --- | --- | ---: |
| 0 | Baseline snapshot and release hardening | 3–5 engineer-days | +500 / −60 |
| 1 | Public API, routing contracts, and streaming prototypes | 6–10 engineer-days | +800 / −100 |
| 2a | V2 models, validator, and storage transaction | 4–7 engineer-days | +1800 / −250 |
| 2b | Safe projection, sessions, matching, and v1 adapter | 6–8 engineer-days | +2700 / −450 |
| 3 | Async routing engine and live-call lifecycle | 10–15 engineer-days | +3000 / −550 |
| 4 | Sync parity, compatibility facades, and the pytest plugin | 10–15 engineer-days | +2000 / −900 |
| 5 | Request-streaming and bidirectional shapes | 25–35 engineer-days | series |
| 6 | Migration tooling, dependency matrix, typing, documentation, and release | 5–12 engineer-days | +1200 / −450 |

Documentation, typing, and the dependency matrix land last, after streaming. They
must close over the *final* public surface, and with the streaming shapes in the
release that surface is not final until Phase 5 lands. Running Phase 6 first
would mean documenting and type-freezing an API that Phase 5 then changes.

The downstream pilot is not a `grpcvcr` pull request. It lands in the consumer's
repository and is tracked by the
[pilot's adoption gate](#appendix-downstream-pilot-notebooklm).

Each phase pull request must be independently reviewable. Regression tests land with
their fix or as explicitly tracked strict `xfail` contracts; the main branch is
never left with unexplained failing tests.

### 0. Baseline snapshot and release hardening

Contains the executable public-surface snapshot taken against the **published
`v0.1.1` tag**, not against `main`, for every root
export, `Cassette` method, matcher, exception attribute,
`grpcvcr.interceptors` export, pytest fixture, CLI option, and marker form;
hardened publishing so a tag builds and tests wheel/sdist artifacts, then waits
at a protected manual-promotion environment before publishing those same bytes;
`pytester` entry-point discovery through that wheel; regeneration of checked-in
test protobufs with the declared grpcio/protobuf floors; and correction of the
false current-release documentation claims.

Exit criteria:

- A wheel and sdist built from `main` install, discover the pytest plugin, and
  pass the existing suite against the dependency floor and current dependencies.
- The published documentation contains no claim that is false today.
- The release workflow cannot publish before its protected environment approval.
- A non-`v` tag `baseline-0.1.x` marks the pre-refactor tree.

Nothing is published here. `release.yml` publishes an artifact it never installs
or tests today, so the hardening lands long before the only tag that will trigger
it — `v0.2.0`, at the end of Phase 6. The `baseline-0.1.x` marker is deliberately
non-`v` so the workflow's `v*` trigger ignores it.

### 1. Public API, routing contracts, and streaming prototypes

Contains the proposed 0.2 public-surface snapshot, normative configuration
objects and signatures, public `Channel`/`MultiCallable`/`Call` conformance spike,
aio interceptor differential harness, and the bidi routing and causal-event
prototypes.

Those prototypes are load-bearing here in a way they were not when streaming was
a later release. Schema v2 embeds the causal event model, and Phase 2a freezes
the v2 validator, so the model must be proven before it is frozen. This phase is
larger than an equivalent contracts-only phase for exactly that reason.

Record the resolved Python policy here: upstream remains Python 3.11+, while
replacement modules avoid needless 3.11-only syntax so the downstream fork can
carry a tested 3.10 packaging patch.

Coherent because the shipped surface it adds is inert — contracts, configuration
objects, and ABCs that neither route nor persist a v2 call. The streaming
prototypes live under `prototypes/`, outside the measured package and outside the
wheel; they prove the event model and are deleted when Phase 5 supersedes them.
Later behavior is measured against its golden files.

Exit criteria:

- Routing feasibility is demonstrated for both stacks, including the aio
  interception chain: the spike reproduces `grpc.aio` interceptor dispatch for
  all four shapes, at prototype fidelity for the request-streaming pair.
- The differential-test harness exists and runs against a real `grpc.aio`
  channel. The implementation it will be pointed at ships in Phase 3,
  which is where the comparison becomes an exit criterion.
- Public compatibility obligations are documented, name by name.
- The strict and legacy default configurations construct and reject every invalid
  cross-profile matcher/policy combination named in this plan.
- Executable prototypes cover ping-pong, server-first, concurrent
  reader/writer, early failure, half-close, cancellation, abandonment, a server
  that terminates mid-request-stream, and `NEW_EPISODES` selecting a
  `stream_unary` interaction before the first write, and the event model records
  and replays each without loss. **This is the
  gate that freezes the v2 event grammar**; Phase 2a's validator encodes whatever
  it proves.

### 2a. V2 models, validator, and storage transaction

Contains v2 models, strict validator, profile metadata, safe internal
constructors, bounded adversarial YAML loading, atomic storage, exclusive writer
locking, source-hash conflict detection, transaction-state errors, and fault
injection.

Coherent because it is pure validated data and I/O and touches no gRPC call
object. It can be reviewed and tested without a channel.

Exit criteria:

- V2 round-trips and rejects invalid state machines before constructing
  persistable objects.
- Faults before replace leave the prior cassette intact; faults after replace
  yield a complete file and explicit `commit_state="unknown"`.
- Linux, macOS, and Windows storage subprocess tests pass.

### 2b. Safe projection, sessions, matching, and v1 adapter

Contains the encode-once projection pipeline, descriptor registry, mandatory
redactor, metadata policy, profile hashing, sessions, atomic reservation,
ordered consumption, structured mismatch reports, safe observability, the frozen
playback-only v1 parser/model adapter, and migration classifier. It depends only
on 2a; it deliberately contains no public call objects or channel routing.

Exit criteria:

- Raw data has no path into a strict persistable object.
- V1 never auto-upgrades.
- `NEW_EPISODES` generation rebuilding preserves observed order.
- Every released-v1 corpus file loads within bounds, ignores stored imports,
  validates into frozen adapter models, and receives the expected migration
  classification. End-to-end calls are gated in Phase 4 after routing exists.

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

- Barrier-controlled fake streams prove no second message is pulled before the
  application requests it; no wall-clock latency threshold is used.
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
`.cassette`, `.target`, and context-manager entry shapes
(`RecordingChannel.__enter__`/`__exit__`, `AsyncRecordingChannel.__aenter__`/
`__aexit__`, `Cassette.__enter__`/`__exit__`), whose semantics change because
sync `close()` now aborts in-flight calls and normal aio `__aexit__` awaits
`close(None)` while exceptional exit uses the abort path; removal of
`async_recorded_channel` and its proper
`aio_recorded_channel` replacement; pytest plugin hardening; removal of the plugin from
`[tool.coverage.run] omit` **and** removal of `-p no:grpcvcr` from the CI and
`Makefile` test invocations, without which the plugin is still never imported in
the run that measures coverage; and resolution of the unusable `grpcvcr_channel`
fixture. That fixture is undocumented in `docs/guides/pytest.md`, so deleting it
is low-risk and preferred over making it functional.

The compatibility facades enforce the runtime binding rule: mixed sync/aio
attachment fails, and sync cassette save/exit cannot terminate an open aio
session. The Phase 1 breaking snapshot and release notes include terminal save,
exceptional-exit abort, and the required async nesting order.

This phase also connects the frozen v1 models to both sync and aio legacy replay
call objects for all four historical shapes. Keeping that integration here avoids
claiming Phase 2b can replay calls before either routing stack exists.

Coherent because it is the whole backwards-compatibility surface, verified
against the golden snapshot from Phase 1. The plugin belongs here because its
fixtures construct exactly the facades this phase restores.

Exit criteria:

- Sync and aio behavior match the **unary-unary and unary-stream rows** of the
  public conformance matrix. The two streaming rows are Phase 5's exit criterion
  and are not asserted here, so Phase 4 does not depend on unmerged Phase 5.
- Every name in the Phase 1 snapshot is resolved as facade, deprecation,
  or documented removal.
- The sync deferred-call state model, per-operation wait timeouts, cancellation,
  finalization, and executor shutdown pass deterministic race tests.
- The corpus replays JSON/YAML, sync/aio, and all four shapes with stored
  imports, transport, DNS, sockets, and credentials disabled. It contains files
  produced by the released 0.1.1 wheel — including async recordings carrying the
  `!!binary` method key — and files produced by post-`262598b` `main`, which
  carry the normalized text key.

### 5. Request-streaming and bidirectional shapes

Implements the pre-stream routing selector contract; async iterator and explicit
`write`/`done_writing`/`read` modes; rejection of mixed iterator and read/write
use as gRPC does; sync client-streaming and bidi request/response iterator
wrappers; and lazy request-iterator consumption that never buffers a stream to
choose a route. Covers server-first, ping-pong, half-close, early failure,
concurrent reader/writer, cancellation, and abandoned calls.

Not a single pull request. It is a series against the event grammar frozen in
Phase 1 and validated in Phase 2a, sequenced after sync parity so both stacks
gain the streaming shapes against one settled call lifecycle.

Coherent because it is the last change to the public surface. Nothing after it
alters an API signature.

Exit criteria:

- No mode drains a request iterator to make a routing decision, and generator
  side effects fire in the same order as against a real channel.
- `stream_stream` replay satisfies the four guarantees in [Replay of
  `stream_stream` interleaving](#replay-of-stream_stream-interleaving), verified
  against a real server, and a differential test shows that two recordings of the
  same deterministic bidi server may differ in `client_messages_sent` while both
  replay identically under the default mode.
- `UnsupportedRpcShapeError` is unreachable for the four standard shapes under
  `protobuf-safe`.

### 6. Migration tooling, dependency matrix, typing, documentation, and release

Contains cassette `validate`, migration dry-run and reporting; a floor job on
Python 3.11 using grpcio 1.50/protobuf 4.21-compatible generated fixtures;
current-dependency jobs across Python 3.11–3.14; Linux, macOS, and Windows storage jobs;
matching classifiers; a separately owned Python 3.10 downstream-fork job;
tightened Pyright on replacement modules and
`pyright --verifytypes` after the public API stabilizes; removal of the unused
`[tool.mypy]` block and the `mypy`, `types-protobuf`, and `types-pyyaml` dev
dependencies, since only pyright runs in pre-commit and CI; `CHANGELOG.md` as the
canonical release-note home; documentation
rewritten to the shapes actually supported; executable documentation examples
restored, or the dead `test-examples` Makefile target and the two
`--ignore=tests/test_examples.py` flags removed, since `tests/test_examples.py`
does not exist; `validation: anchors: warn` added for published documentation;
a separate repository-relative Markdown-link check run over this excluded plan,
because MkDocs never loads it; and installation and testing of wheels and source
distributions before release.

Coherent because it is the release gate: configuration, tooling, and prose over a
frozen public API.

Exit criteria:

- A user can identify whether a v1 cassette is losslessly migratable, degraded,
  or must be re-recorded.
- CLI exit codes, `--json`, no-write dry runs, and canary-safe stdout/stderr pass.
- Documentation examples execute, all anchors validate, and the complete 0.2
  breaking-change list appears in the versioned changelog section.
- The release workflow records final artifact hashes and exposes the protected
  post-merge pilot/rollback-rehearsal gate used before promotion; the five-night
  wait is a release gate below, not a condition for merging the Phase 6 PR.
- `v0.2.0` is tagged only after every gate passes. It is the first and only tag
  this plan publishes.

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
- Every `SafeEvent` kind.
- `__cause__`, `__context__`, `__dict__`, and formatted tracebacks for failures
  invoked normally and from inside a caller-owned canary `except` block.

Decode persisted protobufs with descriptors and verify their structure; regex
scanning base64 text alone is insufficient. Verify that application messages are
never mutated. Verify deterministic, idempotent output across supported Python
and protobuf versions, and across `upb` and the pure-Python implementation.
Include a dynamic proto2 enum whose declared values start at 1 and assert that
`keep_shape` uses the first declared value for singular/repeated fields without a
`ValueError` or lost presence.
Also cover implicit-presence proto3 numeric, bool, string/bytes, and enum fields:
non-default (including `-0.0` and an unnamed open-enum number) becomes the fixed
non-default placeholder and still serializes, while a default remains absent.
Contrast each with proto3 `optional` and oneof presence using default placeholders.

### Schema and storage tests

- Unknown keys, events, and required features.
- YAML anchors, aliases, and duplicate mapping keys.
- YAML node-count, nesting-depth, scalar-length, bounded-read, and file-replacement
  races before Python container construction.
- Invalid and noncanonical base64.
- Invalid metadata typing, `-bin` handling, duplicate keys, and case folding.
- Illegal event ordering and cardinality.
- `client_messages_sent` rules: non-decreasing across server events, never
  exceeding the count of preceding `client_message` events, `0` accepted for a
  server-first exchange, and the omitted-field default resolving to exactly the
  preceding count.
- `client_half_close` cardinality: a legal early-failure recording carrying none,
  rejection of a second one, rejection of a `client_message` following one, and a
  legal empty request stream that half-closes with zero preceding messages.
- Metadata event placement: `server_initial_metadata` before every
  `server_message`, `server_trailing_metadata` immediately before
  `terminal_status`, and a trailers-only response placing the synthesized empty
  initial-metadata event deterministically.
- Payload encoding disagreeing with the envelope profile is a load error in both
  directions.
- Object-scoped `x-<feature>.` keys on interactions and events: accepted when the
  feature is listed, rejected when it is not, ignored when optional-unsupported.
- Missing or multiple terminal events.
- Oversized files and messages against each declared limit.
- Disk-full, permission, serializer, validator, rename, and directory-sync
  failures.
- Cross-filesystem `os.replace`, symlinked destinations, and committed file mode.
- Concurrent reservations, out-of-order completions, channel close with active
  calls, and source-hash conflicts.
- Readers observing only complete old or new files.
- Linux, macOS, and Windows subprocess crash injection after temp write, temp `fsync`,
  replace, and directory sync; assertions distinguish `not_committed` from
  `unknown` rather than promising rollback after replace.
- POSIX and Windows multiprocess lock contention, owner process death, lock reuse,
  a Windows reader holding the destination open, and platform-appropriate
  symlink/reparse-point handling.
- A macOS storage smoke job for no-follow checks, nonblocking lock contention,
  mode preservation, replace, and supported/unsupported directory-sync behavior.

### Call-lifecycle tests

- Await, read, iteration, metadata, status, details, callbacks, and close.
- Every deferred-call state and cancellation transition, including
  `STARTING → DONE` and the cancel/delegate race.
- Deadline expiration before, during, and after transport materialization.
- Replay deadlines with `None`, zero, negative, expiry before first read, between
  stream messages, and after recorded termination, using a fake monotonic clock.
- Multiple operations racing on one call without duplicate startup.
- Concurrent calls sharing transport but not invocation preparation.
- Close during sync/async transport-factory creation, a factory that ignores
  cancellation and returns late, exactly-once close of every default/custom
  transport, deterministic multi-transport close order, blocking close, close
  failure, and storage/transport/finalizer/application-exception precedence.
- Partial server streams followed by non-OK status.
- Request-streaming and bidi lifecycle: lazy iterator pull proven by a
  barrier-gated generator, exactly one message pulled before delegation,
  half-close timing relative to server events, `write`/`done_writing` against a
  real server, and generator side-effect ordering compared against a real
  channel.
- Request style fixed at `__call__`: the first `write()` on an
  iterator-constructed call raises `grpc.aio.UsageError`, and a call constructed
  with `request_iterator=None` has no iterator API to conflict with.
- `write()` after termination and after half-close raises
  `asyncio.InvalidStateError` with gRPC's two messages, never `AioRpcError` or
  `asyncio.CancelledError`; a write pending on the delegation barrier is released
  rather than left hanging.
- `stream_stream` replay satisfies the four interleaving guarantees, and two
  recordings of the same deterministic server differing in `client_messages_sent`
  both replay identically under the default mode.
- Cancellation mid-request-stream stops consumption on both stacks and never
  advances the application's iterator afterward.
- A request iterator that raises is discarded on both stacks and never recorded
  as a server status, including the sync local-`UNKNOWN` case that is
  indistinguishable at the public API from a remote one.
- Concurrent sync client-streaming calls exceeding half the controller pool make
  progress, proving pumps do not share the bounded executor.
- Routing decisions for every mode on both request-streaming shapes, asserting
  the request iterator is untouched at reservation time.
- Local cancellation versus remote `StatusCode.CANCELLED`, asserting
  `asyncio.CancelledError` on aio and `grpc.FutureCancelledError` on sync.
- A call constructed and abandoned without being consumed, asserting no
  "Task exception was never retrieved" or "Future exception was never retrieved"
  warning on startup failure, deadline, or cancellation before delegation.
- `close(None)` and `close(grace)` with active calls.
- Finalizer re-entry into status methods, async failure/slow watchdog, sync blocking,
  late preparer settlement after cancellation, and independent
  `logical_terminal` / `preparation_settled` / `finalization_done` ordering.
- Every row of the session commit/abort table for sync and aio.
- Exception after explicit `save()`, exception after explicit close, concurrent
  double close, repeated terminal close, and commit-only retries from both
  `DIRTY_NOT_COMMITTED` and `DIRTY_UNKNOWN`.
- Mixed sync/aio attachment, aio attachment from a second loop, synchronous
  `save()` and cassette exit during active aio calls, and both correct and reversed
  cassette/channel nesting. Rejection leaves the session open so awaited close
  still releases every resource.
- Exact `SafeEvent` sequences, populated/empty fields, global sequence ordering,
  sink failure/re-entry, and inline-latency deadline behavior — not only canary
  absence by event kind.
- Under asyncio debug, repeated sync/aio session creation followed by normal
  close, cancellation, abandonment, startup failure, and hook failure. Snapshot
  pending controller/dispatcher tasks, session executor threads, live transport
  channels, cassette/lock descriptors, and destination-directory temporary files;
  all return to baseline after close (apart from documented process-global grpcio
  pools measured by a warmed control).

### Compatibility and migration tests

- Existing root exports, signatures, attributes, exceptions, matchers, fixtures,
  and markers against the Phase 0 baseline, plus the resolved 0.2 surface against
  the Phase 1 proposed snapshot.
- A committed corpus generated by the released 0.1.1 wheel: JSON and YAML, sync
  and aio, all four shapes, OK/non-OK, partial streams, metadata/trailers,
  repeated interactions, and absent legacy type fields. Replay runs with stored
  imports, transport, DNS, sockets, and credentials disabled.
- No implicit v1 writing or conversion.
- Metadata dropped by default during migration.
- Refusal to migrate unresolved types or unframed request streams.
- Installed pytest entry-point discovery through a built wheel.
- CLI exit codes, stable `--json`, canary-safe stdout/stderr, read-only validate,
  no-write dry-run, source-equal refusal, and existing/symlink destination refusal.

## Quality gates

This is the single normative gate list. Phase exit criteria assert only what is
specific to that phase.

Before the downstream pilot:

- Replay works with all transport, auth, DNS, and socket paths disabled.
- Playback emits zero `live_started` events.
- Strict sanitizer canaries pass across all required protobuf and metadata
  surfaces, including packed payloads and hostile `Any.type_url` prefixes.
- Original live request and response objects remain unmodified.
- Matching derives from actual emitted request bytes, including custom serializer
  mutation tests.
- Deterministic serialization is byte-stable across `upb`, pure-Python, and the
  protobuf floor.
- Repeated interactions consume in deterministic order.
- Unary-stream recording and replay remain pull-based and lazy.
- Deadline, callback, cancellation, and close races pass.

Before `0.2`:

- The full public conformance matrix passes for all four shapes on both stacks.
- A v2 shape the active profile or descriptor registry cannot represent fails
  before iterator consumption, transport, or authentication; the frozen v1
  adapter passes its separate corpus. No standard shape reaches that path.
- Every public production module, including the pytest plugin, is measured by
  coverage, with the plugin actually imported in the measuring run.
- Targeted mutation testing covers sanitizer, validator, storage transaction,
  record modes, and reservation logic. The archived report has no surviving
  non-equivalent mutant; equivalent mutants require an owner-approved waiver with
  the exact source location and rationale.
- Strict typing passes on replacement modules.
- The Python 3.11 dependency floor, current dependencies on 3.11–3.14, and Linux,
  macOS, and Windows storage jobs pass.
- Both wheel and source distribution are installed and tested before a protected
  manual promotion publishes the exact tested artifacts.
- Five consecutive downstream nights hold the rollout runbook's complete
  candidate identity tuple fixed. A pre-promotion rollback rehearsal pins 0.1.1,
  restores the hashed v1 corpus, and completes offline playback.
- Documentation makes no unsupported credential-safety claim and no JSON-storage
  claim for v2, and
  every breaking change in
  [Breaking changes in 0.2](#breaking-changes-in-02) appears in the release
  notes.

Coverage and aggregate mutation percentages are supporting metrics, not proof of
security. The state-machine, persistable-constructor, and threat-model tests are
release gates in their own right.

## Appendix: downstream pilot NotebookLM

This section is the pilot consumer's integration plan. Its Phase 3 branch trial is
consumer-specific and nonblocking, but the final-candidate five-night run and
rollback rehearsal are release gates because the rollout runbook explicitly uses
them as promotion evidence.

NotebookLM is the first consumer of the v2 async engine. It requires Python 3.10
and tests that in its own fork. It must move `BearerProvider.get()` out of the
channel factory and into the per-live-call invocation preparer, since no
transport, preparation, or completion hook runs on a cassette hit.
Its checked-in profile explicitly keeps the request fields needed to distinguish
pilot calls and every response field asserted by its tests; the deny-all default
is safe but not a useful functional cassette.

Its adoption gate — it retains its custom cassette seam as an oracle and fallback
until all of the following hold:

- The new engine covers more than the single recorded `GetProject` case.
- At least one real unary-stream interaction is recorded and replayed.
- Playback demonstrably constructs no channel and mints no bearer.
- Adversarial credential-leak tests pass.
- Web and Android public suites remain behaviorally equivalent.
- Android live E2E runs as a separate CI/nightly leg.
- Five consecutive Android nightly cycles against one fixed candidate identity
  tuple complete without cassette, authentication, or unexpected-`live_started`
  regressions. The downstream integration owner records the upstream commit and
  release-artifact hashes, packaging-patch hash, downstream Python-3.10 wheel
  hash, consumer profile/config hash, and run URLs in the adoption issue. Any
  changed tuple member resets the count.

Only then should the custom seam be removed.

## Estimates and staffing

| Milestone | Phases | Estimated effort |
| --- | --- | ---: |
| Async unary/unary-stream, pilot installs by commit | 0–3 | 29–45 engineer-days |
| Sync parity; public surface complete except streaming | 0–4 | 39–60 engineer-days |
| All four shapes; public surface final | 0–5 | 64–95 engineer-days |
| `0.2` released | 0–6 | 69–107 engineer-days |

Only the last row corresponds to a published artifact. The first three are
internal checkpoints, and the pilot consumes the first by commit rather than by
version — a direct consequence of the single-release decision.

The work benefits from one core runtime architect and one security/testing
engineer, with a downstream integration owner contributing integration cases.
Parallel staffing reduces test and integration latency but does not divide the
schedule linearly because the schema, call lifecycle, and security projection are
tightly coupled.

## Resolved decisions and residual risks

### Single release: no `0.1.2`, no `0.3`

Decision: `0.2` is the only release. The intermediate patch release is dropped
and the streaming work is folded in rather than deferred.

What this buys: the library never publishes a version supporting fewer RPC shapes
than 0.1.x; there is one cassette schema major instead of two, so no user
migrates v2 → v3; and documentation, typing, and the public-surface freeze happen
once, over a final API.

What it costs, and these are the residual risks rather than settled questions:

- **The merged commits stay unpublished for 69–107 engineer-days**, including two
  replay-correctness fixes for cassettes users already hold. `main` is a usable
  source for them only through Phase 2b. From Phase 3 it carries the full
  breaking refactor — interceptors replaced, `async_recorded_channel` removed,
  strict matching by default, raw exception attributes removed, `GrpcvrError`
  gone — and between Phase 3 and Phase 5 it raises `UnsupportedRpcShapeError` for
  `stream_unary` and `stream_stream`, so for that window `main` supports two of
  the four shapes. The "never fewer shapes than 0.1.x" guarantee is about
  *published* versions and does not extend to `main`; affected users are directed
  to a `0.1.1` plus cherry-pick patch, not to `main`, once Phase 3 merges. If
  that becomes untenable, a `0.1.2` tag is a one-row change to
  [Release scope](#release-scope) and costs no phase work.
- **The `fail_under = 100` gate holds for the whole program with no release
  valve.** Every phase merges green, and green is 100% branch coverage of
  `src/grpcvcr`. Phase 1's streaming prototypes therefore live under
  `prototypes/`, outside the measured package and outside the wheel, and are
  deleted when Phase 5 supersedes them; they are proof artifacts, not shipped
  code. Phase 4 removes `pytest_plugin.py` from `[tool.coverage.run] omit` and
  removes `-p no:grpcvcr` from `Makefile` and `ci.yml`, taking that module from
  unmeasured to fully measured in one PR: bringing it to 100% is Phase 4 scope
  and is included in its estimate. Phases 3 and 5 carry the bulk of the cost,
  because their error branches — transport-factory failure, the cancel/delegate
  race, `DIRTY_UNKNOWN` commit-only retries, and every `SafeEventReason` storage
  stage — need fault injection to reach.
- **The pilot cannot pin a released version.** It installs the Phase 3 branch by
  commit, so the rollout runbook's identity tuple — not a version string — is the
  only reproducible handle on what the pilot ran.
- **The v2 event grammar is frozen at Phase 1 by prototype**, not at Phase 5 by
  implementation. If the streaming implementation discovers the model is wrong,
  the correction lands in a schema the earlier phases already validate against,
  and Phases 2a–4 carry rework. This is the single largest schedule risk created
  by combining, and Phase 1's prototype gate exists specifically to contain it.
- **One long-lived release branch.** With nothing published for the duration, the
  usual pressure-relief valve of shipping a smaller increment is unavailable.

### Python 3.10

The downstream pilot requires Python 3.10, while upstream currently requires 3.11
and Python 3.10 reaches end of life in October 2026. Decision: upstream 0.2 keeps
`requires-python >=3.11`; the consumer carries and tests its packaging patch.
Replacement modules remain source-compatible where practical, but upstream does
not advertise an untested interpreter. A downstream packaging patch updates all
of `requires-python`, classifiers, Ruff, Pyright, CI, `uv.lock`, and installation
documentation together.

### CI record-mode detection

`docs/guides/ci-testing.md` and `CHANGELOG.md` claim automatic `RecordMode.NONE`
in CI. No such code exists. Decision: Phase 0 removes the claim and 0.2 does not
add environment detection. CI chooses explicitly with
`--grpcvcr-record=none`; the library never infers policy from vendor-specific
environment variables.

### Publishing this plan to the public documentation site

Decision: this internal plan is not published. `mkdocs.yml` removes the nav entry
and sets `exclude_docs: plan/**`, so MkDocs does not emit the page at an unlinked
URL. The file remains in-repo for implementation review. Because the file is excluded
from the build set, `mkdocs build --strict` neither renders it nor validates its
links; Phase 6 adds an explicit test asserting `site/plan/` is absent after a
build, plus the separate repository-relative Markdown-link check named in that
phase. Anchors in this document are therefore verified against both
Python-Markdown and GitHub slug rules, since GitHub is the only renderer that
actually displays it.

### gRPC public API drift

The conformance suite, rather than inheritance alone, defines compatibility. No
private gRPC classes are permitted in the v2 implementation. The aio interception
chain is the largest exposure to upstream drift, because it reimplements behavior
gRPC keeps private; its differential test against a real `grpc.aio` channel is
the control.

### Pre-intercepted channels

Pre-intercepted channels can change request objects after the safe projection is
created and therefore violate matching and security assumptions. Strict custom
transport factories contractually return base channels. grpcvcr does not inspect
private gRPC wrapper types to guess whether callers complied; violating the
factory contract is outside the strict guarantee. Interceptors passed directly
to the routing API run before serialization and remain supported.

### Credential retry behavior

Credential generation and invalidation remain application-owned through opaque
per-invocation context and completion hooks.

### Client-streaming and bidi routing

Full-body matching is unavailable before request consumption. The streaming
shapes under `NEW_EPISODES` require a pre-stream selector rather than hiding eager buffering or
speculative network behavior. This decision must not be relaxed for convenience
without a separately named policy that clearly states its semantic trade-offs.

The current implementation does the opposite today — `requests = [r async for r
in request_iterator]` in `src/grpcvcr/interceptors/aio.py` and
`list(request_iterator)` in `sync.py` — destroying laziness, half-close timing,
and any generator with side effects. Convenience must not reintroduce it outside
the explicitly frozen v1 playback adapter.

### Canonicalization scope

`protobuf-python-deterministic-v1` promises stability only across the Python,
protobuf, and `upb`/pure-Python combinations in the declared compatibility
matrix. A same-runtime fixpoint is necessary but does not prove cross-language or
future-runtime canonicality. A cassette that fails on a supported reader is
rejected; no broader language-neutral claim is made.

### Advisory locking

The sidecar lock serializes cooperating grpcvcr writers. Source hashing detects
most editor/legacy-process interference but cannot close the final race with a
writer that ignores the lock. Repositories requiring hostile-writer protection
must provide external serialization; it is not a 0.2 guarantee.
