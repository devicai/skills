# Speech-to-Text (Voice Input)

`@devicai/ui` can add voice input to the chat prompt box. Audio is recorded in
the browser, transcribed through the Devic `/whisper` endpoint, and the result
fills the input for the user to review before sending. The transcript is linked
to the sent message via a `transcriptId`, so the conversation keeps a reference
to the original audio (and you can tell when the user edited the text).

This is the UI counterpart of the [`/api/v1/whisper`](../devic-api/whisper.md)
endpoint.

## Requirements

- A secure context (HTTPS or `localhost`) — browsers only grant microphone
  access over secure origins.
- A browser with `navigator.mediaDevices.getUserMedia` and `MediaRecorder`
  (all modern browsers). When unsupported, the microphone control is hidden.
- A configured Devic API key (the same one the chat uses).

## Quick start (default prompt box)

Enable the microphone in the `ChatDrawer` prompt box with the
`enableSpeechToText` option. Optionally pass a `speechLanguage` hint.

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    enableSpeechToText: true,
    speechLanguage: 'es', // optional ISO-639-1 hint
  }}
/>
```

What the user sees:

1. A **microphone button** next to the input.
2. Pressing it starts recording and shows a live **equalizer animation** with a
   timer, plus **pause/resume**, **confirm** and **cancel** controls.
3. On **confirm**, the recording stops and a **loading state** ("Transcribing…")
   is shown while the audio is sent to `/whisper`.
4. The transcribed text **fills the input** for review/editing.
5. The user sends it with the **normal send button**.

The `transcriptId` returned by the transcription is attached automatically to
the message when sent. If the user clears the input entirely, the link is
dropped so a fresh message isn't attributed to the previous transcription.

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enableSpeechToText` | `boolean` | `false` | Show the microphone control in the prompt box |
| `speechLanguage` | `string` | — | ISO-639-1 language hint passed to Whisper (e.g. `'es'`, `'en'`) |
| `speechAutoStop` | `boolean` | `true` | Auto-confirm the recording after a short silence, once speech was detected (see [Auto-stop](#auto-stop-on-silence)). Set `false` to require pressing confirm |
| `speechAutoStopCountdownMs` | `number` | `1000` | Duration of the auto-stop circular countdown around the confirm button |
| `speechHandoff` | `boolean` | `false` | Enable the hands-free conversation loop (see [Hands-free mode](#hands-free-handoff-mode)). Requires `@devicai/ui` ≥ 0.20.0 |
| `speechHandoffSendDelayMs` | `number` | `1000` | In hands-free mode, the delay from the transcription being ready to the message being auto-sent (the cancellable pending window). Requires ≥ 0.21.0 |

## Auto-stop on silence

By default (`speechAutoStop: true`) the recording **stops and transcribes by
itself** once the user goes quiet — they don't have to press confirm. The
behaviour is robust to ambient noise:

- It only arms **after real speech is detected** (a quiet room never triggers
  it on its own).
- The silence threshold is **adaptive**: it's a fraction of the loudest voice
  heard in the recording, so a loud speaker and a quiet one calibrate
  differently.
- When silence is detected, an **inverted circular countdown** drains around
  the confirm button (`speechAutoStopCountdownMs`, default 1s). Talking again
  cancels it; if it completes, the audio is transcribed.

Set `speechAutoStop: false` to require an explicit confirm press instead.

## Hands-free (handoff) mode

> Requires `@devicai/ui` ≥ 0.20.0.

`speechHandoff: true` turns the mic into a **hands-free conversation loop**: the
user speaks, the message is sent automatically, and once the assistant replies
the mic re-opens for the next turn — no clicking between turns.

```tsx
<ChatDrawer
  assistantId="my-assistant"
  options={{
    enableSpeechToText: true,
    speechHandoff: true, // opt-in; the mic starts the loop
  }}
/>
```

### The loop

```
[listen] --silence/auto-stop--> [transcribe] --empty text--> EXIT handoff
                                     | (non-empty)
                                     v
                  [pending 1s + animation] --click/keystroke--> EXIT (text stays in input)
                                     | (no interaction)
                                     v
                        [auto-send] --> [waiting for reply] --reply done--> back to [listen]
```

1. Pressing the **mic** enters the loop (the mic *is* the entry — there's no
   separate toggle). A **"Hands-free on"** status bar appears with a Stop (✕).
2. The user talks; the silence auto-stop transcribes the audio.
3. A short **pending countdown** (its own draining-ring animation around a send
   icon, with a preview of the text) precedes sending. Its length is
   `speechHandoffSendDelayMs` (default 1s, ≥ 0.21.0).
4. If the user **clicks or types** during that countdown, the auto-send is
   cancelled and the loop exits — the text stays in the input for manual edit.
5. Otherwise the message is **sent automatically**; the loop stays active
   ("Hands-free · waiting for reply").
6. When the assistant finishes, **listening re-activates** for the next turn.

### How the loop ends

- A **silent / empty-transcript turn** (the user said nothing meaningful).
- **Inactivity**: the mic re-opens but no speech is detected within ~6s.
- A **click or keystroke** during the pending countdown.
- The user presses **Stop (✕)** in the hands-free status bar.

### Notes

- Handoff is built on top of auto-stop, so `speechAutoStop` must stay enabled
  (it is by default).
- The loop drives the **built-in** prompt box. With a `customPromptBox` you own
  the UI — replicate the loop yourself using `useSpeechRecording` (its
  `onAutoStop` callback and `speechDetected` flag) plus `isLoading`.
- **Styling:** the hands-free bar and pending panel ship in
  `@devicai/ui/styles.css`. If the bar renders unstyled (plain text, no
  background), the app is loading an **older `styles.css` than its JS** — make
  sure the package is on ≥ 0.20.0, the `import '@devicai/ui/styles.css'` is
  present, and the bundler cache is cleared (e.g. Vite `node_modules/.vite`).

## Custom prompt box

If you replace the input with `customPromptBox`, you can drive transcription
yourself. The props include `transcribeAudio` and a `sendMessage` that accepts a
`transcriptId`.

```tsx
import { ChatDrawer, CustomPromptBoxProps } from '@devicai/ui';
import { useState } from 'react';

function VoicePromptBox({ sendMessage, transcribeAudio, isLoading }: CustomPromptBoxProps) {
  const [text, setText] = useState('');
  const [transcriptId, setTranscriptId] = useState<string | undefined>();

  // Transcribe a recording (binary). You provide the Blob from your own
  // MediaRecorder logic (or use the useSpeechRecording hook below).
  const onRecordingReady = async (audio: Blob) => {
    const { transcriptId, text } = await transcribeAudio(audio, { language: 'es' });
    setText(text);
    setTranscriptId(transcriptId);
  };

  const handleSend = () => {
    if (!text.trim()) return;
    sendMessage(text.trim(), undefined, { transcriptId });
    setText('');
    setTranscriptId(undefined);
  };

  return (
    <div style={{ display: 'flex', gap: 8, padding: 8 }}>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && handleSend()}
        placeholder="Ask anything..."
        disabled={isLoading}
        style={{ flex: 1 }}
      />
      <button onClick={handleSend} disabled={!text.trim()}>Send</button>
    </div>
  );
}

<ChatDrawer
  assistantId="my-assistant"
  options={{ customPromptBox: (props) => <VoicePromptBox {...props} /> }}
/>
```

`transcribeAudio` accepts **either a binary or a download URL**:

```tsx
// From a recorded binary (Blob/File)
await transcribeAudio(recordedBlob, { language: 'es' });

// From an already hosted audio file
await transcribeAudio('https://example.com/recording.mp3');
```

### `CustomPromptBoxProps` (speech-related)

| Prop | Type | Description |
|------|------|-------------|
| `sendMessage` | `(message: string, files?: File[], meta?: { transcriptId?: string }) => void` | Send a message. Pass `meta.transcriptId` to link it to a transcript |
| `transcribeAudio` | `(audio: Blob \| string, options?: { language?: string; messageUid?: string; chatUid?: string }) => Promise<WhisperTranscriptionResponse>` | Transcribe a binary (Blob/File) or a download URL via `/whisper`. Returns `{ transcriptId, text, language, audioUrl, model }` |

## Recording with `useSpeechRecording`

For full control over the recording UI (custom equalizer, buttons, etc.) use the
`useSpeechRecording` hook. It wraps `MediaRecorder` and exposes live amplitude
levels for an equalizer, plus pause/resume/stop/cancel.

```tsx
import { useSpeechRecording, DevicApiClient } from '@devicai/ui';

function MicButton({ apiKey }: { apiKey: string }) {
  const rec = useSpeechRecording({ bars: 5 });
  const client = new DevicApiClient({ apiKey, baseUrl: 'https://api.devic.ai' });

  const confirm = async () => {
    const blob = await rec.stop();
    if (!blob) return;
    const { text, transcriptId } = await client.transcribeAudio(blob, { language: 'es' });
    // use text / transcriptId...
  };

  if (!rec.isSupported) return null;

  return rec.isRecording || rec.isPaused ? (
    <div>
      {/* simple equalizer */}
      <div style={{ display: 'flex', gap: 3, alignItems: 'center', height: 24 }}>
        {rec.levels.map((l, i) => (
          <span key={i} style={{ width: 3, height: `${Math.max(10, l * 100)}%`, background: '#1890ff' }} />
        ))}
      </div>
      <span>{Math.floor(rec.durationMs / 1000)}s</span>
      <button onClick={rec.isPaused ? rec.resume : rec.pause}>{rec.isPaused ? 'Resume' : 'Pause'}</button>
      <button onClick={confirm}>Confirm</button>
      <button onClick={rec.cancel}>Cancel</button>
    </div>
  ) : (
    <button onClick={rec.start}>🎤</button>
  );
}
```

### `UseSpeechRecordingResult`

| Field | Type | Description |
|-------|------|-------------|
| `status` | `'idle' \| 'recording' \| 'paused'` | Current recording state |
| `isRecording` | `boolean` | `status === 'recording'` |
| `isPaused` | `boolean` | `status === 'paused'` |
| `isSupported` | `boolean` | Whether the browser supports audio recording |
| `levels` | `number[]` | Normalized amplitude per bar (0..1) for an equalizer, updated live |
| `durationMs` | `number` | Elapsed recording time in ms (excludes paused time) |
| `error` | `string \| null` | Last error (e.g. permission denied) |
| `isAutoStopping` | `boolean` | `true` while the silence auto-stop countdown is running |
| `autoStopProgress` | `number` | Auto-stop countdown progress, `1`→`0` (for an inverted ring) |
| `speechDetected` | `boolean` | `true` once a real voice peak was detected in the current recording |
| `start` | `() => Promise<void>` | Request mic access and start recording |
| `pause` | `() => void` | Pause an active recording |
| `resume` | `() => void` | Resume a paused recording |
| `stop` | `() => Promise<Blob \| null>` | Stop and return the recorded audio Blob |
| `cancel` | `() => void` | Stop and discard the recording |

`useSpeechRecording(options)` accepts `{ bars?: number; mimeType?: string }`
(`bars` defaults to 5, `mimeType` is auto-selected from supported types), plus
the silence **auto-stop** controls:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `autoStop` | `boolean` | `true` | Auto-stop after silence once speech is detected |
| `autoStopSilenceMs` | `number` | `1000` | Continuous silence before the countdown starts |
| `autoStopCountdownMs` | `number` | `1000` | Length of the countdown before it fires |
| `autoStopSilenceRatio` | `number` | `0.1` | Silence threshold as a fraction of the loudest voice (adaptive) |
| `autoStopSilenceLevel` | `number` | `0.02` | Absolute floor for the adaptive silence threshold |
| `autoStopSpeechLevel` | `number` | `0.12` | Absolute floor a peak must clear to count as speech |
| `onAutoStop` | `() => void` | — | Fired when the countdown completes (you decide what "stop" means) |

`onAutoStop` + `speechDetected` are the hooks you need to build a custom
hands-free loop on top of a `customPromptBox`.

## Direct client usage

The API client method works outside of components too:

```tsx
import { DevicApiClient } from '@devicai/ui';

const client = new DevicApiClient({ apiKey: 'your-api-key' });

const result = await client.transcribeAudio(audioBlob, {
  language: 'es',
  // messageUid, chatUid are optional links
});
// result: { transcriptId, text, language, audioUrl, model }
```

## Types

```tsx
import type {
  WhisperTranscriptionResponse,
  UseSpeechRecordingOptions,
  UseSpeechRecordingResult,
  SpeechRecordingStatus,
  CustomPromptBoxProps,
} from '@devicai/ui';

import { useSpeechRecording } from '@devicai/ui';
```

```tsx
interface WhisperTranscriptionResponse {
  transcriptId: string;   // attach to ProcessMessageDto.transcriptId
  text: string;
  language?: string;
  audioUrl?: string;
  model?: string;
}
```
