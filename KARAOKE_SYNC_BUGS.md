# Karaoke latency compensation — bug report

Findings from a scan of `USharpVideoPlayer` / `VideoPlayerManager` / `SyncDelayManager` and their
interaction with [USharpVideoQueue](https://github.com/sam-ln/USharpVideoQueue), targeting two
reported symptoms:

1. When somebody is still on stage or goes off stage when their song ends, *sometimes* the next
   song gets skipped.
2. The song just may not start to play for some users.

GitHub Issues are disabled on this repository, so the USharpVideo-side findings are recorded here.
The queue-side findings are filed as issues:

- sam-ln/USharpVideoQueue#15 — next song skipped: the duplicate-end failsafe is incomplete, error path has none
- sam-ln/USharpVideoQueue#16 — queue stalls permanently if the playback hand-off is dropped
- sam-ln/USharpVideoQueue#17 — `QueueArray.Clear` off-by-one leaves a stale entry

Severities below reflect a second, verification-focused pass. Two VRChat SDK behaviours were
confirmed from community/SDK references during that pass and are load-bearing for Issue 2:

- `Networking.GetServerTimeInMilliseconds()` returns a **large positive value for roughly half the
  players in an instance and a negative value for the rest**.
- `Networking.GetServerTimeInSeconds()` is a large, continuously increasing value that eventually
  wraps; VRChat's own guidance is not to subtract raw server-time values.

---

## Issue 2 (highest confidence) — `SyncVideo`/`ForceSyncVideo` seek to the end of the video when `_networkTimeVideoStart == 0`

This is the strongest single explanation for both symptoms, and the confirmed server-time behaviour
above explains why it hits "*some* users" and only "*sometimes*".

`USharpVideoPlayer.cs:685-722`

```csharp
public void SyncVideo()   // and ForceSyncVideo(), identical clamp
{
    if (IsInVideoMode())
    {
        float offsetTime = Mathf.Clamp(
            (Networking.GetServerTimeInMilliseconds() - _networkTimeVideoStart) / 1000f + localSyncOffset,
            0f, _videoPlayerManager.GetDuration());
        ...
    }
}
```

The normal case is safe because it subtracts **two** server-time values: `GetServerTimeInMilliseconds()`
and `_networkTimeVideoStart` are both large same-origin numbers, so the large (per-player, half
positive / half negative) component cancels and the result is a small elapsed time.

`_networkTimeVideoStart == 0` is the sentinel for "the owner has not started a video yet". It is set to
`0` in `StopVideo()` (line 560) and in `PlayVideoInternal` (line 626), and only replaced with a real
value in `OnVideoStart` (line 280). Through the whole load — which with the queue's
`waitSecondsBeforePlayback` plus URL resolution is routinely 5–20 s — it stays `0`.

With the second operand `0`, the cancellation no longer happens and the **raw** server time is used:

- On the ~half of players where `GetServerTimeInMilliseconds()` is positive, `offsetTime` clamps to
  `GetDuration()` → the video is seeked to its **last frame**.
- On the other ~half it clamps to `0` → seeked to the start (mostly harmless).

Consequences:

- `SyncDelayManager.OnSliderChange` (`Runtime/Examples/Scripts/SyncDelayManager.cs:29-39`) calls
  `ForceSyncVideo()` on **every** slider value change, i.e. continuously while a user drags the
  karaoke-delay slider. A user adjusting their delay during a load, or in the inter-song gap, jumps
  their player to `duration` — and if they are the video-player owner (the singer), the video then
  immediately satisfies the end condition and their own song ends the moment it starts.
- `SyncVideo()` is also called from `Update()` → `SyncVideoIfTime()` every `syncFrequency` (8 s) for
  the owner unconditionally, and for anyone with `_waitForSync == false`. `_lastSyncTime` is global
  and never reset per video, so whether a sync tick lands in the `Play()`→`OnVideoStart` window
  (where `_networkTimeVideoStart` is still `0` but `GetDuration()` is already valid) is effectively
  random per song — which is exactly the reported "*sometimes*", and lands on a different subset of
  players each time, matching "*some* users".

Note `GetDuration()` is the second clamp bound, so this only fires once the video has a real duration
(post-`OnVideoReady`); before that, `Clamp(x, 0, 0) == 0` makes it a harmless seek-to-0.

### Suggested fix

Guard both methods on a real start time instead of clamping a sentinel:

```csharp
private bool HasVideoStartTime() => _networkTimeVideoStart != 0;   // or a dedicated bool flag

public void ForceSyncVideo()
{
    if (!IsInVideoMode() || !HasVideoStartTime()) return;
    float duration = _videoPlayerManager.GetDuration();
    if (duration <= 0f) return;
    ...
}
```

A dedicated `bool _hasVideoStartTime` is more robust than the magic `0`, since
`GetServerTimeInMilliseconds()` can itself legitimately be near `0`. `SyncDelayManager.OnSliderChange`
should additionally debounce — sync on slider *release*, or a few times a second at most — instead of
firing two `SetTime` calls per drag frame.

---

## Issue 1 (high) — `VideoPlayerManager.CheckForVideoEnd` fires premature / false `OnVideoEnd`

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
client** and synthesises an end from purely local state.

### 1a. No `GetDuration() <= 0` guard (affects streams most)

If `GetDuration()` returns `0` while `IsPlaying` is true, the condition is `0.5 >= 0` → **always
true**, so the video is stopped and "ended" on the first frame. This bites:

- AVPro live streams / HLS that report `0` duration. Only `rtsp://`-family URLs are excluded, and only
  by `USharpVideoPlayer.OnVideoEnd` — too late (see 1c). This is more a general stream-playback bug
  than a karaoke one (karaoke uses the Unity backend with known-duration videos), but it is real.
- The brief window right after `Play()` before media info is populated.

`USharpVideoPlayer.OnVideoReady` already computes whether the video is finite/seekable
(`duration == float.MaxValue || IsInfinity || IsRTSPStream()` → `_videoSync = false`), but
`CheckForVideoEnd` never consults `_videoSync`.

### 1b. Runs on every client's karaoke-shifted timeline

`GetTime()` is the local position, which `SyncVideo()` deliberately offsets by `localSyncOffset`
(`USharpVideoPlayer.cs:690`). Each client independently reaches the end threshold at a different
moment. A listener with a delay set loses the last `|localSyncOffset| + 0.5 s` of every song. And
after the local `Stop()`, `USharpVideoPlayer` has **no path back to `Play()`** for that same song —
`Update()` takes the `!_waitForSync` branch and only ever calls `SyncVideo()`/`SetTime()` on a
stopped player. The client recovers on the *next* song (a new `_syncedVideoIdx` triggers
`StartVideoLoad` via `OnDeserialization`), so the practical impact is "loses the tail of the current
song", escalating to "misses most of the song" if 1a/Issue 2 trip the threshold early.

### 1c. `Stop()` runs before the event, defeating the RTSP guard

`USharpVideoPlayer.OnVideoEnd()` opens with `if (IsRTSPStream()) return;`, but `CheckForVideoEnd` has
already called `_currentPlayer.Stop()`. The stream is dead regardless.

### 1d. No de-duplication latch (contributes to the queue skip)

`VideoPlayerManager.OnVideoEnd()` is both the SDK callback (line 62) and the method `CheckForVideoEnd`
calls directly (line 142), with nothing to de-duplicate them. Any path that lets both through fires
`SendCallback("OnUSharpVideoEnd")` twice, which downstream becomes two `RPC_OnVideoOwnerVideoEnd`
calls — one of the candidate duplicate-event sources in sam-ln/USharpVideoQueue#15. (Whether the
SDK's `Stop()` re-raises `OnVideoEnd` was not confirmed, so this is a latent robustness gap, not a
proven always-double.)

