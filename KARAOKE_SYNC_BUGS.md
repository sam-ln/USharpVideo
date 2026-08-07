# Karaoke latency compensation — bug report

Findings from a scan of `USharpVideoPlayer` / `VideoPlayerManager` / `SyncDelayManager` and their
interaction with [USharpVideoQueue](https://github.com/sam-ln/USharpVideoQueue), targeting two
reported symptoms:

1. When somebody is still on stage or goes off stage when their song ends, *sometimes* the next
   song gets skipped.
2. The song just may not start to play for some users.

GitHub Issues are disabled on this repository, so the four issues below are recorded here. The two
queue-side issues were filed directly:

- sam-ln/USharpVideoQueue#15 — duplicate/late video-end events skip the next song
- sam-ln/USharpVideoQueue#16 — queue stalls permanently if the playback handoff is dropped

---

## Issue 1 — `CheckForVideoEnd` fires premature/false `OnVideoEnd`

**Severity: high.** Most likely single cause of both reported symptoms.

`Packages/com.arcanescripts.usharpvideo/Runtime/Scripts/VideoPlayerManager.cs:134-144`

```csharp
private void CheckForVideoEnd()
{
    if (!_initialized || !_currentPlayer.IsPlaying) return;

    float videoEndThreshold = 0.5f;
    if (_currentPlayer.GetTime() + videoEndThreshold >= _currentPlayer.GetDuration())
    {
        _currentPlayer.Stop();
        OnVideoEnd();
    }
}
```

Added in d3214df to work around AVPro not emitting `OnVideoEnd`. It runs every frame on **every
client** and synthesises a video-end from purely local state, with no validity check on the
duration, no awareness of stream mode, and no de-duplication against the real SDK event. In a
karaoke setup every client deliberately runs on a *different* playback timeline via
`localSyncOffset`, which makes all of this much worse.

### 1a. No guard against `GetDuration() <= 0`

If `GetDuration()` returns `0` while `IsPlaying` is true, the condition is `0.5 >= 0` → **always
true**, so the video is stopped and "ended" on the first frame of playback. This happens for:

- AVPro live streams / HLS that report `0` duration. Only `rtsp://`-prefixed URLs are excluded, and
  only by `USharpVideoPlayer.OnVideoEnd`, which is too late — see 1c.
- The transient window right after `Play()` before AVPro has populated media info.

`USharpVideoPlayer.OnVideoReady` already has the "is this a real, seekable video" logic
(`duration == float.MaxValue || IsInfinity || IsRTSPStream()` → `_videoSync = false`), but
`CheckForVideoEnd` never consults `_videoSync`.

### 1b. It runs on every client, on that client's karaoke-shifted timeline

`GetTime()` is the *local* position, which `SyncVideo()` deliberately offsets by `localSyncOffset`
(`USharpVideoPlayer.cs:690`). Each client independently decides the video has ended, at a different
wall-clock moment. For a listener with a 2 s karaoke delay:

- they lose the last `|localSyncOffset| + 0.5 s` of every song — the owner ends it and the queue
  advances while they are still 2 s behind, which in karaoke is the end of the vocal line;
- if their own check trips first for any reason (drift, a duration re-report), their player is
  `Stop()`ed with **no path back to playing**.

That second point is the important one. After the local `Stop()`, `USharpVideoPlayer` never calls
`Play()` again: `Update()` takes the `!_waitForSync` branch and keeps calling `SyncVideo()` →
`SetTime()` on a stopped player forever. The only escape is a *new* deserialization that changes
`_syncedVideoIdx`. The user sees a frozen/black screen for the rest of the song — symptom 2.

### 1c. `Stop()` is called *before* the event, defeating the RTSP guard

`USharpVideoPlayer.OnVideoEnd()` opens with `if (IsRTSPStream()) return;` — but `CheckForVideoEnd`
has already called `_currentPlayer.Stop()`. The stream is dead regardless of whether the guard
trips.

### 1d. Double-firing → the next song gets skipped

`VideoPlayerManager.OnVideoEnd()` is both the SDK callback (line 62) and the method
`CheckForVideoEnd` calls directly (line 142). Nothing de-duplicates them. When the underlying player
*does* emit its own end event — always for the Unity backend, usually for AVPro, and `Stop()` itself
can raise it — `receiver.OnVideoEnd()` runs twice and `SendCallback("OnUSharpVideoEnd")` fires
twice. Downstream that becomes two `RPC_OnVideoOwnerVideoEnd` calls; see
sam-ln/USharpVideoQueue#15 for why the queue's failsafe does not catch the second one.

### Suggested fix

```csharp
private void CheckForVideoEnd()
{
    if (!_initialized || _videoEndFired || !_currentPlayer.IsPlaying) return;

    float duration = _currentPlayer.GetDuration();

    // Not a finite, seekable video (stream, or media info not populated yet) — never synthesise an end.
    if (duration <= 0f || float.IsInfinity(duration) || duration == float.MaxValue) return;

    if (_currentPlayer.GetTime() + VIDEO_END_THRESHOLD >= duration)
    {
        _videoEndFired = true;
        OnVideoEnd();          // let the receiver decide; don't Stop() behind its back
    }
}
```

plus:

- a `_videoEndFired` latch, reset in `LoadURL`/`Play` and set in `OnVideoEnd()`, so the SDK event and
  the synthetic one cannot both get through;
- gate the whole check on `receiver.IsInVideoMode()` so stream mode is never affected;
- drop the unconditional `_currentPlayer.Stop()`, or move it into `USharpVideoPlayer.OnVideoEnd()`
  *after* the `IsRTSPStream()` guard;
- ideally, only synthesise the end on the client that owns the video player, so a karaoke-delayed
  listener can never end the song for the instance.

---

## Issue 2 — `SyncVideo`/`ForceSyncVideo` seek to the end of the video when `_networkTimeVideoStart == 0`

**Severity: high.** Directly triggered by the karaoke slider.

`USharpVideoPlayer.cs:685-722`

```csharp
public void ForceSyncVideo()
{
    if (IsInVideoMode())
    {
        float offsetTime = Mathf.Clamp(
            (Networking.GetServerTimeInMilliseconds() - _networkTimeVideoStart) / 1000f + localSyncOffset,
            0f, _videoPlayerManager.GetDuration());

        float syncNudgeTime = Mathf.Max(0f, offsetTime - 1f);
        _videoPlayerManager.SetTime(syncNudgeTime);
        _videoPlayerManager.SetTime(offsetTime);
    }
}
```

`_networkTimeVideoStart == 0` is the sentinel meaning "the owner has not started a video yet". It is
set by `StopVideo()` (line 560) and again in `PlayVideoInternal` (line 626), and it stays `0` for the
entire load — which with the queue's `waitSecondsBeforePlayback` and YouTube resolution is routinely
5–20 seconds.

But `0` is not treated as a sentinel here. It is subtracted as if it were a real timestamp, so
`(GetServerTimeInMilliseconds() - 0) / 1000f` is an arbitrary large value (server time is a big
wrapping ms counter). The `Mathf.Clamp` then pins `offsetTime` to `GetDuration()` when server time
is positive, or `0` when it is negative. Neither is a meaningful position.

Consequences:

- **`ForceSyncVideo` seeks the video to its very last frame.** `SyncDelayManager.OnSliderChange`
  (`Runtime/Examples/Scripts/SyncDelayManager.cs:29-39`) calls `ForceSyncVideo()` on *every* slider
  value change, i.e. continuously while the user drags. So a user adjusting their karaoke delay
  while a song is loading, or during the inter-song gap, jumps their player to `duration`.
- Combined with Issue 1, `CheckForVideoEnd` then sees `GetTime() + 0.5 >= GetDuration()` on the very
  next frame and stops the video and fires the end event. **If that user is the video player owner —
  the singer — their own song is skipped the moment it starts.**
- Even with a valid `_networkTimeVideoStart`, the same thing happens if the slider is moved in the
  last `0.5 s + |offset|` of a song: dropping the delay back to 0 moves `offsetTime` forward past the
  `CheckForVideoEnd` threshold.
- `SyncVideo()` (line 685) has the identical clamp and is called from `Update()` → `SyncVideoIfTime()`
  every `syncFrequency` (8 s default) for the owner unconditionally, and for anyone with
  `_waitForSync == false`. `_lastSyncTime` is global and never reset per video, so whether the tick
  lands in the window between `Play()` and `OnVideoStart` (where `_networkTimeVideoStart` is still
  `0`) is essentially random — which is exactly the reported "*sometimes*".

### Suggested fix

Add an explicit "no start time known" guard shared by both methods, and bail out rather than
clamping:

```csharp
private bool HasVideoStartTime() => _networkTimeVideoStart != 0;

public void ForceSyncVideo()
{
    if (!IsInVideoMode() || !HasVideoStartTime()) return;

    float duration = _videoPlayerManager.GetDuration();
    if (duration <= 0f) return;
    ...
}
```

A separate `bool _hasVideoStartTime` flag would be more robust than the magic `0`, since
`GetServerTimeInMilliseconds()` can legitimately return `0`. `SyncDelayManager.OnSliderChange` should
also debounce (call `ForceSyncVideo` on slider release, or at most a few times a second) instead of
issuing two `SetTime` calls per drag frame.

---

## Issue 3 — `localSyncOffset` leaks into the synced timebase via pause, seek and reload

**Severity: medium.** Corrupts the shared timeline for everyone, not just the offset user.

`localSyncOffset` is meant to be a purely local view offset. It is applied when *reading* the network
time (`SyncVideo`, `ForceSyncVideo`), but the three places that *write* `_networkTimeVideoStart` from
the local playback position never subtract it back out. Every round trip therefore bakes the offset
into the synced value.

**Pause** — `USharpVideoPlayer.cs:432-442`:

```csharp
if (_locallyPaused)
{
    if (IsInVideoMode())
        _networkTimeVideoStart = Networking.GetServerTimeInMilliseconds() - (int)(_videoPlayerManager.GetTime() * 1000f);
}
```

`GetTime()` is already offset-shifted. On resume `SyncVideo()` computes
`GetTime() + localSyncOffset`, so the video jumps back by `|localSyncOffset|` on every unpause. On
the owner this value is serialized, so **every pause/resume cycle rewinds the video for the whole
instance** by the owner's local delay. In a karaoke world where the host pauses between singers this
compounds.

Secondary problem in the same block: this is a `[UdonSynced]` field being written by non-owners every
frame while paused. That corrupts the `_networkTimeVideoStart == 0` sentinel that `OnVideoReady`
depends on (see Issue 4) and makes `OnDeserialization`'s `_networkTimeVideoStart != _localNetworkTimeStart`
seek detection (line 509) fire spuriously.

**Seek** — `SeekTo` (line 869) sets `_networkTimeVideoStart = ServerTime - newTargetTime` and then
calls `SyncVideo()`, which lands the owner at `newTargetTime + localSyncOffset`. Scrubbing with a
delay set therefore lands short of where the user clicked.

**Reload** — `Reload()` (line 1165) sets `_videoTargetStartTime = GetVideoManager().GetTime()`, the
offset-shifted position; `OnVideoStart` then rebases `_networkTimeVideoStart` from it, rewinding
everyone by the reloading owner's offset. `EnsureCorrectUrlLoaded()` calls `Reload()`, so this fires
on the automatic URL-mismatch recovery path too.

**Loop** — `OnVideoLoop()` (line 414) sets `_localNetworkTimeStart = _networkTimeVideoStart = now` on
**every** client, including non-owners, again writing a synced field from a non-owner and ignoring
the offset.

### Suggested fix

Normalise to the network timeline whenever writing `_networkTimeVideoStart`:

```csharp
// while paused
_networkTimeVideoStart = Networking.GetServerTimeInMilliseconds()
                       - (int)((_videoPlayerManager.GetTime() - localSyncOffset) * 1000f);
```

and the same `- localSyncOffset` correction in `Reload()`'s `_videoTargetStartTime`. Guard the paused
block and `OnVideoLoop`'s write with `if (Networking.IsOwner(gameObject))` so non-owners never mutate
synced state.

---

## Issue 4 — Non-owner can be left with a loaded video that never plays

**Severity: medium-high.** Direct match for symptom 2.

`USharpVideoPlayer.OnVideoReady`, lines 232-264. For a non-owner where `_ownerPlaying` is false:

```csharp
#if USE_SERVER_TIME_MS
    if (_networkTimeVideoStart == 0)
#else
    if (_videoStartNetworkTime == 0f || _videoStartNetworkTime > GetNetworkTime() - _videoPlayerManager.GetDuration())
#endif
    {
        _waitForSync = true;
        SetStatusText("Waiting for owner sync...");
    }
    else
    {
        _waitForSync = false;
        SyncVideo();
        SetStatusText("");
        ...
    }
```

Two problems:

**4a. The `else` branch never calls `Play()`.** It sets `_waitForSync = false` and syncs the time, but
the player is not started. `Update()` then takes the `!_waitForSync` branch and only ever calls
`SyncVideoIfTime()` — there is no code path that starts playback again. The only rescue is
`OnDeserialization`'s tail (line 520-531), which requires *another* serialization from the owner to
arrive; if the owner is idle, none does. The client sits on a loaded, silent video indefinitely.

**4b. The staleness guard only exists in the non-default compile path.** `USE_SERVER_TIME_MS` is
defined at the top of the file (line 2), so the active branch checks *only* `_networkTimeVideoStart == 0`.
The `#else` branch additionally validates that the start time is recent enough to be plausible
(`_videoStartNetworkTime > GetNetworkTime() - duration`). With a stale non-zero `_networkTimeVideoStart` —
which Issue 3 shows non-owners can produce locally — the active path takes the `else` branch and hits 4a.

**4c. `_waitForSync` has no watchdog.** It is cleared only in `Update()` when `_ownerPlaying` becomes
true. `_ownerPlaying` is a synced field, and `QueueSerialize()` is called only on state changes —
there is no periodic re-serialization. A single dropped or coalesced sync leaves the client showing
"Waiting for owner sync..." permanently. The queue works around this with its
`waitSecondsBeforePlayback` setting, whose tooltip already says "It's recommended to keep this value
above 2 seconds to allow networked data in USharpVideoPlayer to settle."

### Suggested fix

- Call `_videoPlayerManager.Play()` in the `else` branch, since the intent is "we know where we
  should be, start playing there".
- Port the staleness check into the `USE_SERVER_TIME_MS` path:
  `if (_networkTimeVideoStart == 0 || (Networking.GetServerTimeInMilliseconds() - _networkTimeVideoStart) / 1000f > _videoPlayerManager.GetDuration())`.
- Add a watchdog: when `_waitForSync` has been true for more than a few seconds, have the client
  request a fresh serialization from the owner (or have the owner re-serialize periodically while
  `_ownerPlaying` is true), so one lost packet is not permanent.
