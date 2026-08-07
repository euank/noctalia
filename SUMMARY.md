# Handoff summary: ext-session-lock duplicate-output crash

## Request and observed failure

The reported crash was:

```text
17:26:03.771 [INF] [lockscreen] session lock requested
ext_session_lock_v1#74: error 3: Output is already locked.
17:26:03.814 [ERR] fatal: failed to dispatch pending Wayland events after poll: display_error=71 (Protocol error), protocol_error.interface=ext_session_lock_v1, object_id=74, code=3
```

Protocol error 3 on `ext_session_lock_v1` is `duplicate_output`: the client sent more than one
`get_lock_surface` request for an output during a session lock. The exact error text, "Output is
already locked.", comes from Smithay's ext-session-lock implementation (used by niri and other
compositors). A Wayland protocol error disconnects the whole client, which explains the fatal
main-loop error.

Relevant protocol reference:

- <https://wayland.app/protocols/ext-session-lock-v1>
- Smithay checks its lock-wide `locked_outputs` list and emits this exact error text:
  <https://docs.rs/smithay/latest/src/smithay/wayland/session_lock/lock.rs.html>

## Diagnosis

`LockScreen::syncInstances()` treated lock surfaces like ordinary per-output shell surfaces. It
removed an existing lock surface not only when its `wl_output` disappeared, but also whenever the
local `WaylandOutput` metadata was not done or temporarily lacked usable geometry. A later output
reconciliation would then create another lock surface for the same output.

That lifecycle is unsafe for ext-session-lock. Smithay remembers that the output was claimed for
the lifetime of the lock, so the second request is rejected with the reported fatal error. There
was also no independent record of outputs already claimed by the current lock; the existence of a
live entry in `m_instances` was the only duplicate guard.

## Changes made

Changed `src/shell/lockscreen/lock_screen.{h,cpp}`:

1. Added `m_claimedOutputNames`, a set of Wayland registry output names claimed during the current
   session-lock lifetime.
2. `createInstance()` inserts the output name before initializing the surface. This also prevents
   a re-entrant reconciliation from issuing the same request twice.
3. `syncInstances()` consults the claim set before creating a lock surface.
4. Once created, a lock surface is retained through temporary metadata/geometry changes. It is
   removed only when its actual `wl_output` global is gone or replaced.
5. The claim set is cleared at all lock-lifetime boundaries: new lock setup, normal unlock,
   compositor `finished`, and `resetLockState()`.

Design tradeoff: if `LockSurface::initialize()` fails after the claim is recorded for a newly
hotplugged output, the code will not retry that output until the next lock. This is intentional for
safety because after `get_lock_surface` has potentially been marshalled, retrying is ambiguous and
can kill the Wayland connection. During initial lock setup, an empty instance list calls
`resetLockState()`, which clears the claim set.

## Verification performed

- `git diff --check` passes.
- The host shell did not have `just`, so verification was started with:

  ```sh
  nix develop --command just build
  ```

- Nix created a fresh `build-debug` directory and compilation reached approximately target
  347/714 without errors. The build was interrupted by the user before completion; the only output
  seen was the repository's existing debug-build `_FORTIFY_SOURCE requires compiling with
  optimization` warning.
- No runtime Wayland/compositor reproduction was performed.
- No regression test was added. `LockScreen` is tightly coupled to live Wayland protocol objects;
  a useful automated regression would require a fake compositor or extracting the reconciliation
  state machine into a testable helper.

## Suggested next steps

1. Finish the incremental build (the Nix environment and `build-debug` are already populated):

   ```sh
   nix develop --command just build
   ```

2. Run tests:

   ```sh
   nix develop --command just test
   ```

3. Run formatting or at least a dry check on the two changed source files:

   ```sh
   nix develop --command clang-format --dry-run --Werror \
     src/shell/lockscreen/lock_screen.cpp src/shell/lockscreen/lock_screen.h
   ```

4. Reproduce on the affected Smithay compositor, ideally while changing output mode/geometry or
   disconnecting and reconnecting outputs during a pending/active lock.
5. Review the open upstream Noctalia PR #3825 before combining work. It addresses a related
   output-disconnect lockscreen crash on Hyprland, but its current patch is separate from this
   duplicate-output fix: <https://github.com/noctalia-dev/noctalia/pull/3825>.

## Additional observations (not changed)

- `LockScreen::resetLockState()` always sends `unlock_and_destroy` when `m_lock` exists, including
  while locally pending, whereas `unlock()` chooses `destroy` until the `locked` event is received.
  The protocol's pending-event race makes this subtle. It was not changed because it is outside
  the reported duplicate-output failure and deserves separate compositor-backed testing.
- Smithay's current implementation keeps `locked_outputs` in manager-wide state and visibly clears
  it on `UnlockAndDestroy`; cancellation/denial behavior may warrant separate investigation if the
  crash persists after this patch.