### Suggested fix

```csharp
private void CheckForVideoEnd()
{
    if (!_initialized || _videoEndFired || !_currentPlayer.IsPlaying) return;
    if (!receiver.IsInVideoMode()) return;                 // never touch stream mode

    float duration = _currentPlayer.GetDuration();
    if (duration <= 0f || float.IsInfinity(duration) || duration == float.MaxValue) return;

    if (_currentPlayer.GetTime() + VIDEO_END_THRESHOLD >= duration)
    {
        _videoEndFired = true;
        OnVideoEnd();          // let the receiver decide; don't Stop() behind its back
    }
}
```

plus: a `_videoEndFired` latch reset in `LoadURL`/`Play` and set in `OnVideoEnd()`; drop the
unconditional `Stop()` (or move it into `USharpVideoPlayer.OnVideoEnd()` *after* the `IsRTSPStream()`
guard); and ideally only synthesise the end on the client that owns the video player, so a
karaoke-delayed listener can never end the song for the instance.

---

## Issue 4c (medium) — `_waitForSync` has no watchdog

`USharpVideoPlayer` non-owners set `_waitForSync = true` and show "Waiting for owner sync..." until
`_ownerPlaying` becomes true in `Update()` (lines 447-457). `_ownerPlaying` is a `[UdonSynced]` field
that is only re-sent on state changes (`QueueSerialize()` is not periodic). A single dropped or
coalesced deserialization therefore leaves a client displaying "Waiting for owner sync..."
indefinitely — a direct match for "the song just may not start to play for some users". The queue
already works around this with its `waitSecondsBeforePlayback` setting, whose tooltip says to keep it
"above 2 seconds to allow networked data in USharpVideoPlayer to settle."

