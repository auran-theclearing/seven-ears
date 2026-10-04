# Adapting Seven Ears to Your Companion Stack

Seven Ears is a set of local command-line tools. It does not import an agent SDK,
call an LLM, register an OpenClaw plugin, or require Discord. A companion harness
only needs to make an audio file available, run the appropriate Python command,
and place the result in the companion's context.

The analysis is harness-agnostic; attachment retrieval, tool registration,
prompt placement, access control, and artifact retention belong to the enclosing
system. An MCP-only system needs a thin wrapper because this repository does not
ship an MCP server.

## A plain-language prompt you can use

> Please inspect my companion setup and connect this repository as a local audio
> analysis tool. Do not rewrite the analyzer for my harness. Find how my system
> stores incoming attachments, runs local commands or tools, and gives tool output
> back to the companion. Use `seven_ears_card.py` for short spoken voice notes and
> `seven_ears_music.py` for music. Keep audio and generated artifacts private,
> preserve the distinction between measured acoustic evidence and interpretation,
> and verify the integration with synthetic or non-private audio before using a
> personal recording. If my system only accepts MCP tools, build a narrow local
> wrapper around the existing CLI and return its stdout without inventing new
> measurements.

## Choose the listening tool

| Need | Entry point | Result |
| --- | --- | --- |
| Spoken voice note | `seven_ears_card.py` | Human-readable console/Discord card or structured JSON; optional local Whisper transcript. |
| Whole-song measurements | `seven_ears_music.py` | Music card or JSON covering time-body, color, pulse, and dynamics while naming unsupported claims. |
| Waveform and spectrogram | `seven_ears_visual.py` | A generated PNG path. |
| Sequential music encounter | `seven_ears_first_listen.py` | JSON packets and a private, restart-safe SQLite journal; see `docs/FIRST_LISTEN.md`. |
| Relative-to-speaker voice layer | `normalize.py` or `--speaker` on the spoken card | A rolling local baseline; this is private biometric-like household data and should be opt-in. |
| Manual spoken-voice workbench | `dashboard_server.py` | Localhost web UI, not a public service or agent API. |

Start with one tool. Most companion integrations only need the spoken card, the
music card, or both.

## Smallest useful integrations

### Spoken voice notes

If the chat platform already supplies a transcript, pass it through rather than
running speech-to-text again:

```sh
python3 seven_ears_card.py /private/path/note.ogg \
  --transcript "The transcript supplied by the platform" \
  --transcript-source manual \
  --json
```

The accepted transcript-source labels are `discord`, `openclaw`, `manual`,
`stt_api`, `stt_whisper`, and `none`. Choose the truthful provenance label; do
not call platform text `stt_whisper` merely because Whisper is installed.

When no transcript exists, use local faster-whisper:

```sh
python3 seven_ears_card.py /private/path/note.ogg \
  --stt whisper --whisper-model base.en --json
```

The first use may download the selected Whisper model. `ffmpeg` must be on
`PATH`. Use `--format discord` instead of `--json` when the integration wants a
ready-to-display card rather than structured data.

### Music

```sh
python3 seven_ears_music.py /private/path/song.mp3 --json
```

Music Edition uses NumPy and the standard library for analysis, with `ffmpeg`
to decode compressed/non-analysis-rate input. It does not transcribe lyrics or
claim emotion, genre, vocal presence, key, or melody. Give verified lyrics to the
companion separately if they matter; do not relabel speech transcription over a
full mix as reliable lyric extraction.

### First Listen

First Listen is a stateful protocol rather than a single analyzer call. Its
commands are `prepare`, `next`, `note`, `finish`, and `journal`; follow
**[Seven Ears: First Listen](docs/FIRST_LISTEN.md)** for the exact sequence. The
wrapper must:

- keep the SQLite file outside the repository in a private directory;
- return only the current command's JSON output to the listening companion;
- require the companion's nonblank note and acknowledgement token before
  advancing;
- replay an unacknowledged packet rather than skipping it after a retry;
- avoid exposing the source name, database, later packets, or whole-song card
  before `finish` during a genuine sequential encounter.

