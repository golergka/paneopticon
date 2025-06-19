goal
deliver a bun-powered monorepo that exposes a read-only, live-synced tmux session to any web client, preserving pane layout, colors, and scrollback, with absolutely zero input path from the browser. everything ships as two npm libraries ready to be consumed by a larger site or api stack.

⸻

0. repo shape

paneopticon/
  package.json        # bun workspace root
  bun.lockb
  packages/
    backend/          # npm: @paneopticon/backend
    frontend/         # npm: @paneopticon/frontend
  examples/
    node-demo/        # shows integration w/ express or h3
  .github/workflows/ci.yml

workspace config in root package.json:

{
  "name": "paneopticon",
  "private": true,
  "workspaces": ["packages/*"]
}


⸻

1. backend lib – @paneopticon/backend

item	detail
language	typescript (bun ts-x)
entry	src/index.ts exports createServer(opts)
runtime	bun v1+
deps	none (bun stdlib, zod)
ext deps	tmux >= 3.2 installed on host

1.1 api surface

import { createServer } from "@paneopticon/backend";

createServer({
  session: "experiment",       // tmux session name
  port: 5080,                  // ws listen port
  frameInterval: 100,          // ms between diffs
  scrollback: 5000,            // lines per pane
  authenticator: async (token)=>boolean, // optional
});

returns { shutdown(): Promise<void> }.

1.2 capture loop
	1.	poll tmux list-panes -t <session> -F ... to detect layout.
	2.	per pane call
tmux capture-pane -pJ -e -S -<scrollback> -t %<id>
where -e keeps ansi, -J strips line wraps.
	3.	compute unified diff vs last snapshot (simple slice/compare, keep last 10 k lines in memory).
	4.	emit json packet over websocket (see §3).

runs in single bun runtime using setInterval, no threads.

1.3 websocket server
	•	bare bun serve with upgrade to ws.
	•	packs messages as Uint8Array of utf-8 json lines (newline-delimited).
one packet per frame, multiplexing all pane diffs.
	•	closes connection on any received data (read-only guarantee).

1.4 auth & tls
	•	backend itself speaks plaintext on localhost by default.
	•	caller (larger site) expected to terminate tls and pass auth token in Sec-WebSocket-Protocol.
	•	optional pluggable authenticator(token) hook, default allow-all.

⸻

2. frontend lib – @paneopticon/frontend

item	detail
language	typescript, esm
bundler	vite or consumer’s choice
deps	xterm.js, zod
export	mount(element: HTMLElement, wsUrl: string, opts?: FrontendOpts)

2.1 internals
	•	maintains Map<paneId, XTerm>
	•	on layout message rebuilds css grid:

display: grid;
grid-template-columns: repeat(<cols>, 8px);
grid-template-rows: repeat(<rows>, 18px);


	•	each pane sits in its own <div class="pane" style="grid-area: ..."> with overflow: auto;
	•	writes incoming ansi text via term.write.
	•	scrollback handled by xterm (set to backend’s scrollback value).
	•	global keydown / keypress listeners call event.preventDefault() to enforce no input.

2.2 theme
	•	monospace font fallback stack: ui-monospace, "Cascadia Code", Menlo, monospace
	•	dark mode default, follow prefers-color-scheme.

⸻

3. wire format

newline-delimited json (one object per frame):

{
  "ts": 1687219123456,
  "layout": {                // omitted if unchanged
    "cols": 190,
    "rows": 48,
    "panes": [
      { "id": "%1", "x":0,"y":0,"w":95,"h":24 },
      { "id": "%2", "x":95,"y":0,"w":95,"h":24 }
    ]
  },
  "diffs": [
    { "id":"%1", "scroll":["+ ansi_line_here", "- prev_line_removed"] }
  ]
}

	•	first packet after connect always includes full layout plus full buffers (scroll array == complete pane snapshot).
	•	diffs use + / - prefixes purely for consumer-side patch algorithm; no actual git diff format.

⸻

4. dev / build scripts (root)

{
  "scripts": {
    "dev": "bun run packages/backend/src/cli.ts",
    "build": "bun run rimraf dist && bun run packages/*/build",
    "test": "bun wiptest",
    "lint": "biome check .",
    "format": "biome format ."
  }
}

each package ships its own build that emits dist/esm & dist/types.

⸻

5. ci

.github/workflows/ci.yml
	•	matrix: bun-latest on ubuntu-latest
	•	steps: bun install, bun run lint && bun run test && bun run build
	•	artifact upload of both tarballs

⸻

6. example integration

examples/node-demo/index.ts

import express from "express";
import * as http from "http";
import { createServer as createBackend } from "@paneopticon/backend";

const app = express();
const server = http.createServer(app);

createBackend({ session: "exp", port: 0 }); // unix socket auto

app.use(express.static("public")); // serves bundle using @paneopticon/frontend

server.listen(8080, () => console.log("open http://localhost:8080"));


⸻

7. security checklist
	•	ensure tmux binary path is whitelisted (no env injection).
	•	reject any websocket frame length >256 KiB (basic flood guard).
	•	strip \x1b]50; & other risky OSC sequences before forwarding (prevents font download exploits).
	•	run backend under dedicated unix user; process has no tty.

⸻

8. open items / nice-to-haves
	•	wasm diff algo for o(n) → o(Δ) memory.
	•	opt-in recording: write raw packets to .ndjson for replay.
	•	director focus-pane highlight (back-channel via filewatch or tmux hooks).

⸻

ship it.