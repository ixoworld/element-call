# IXO fork notes

This is IXO's fork of [element-hq/element-call](https://github.com/element-hq/element-call).

## Branch layout

- `ixo` — IXO's working branch and Vercel's production branch (every merge deploys to `element-call-silk.vercel.app`, which the recording bot uses). Contains the patches below on top of an upstream release tag.
- Upstream is `element-hq/element-call` (add it as the `upstream` remote). Track upstream **release tags** (`vX.Y.Z`), not upstream `main`.

Current base: upstream `v0.28.2` (Nov 5 2026), merged into `ixo` on top of the earlier base (upstream `main` at `d335a307`, Apr 27 2026).

## What we use this for

The IXO portal (`impacts-x-web`) embeds Element Call as a widget for in-room voice/video. Separately, the IXO recording bot (`ixo-matrix-recording-bot`) runs Element Call in a headless Puppeteer browser to record calls — joining as a regular Matrix user, capturing audio/video, and producing transcripts.

The recording bot needs to know **who is speaking when** so transcripts can attribute lines to real participants instead of LiveKit-internal track IDs. Mapping `track → matrix_user_id:device_id` requires access to the LiveKit `Room` object, which upstream keeps inside React/RxJS state and never exposes globally.

## Patches

### 1. Expose live LiveKit Rooms on `window.livekitRooms`

**File**: `sdk/src/media-backend/livekit/LivekitBackend.ts`

Since v0.28.1 the MatrixRTC and LiveKit logic lives in the in-repo SDK (`sdk/`), and the app's `CallViewModel` no longer sees LiveKit `Room` objects (the old `vm.allConnections$` is gone). The LiveKit backend's `connectionMap$` holds every active connection (one per SFU transport), each with its `livekitRoom`. We subscribe to it in the backend's scope, mirror it onto `window.livekitRooms`, and delete the global when the scope ends.

```ts
connectionMap$.pipe(scope.bind()).subscribe(({ value: connectionMap }) => {
  window.livekitRooms = connectionMap.getConnections().map((c) => ({
    room: c.livekitRoom,
    livekitAlias: c.livekitAlias,
    url:
      "livekit_service_url" in c.transport
        ? c.transport.livekit_service_url
        : undefined,
  }));
});
scope.onEnd(() => {
  delete window.livekitRooms;
});
```

The shape of `window.livekitRooms` (`{ room, livekitAlias, url }[]`) is unchanged, so the bot needs no change. This gives any in-page consumer the full `livekit-client` Room API — `RoomEvent.ActiveSpeakersChanged`, `TrackSubscribed`, `room.remoteParticipants` — and each `RemoteParticipant.identity` is `${matrix_user_id}:${device_id}` for participants authenticated through the JWT service's `/sfu/get` endpoint (every publisher on our servers).

The `declare global { interface Window { livekitRooms?: ... } }` block sits at the top of the same file. Upstream's `AGENTS.md` forbids page globals in upstream code; this is a deliberate fork-only divergence.

**Why this approach**: it's the smallest patch (one subscription in one file), it reads the backend's own connection list, and it doesn't change any existing behaviour. Alternatives we considered:

- **React fiber walk from outside** — works without any source change, but is fragile to React minification across upstream releases. Maintaining the fork is cheaper.
- **Monkey-patching the `livekit-client` `Room` prototype from `evaluateOnNewDocument`** — works but depends on the bundler not renaming things; brittle.
- **WebSocket / SDP interception** — exposes `participantSid` only, not the matrix identity, so it doesn't actually solve the mapping.

## Building & deploying

The fork is a drop-in replacement for upstream — same `pnpm build:embedded`, same `dist/` output. Two consumption paths:

1. **Vendored into the portal** — `impacts-x-web` copies `dist/` to `public/element-call/`. Today this is automated via a `copy-element-call.js` script that pulls from the npm package; for the fork, copy `dist/` directly (or update the script to point at this repo).
2. **Standalone Vercel deployment** (Vercel project `ixo-world/element-call`, production branch `ixo`). Build command `pnpm build:embedded`, output `dist`, install `pnpm install --frozen-lockfile`, Node 24.x. Vercel uses the pnpm version in `package.json#packageManager` (pnpm 11 since v0.28; pnpm 10 cannot read this lockfile's workspace settings). The embedded build ships no `config.json`; the recording bot intercepts the `config.json` request and answers with its own (homeserver, `livekit.livekit_service_url` fallback), so human visitors to the standalone URL get no homeserver.

The bot loads the page as a standalone app (no widget parameters), injects its session into the `matrix-auth-store` localStorage key, and clicks the `lobby_joinCall`, `incall_mute`, `incall_videomute` and `incall_leave` buttons by `data-testid`. Keep those working.

## Maintenance

When upstream releases a new version:

```bash
git fetch upstream --tags
git switch -c <you>/update-upstream-vX.Y.Z origin/ixo
git merge vX.Y.Z
pnpm install --frozen-lockfile
pnpm lint
pnpm test:unit --run
pnpm build:embedded
```

Then open a PR into `ixo`: the PR's Vercel preview build checks the build on Vercel before the merge deploys to production.

Always run `pnpm lint` (it includes `tsc`). The v0.28.2 merge was textually clean but left the old patch pointing at a variable upstream had removed; Vite builds without type-checking, so only `tsc` caught it before it reached production.

The patch only touches one file and reads backend internals (`connectionMap$`, `getConnections()`, `Connection.livekitRoom`, `Connection.livekitAlias`). Check them on every upstream merge.

Server requirements to keep in mind: on our homeservers the transport comes from the bot's `config.json` fallback (`org.matrix.msc4143` is not advertised), and our `lk-jwt-service` (0.3.0) only serves `/sfu/get`. Element Call ≥ v0.28.1 authenticates subscribe-only connections (transports other than our own) through `/get_token`, which needs lk-jwt-service ≥ 0.4.0.

## Licence

Element Call is dual-licensed AGPL-3.0 / Element Commercial. This fork is published under the same terms — see `LICENSE-AGPL-3.0` and `LICENSE-COMMERCIAL`.
