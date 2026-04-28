# IXO fork notes

This is IXO's fork of [element-hq/element-call](https://github.com/element-hq/element-call).

## Branch layout

- `livekit/main` — tracks upstream `element-hq/element-call`. Pull from upstream here, do not commit IXO changes to this branch.
- `ixo` — IXO's working branch. Contains the patches below, on top of `livekit/main`.

To pick up upstream changes: pull `livekit/main`, then merge `livekit/main` into `ixo`.

## What we use this for

The IXO portal (`impacts-x-web`) embeds Element Call as a widget for in-room voice/video. Separately, the IXO recording bot (`ixo-matrix-recording-bot`) runs Element Call in a headless Puppeteer browser to record calls — joining as a regular Matrix user, capturing audio/video, and producing transcripts.

The recording bot needs to know **who is speaking when** so transcripts can attribute lines to real participants instead of LiveKit-internal track IDs. Mapping `track → matrix_user_id:device_id` requires access to the LiveKit `Room` object, which upstream keeps inside React/RxJS state and never exposes globally.

## Patches

### 1. Expose live LiveKit Rooms on `window.livekitRooms`

**File**: `src/room/InCallView.tsx`

`CallViewModel.allConnections$` already holds every active LiveKit `Room` (one per SFU connection), and React subscribes to it via `useBehavior(vm.allConnections$)`. Upstream uses this to populate the developer-settings tab. We add a `useEffect` that mirrors the same data onto `window.livekitRooms` and clears it on unmount.

```ts
useEffect(() => {
  window.livekitRooms = allConnections.getConnections().map((c) => ({
    room: c.livekitRoom,
    livekitAlias: c.livekitAlias,
    url: c.transport.livekit_service_url,
  }));
  return (): void => { delete window.livekitRooms; };
}, [allConnections]);
```

This gives any in-page consumer the full `livekit-client` Room API — `RoomEvent.ActiveSpeakersChanged`, `TrackSubscribed`, `room.remoteParticipants` — and crucially each `RemoteParticipant.identity` is `${matrix_user_id}:${device_id}`, which is what the recording bot needs.

A matching `declare global { interface Window { livekitRooms?: ... } }` block sits next to the existing `window.rtcSession` and `window.matrixclient` declarations.

**Why this approach**: it's the smallest possible patch (one `useEffect`), it reuses upstream's already-public `allConnections$` Behavior, and it doesn't change any existing behaviour. Alternatives we considered:

- **React fiber walk from outside** — works without any source change, but is fragile to React minification across upstream releases. Maintaining the fork is cheaper.
- **Monkey-patching the `livekit-client` `Room` prototype from `evaluateOnNewDocument`** — works but depends on the bundler not renaming things; brittle.
- **WebSocket / SDP interception** — exposes `participantSid` only, not the matrix identity, so it doesn't actually solve the mapping.

## Building & deploying

The fork is a drop-in replacement for upstream — same `pnpm build:embedded`, same `dist/` output. Two consumption paths:

1. **Vendored into the portal** — `impacts-x-web` copies `dist/` to `public/element-call/`. Today this is automated via a `copy-element-call.js` script that pulls from the npm package; for the fork, copy `dist/` directly (or update the script to point at this repo).
2. **Standalone Vercel deployment** — push the `ixo` branch to GitHub, point Vercel at it. Build command `pnpm build:embedded`, output `dist`, install `pnpm install --frozen-lockfile`. Note: a fresh deploy serves placeholder URLs in `config.json`; the recording bot intercepts and overrides `config.json` at runtime so this doesn't affect the bot, but human visitors to the standalone URL will see initialisation errors until `config.json` is fixed (edit `vite-embedded.config.ts` to bake in real homeserver + LiveKit URLs, or serve a custom `config.json` via a Vercel rewrite).

## Maintenance

When upstream releases a new version:

```bash
git checkout livekit/main
git pull upstream main
git checkout ixo
git merge livekit/main
pnpm install
pnpm lint:types
pnpm build:embedded
```

The patch only touches one file (`src/room/InCallView.tsx`) and references stable upstream APIs (`vm.allConnections$`, `getConnections()`, `connectionItem.livekitRoom`). Conflicts on rebase should be rare.

## Licence

Element Call is dual-licensed AGPL-3.0 / Element Commercial. This fork is published under the same terms — see `LICENSE-AGPL-3.0` and `LICENSE-COMMERCIAL`.