This is a cooperative information boundary, not a sandbox. A harness with broad
filesystem access can bypass it.

## Wrapper contract

Whether the adapter is a shell tool, Python function, bot command, job worker, or
local MCP server, keep the boundary narrow:

1. **Input:** an authorized local audio path plus explicit options. Do not accept
   arbitrary server paths from untrusted chat text.
2. **Execution:** invoke the checked-out script with an argument array and a
   timeout. Do not construct a shell command by concatenating filenames or
   transcripts.
3. **Output:** return stdout as either the selected card or parsed JSON. Keep
   stderr and exit status available for diagnosis; never turn a failed analysis
   into a confident card.
4. **Context placement:** label the result as acoustic evidence. Let the
   companion respond naturally rather than forcing it to repeat every metric.
5. **Cleanup:** follow the surrounding platform's retention policy for the
   original attachment and generated image. The analyzers clean up their own
   temporary conversions, not the source file handed to them.

For MCP, one tool per major operation is easier to authorize and validate than a
single free-form command tool. A minimal wrapper might expose
`analyze_spoken_audio`, `analyze_music`, and `render_audio_visual`; add the five
First Listen operations only if that workflow is actually wanted.

## Platform patterns

### OpenClaw or another tool-using companion runtime

Register a local tool or skill that resolves the platform's downloaded attachment
to a local path, runs the selected CLI, and returns stdout. Keep the analysis
result in the same private conversation scope as the attachment. No OpenClaw
plugin is required by the analyzer itself.

### Claude Code, Codex, or another coding-agent CLI

The agent can run the documented commands directly when it has filesystem and
process access. Give it this repository plus the audio path and ask it to read the
card as evidence before responding. A project instruction can establish when to
use the tool, but should not paste private card output into durable repository
instructions.

### A custom Discord, Telegram, or WhatsApp bot

Download the attachment into a private temporary directory, run the analyzer in a
worker rather than blocking the message event loop, and pass the result to the
existing companion turn. Delete or retain the downloaded source according to an
explicit household policy. The analyzer does not supply chat authorization or a
message handler.

### A remote or containerized companion

Mount or securely transfer only the intended attachment. Install `ffmpeg`, NumPy,
and optionally faster-whisper in the analysis environment. Remember that a
container's writable filesystem may be ephemeral: First Listen journals and
opt-in speaker baselines need persistent, private storage if they should survive
redeploys.

## Evidence and privacy boundaries

- Acoustic measurements describe the recording as well as the speaker: room,
  microphone, compression, and background noise matter.
- The output must not be promoted into emotion detection, lie detection,
  diagnosis, or certainty about intent.
- Supplying a transcript does not make it acoustically verified. Preserve its
  source label and check soft or consequential words with the human.
- Local analysis does not control what the surrounding harness logs or sends to
  a cloud model. Inspect that system separately.
- Per-speaker baselines and First Listen journals should stay outside the repo,
  out of logs, and scoped to the people who consented to them.
- `dashboard_server.py` binds to `127.0.0.1` by default. Do not expose it to a
  network without adding real authentication, transport security, and deployment
  hardening.

## Verification checklist

Before calling an integration complete:

1. Install the Python requirements and confirm `ffmpeg` is available.
2. Run the repository regression suite:

   ```sh
   python3 -m unittest -v
   ```

3. Exercise every exposed wrapper operation with synthetic or non-private audio.
4. Check a filename containing spaces and punctuation; it must be passed as one
   argument, not interpolated into a shell command.
5. Confirm malformed media returns an error and no fabricated card.
6. Confirm private audio, JSON, images, SQLite journals, and speaker baselines do
   not enter source control or a public log.
7. Restart the harness and verify any intentionally persistent First Listen or
   baseline state remains private and usable.
8. Read one result in the real companion context and verify that measurements are
   presented as evidence, with interpretation left to the relationship.

The portable organ is the command-line evidence layer. Adapt the attachment and
context wiring to the body your companion already lives in.