Suggested: when `_waitForSync` has been true for more than a few seconds, have the client request a
fresh serialization from the owner (or have the owner re-serialize periodically while `_ownerPlaying`
is true), so one lost packet is not permanent.

### Related, lower-impact (4a/4b)

In `OnVideoReady`, the non-owner `else` branch (lines 253-262) syncs the time but never calls
`Play()`, and the staleness check that would keep a client out of that branch
(`_videoStartNetworkTime > GetNetworkTime() - duration`) exists only in the inactive `#else` compile
path, not in the active `USE_SERVER_TIME_MS` path (which checks only `_networkTimeVideoStart == 0`).
This only triggers when `_ownerPlaying` is false but a start time exists — i.e. late-joining into a
finished/paused video — so it is niche during active karaoke (where `_ownerPlaying` is true and the
`if (_ownerPlaying)` branch does call `Play()`). Worth porting the staleness guard into the active
path and calling `Play()` in the `else` branch for correctness.

---

## Issue 3 (medium-low) — `localSyncOffset` leaks into the timebase; non-owners write a synced field

`localSyncOffset` is meant to be a purely local *view* offset — applied when reading network time
(`SyncVideo`/`ForceSyncVideo`) — but several places that write `_networkTimeVideoStart` from the local
playback position never subtract it back out, and some run on non-owners.

- **Paused update** (`USharpVideoPlayer.cs:432-442`) writes `_networkTimeVideoStart =
  ServerTime - GetTime()` every frame while `_locallyPaused`, with **no owner check**. `GetTime()` is
  already offset-shifted, and `_networkTimeVideoStart` is `[UdonSynced]`. A non-owner cannot push the
  value to others, but it corrupts their **local** copy, which then makes `OnDeserialization`'s
  `_networkTimeVideoStart != _localNetworkTimeStart` seek-detection (line 509) misfire and their
  `SyncVideo` land `localSyncOffset` off. Guard this block with `if (Networking.IsOwner(gameObject))`
  and normalise: `ServerTime - (int)((GetTime() - localSyncOffset) * 1000f)`.
- **`SeekTo`** (line 869) sets the start time then calls `SyncVideo()`, which lands the seeker at
  `clickedTime + localSyncOffset` — scrubbing overshoots/undershoots by the delay for offset users.
- **`Reload`** (line 1165) sets `_videoTargetStartTime = GetTime()` (offset-shifted); `OnVideoStart`
  then rebases the start time from it. `EnsureCorrectUrlLoaded()` calls `Reload()`, so the automatic
  URL-mismatch recovery path inherits this too.
- **`OnVideoLoop`** (line 414) writes `_localNetworkTimeStart = _networkTimeVideoStart = now` on
  **every** client including non-owners.

Note: in the intended karaoke model the offset is set by *listeners*, while the *owner* (singer) runs
at offset 0, so the earlier framing of an instance-wide rewind on every owner pause was overstated —
with an offset-0 owner those synced writes carry no offset. The real, reproducible harm is local
desync for the offset users themselves plus non-owners mutating a synced field. Normalising the
offset out on every write and gating the writes on ownership fixes all four.

---

## Summary of severities

| # | Location | Symptom | Confidence |
|---|----------|---------|------------|
| 2 | `SyncVideo`/`ForceSyncVideo` seek-to-end on `_networkTimeVideoStart == 0` | premature skip + "some users" | **high — mechanism confirmed against VRChat server-time behaviour** |
| 1 | `CheckForVideoEnd` premature/false end (no duration guard, offset timeline, no latch) | premature end, tail loss, feeds queue skip | high |
| 4c | `_waitForSync` never times out | "song never starts for some users" | medium |
| 3 | `localSyncOffset` leaks into timebase; non-owners write synced field | local desync for offset users | medium-low |

Queue-side: sam-ln/USharpVideoQueue#15 (skip on duplicate/late end), #16 (permanent stall on dropped
hand-off / locked player), #17 (`Clear` off-by-one).
