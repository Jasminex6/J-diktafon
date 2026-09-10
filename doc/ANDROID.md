# Android notes & troubleshooting

Device-focused notes for running Diktafon on Android (developed and verified
against a Samsung Galaxy S24). Covers background playback/recording, the
summaries pipeline, and how to read the logs when something misbehaves.

## Background behavior (M1, D13)

| Scenario | Behavior |
| --- | --- |
| Playback + screen off | Continues (media-type foreground service, `FOREGROUND_SERVICE_MEDIA_PLAYBACK`). |
| Playback + app swiped away | Continues; the media notification (play/pause, seek bar, skip memo) stays. |
| Playback + headphone unplug / call | Pauses on focus loss (audio_service handles interruptions). |
| Recording + screen off | Continues (microphone-type foreground service, unchanged from D13). |
| Recording + app swiped away | Continues; the recording notification carries a **Stop** button that finalizes the memo through the normal pipeline. |

Both services are declared in `android/app/src/main/AndroidManifest.xml`;
controls forward into the same `TapePlayerService`/`RecordingController` the
in-app UI uses.

Samsung specifics (One UI): if the app is killed anyway in the background,
exclude Diktafon from *Settings → Battery → Background usage limits* ("Put
unused apps to sleep" is aggressive on Samsung).

## Summaries pipeline (D3)

Summaries need three things, checked in this order:

1. **Settings → Summaries** must not be "No summaries" (`summariesEnabled`).
2. The **LLM model must be downloaded** — Settings → Models → Summary. The
   default (Qwen3 1.7B, ~1.7 GB) downloads over Wi-Fi; the download progress
   mirrors into the notification area.
3. Memos only get a gist when the transcript is longer than ~117 estimated
   tokens (§6.7) — short memos are their own summary by design.

### Reading the logs

Everything below is visible in `adb logcat` (all build types):

```sh
adb logcat -s flutter | grep dk_
```

- `[dk_jobs]` — the background queue: job failures with attempt counts, and
  why summary jobs are parked:
  - `summary job(s) parked: summaries disabled in settings` → case 1 above.
  - `summary job(s) parked: LLM model not ready` → case 2 above.
- `[dk_llm]` — the LLM engine: model loads, generate durations, and crashes.
  - `model loaded in … ms` — the Qwen model loaded into llama.cpp.
  - `generate done in … ms` — one summary exchange completed.
  - `worker error: …` / `worker exited with N request(s) pending` — the
    native engine died (typically OOM or a corrupt download); pending
    summaries now fail fast and retry instead of hanging on "summarizing…".
  - `generate timed out after 5 min — restarting worker` — a wedged native
    call was killed; the queue's retry spawns a fresh worker.

A memo stuck on "summarizing…" across app restarts with none of these lines
means the job row was orphaned by a process death — the launch sweep requeues
it (`_recoverOrphans`), so it resumes on the next start.

### RAM guidance

The default 1.7B Q8 model wants ~2 GB of free RAM at load time (the app
targets 4 GB+ devices). On 6–8 GB phones (S24: 8/12 GB) it loads fine, but
heavy background apps can still push llama.cpp into an OOM — the worker
crash handling above turns that into a retry with a log line instead of a
silent hang. If summaries keep failing with `worker exited`, pick the 0.6B
model in Settings → Models.

## Performance (transcript scrolling)

The cassette screen no longer rebuilds wholesale on playback ticks; word
spans/recognizers are cached per memo and transcript JSON decodes off the
UI isolate. If scrolling regresses on a device, profile with:

```sh
flutter run --profile
# then: Performance overlay + "r" to hot-reload while scrolling the transcript
```

Known heavy paths are all in `lib/presentation/widgets/transcript_view.dart`
and `lib/presentation/screens/cassette_screen.dart`.

## Building

Requires the Android SDK with **NDK 28.2.13676358** and Java 17 (see the
README). Debug build:

```sh
flutter build apk --debug
adb install build/app/outputs/flutter-apk/app-debug.apk
```
