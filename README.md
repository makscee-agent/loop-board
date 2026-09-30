# loop-board

A GitHub board your coding agent works through on its own. You write tasks as issues; a **loop worker** in a terminal
tab takes the next Ready one, gives the agent one timeboxed attempt with fresh context, and the agent pushes a branch,
opens a PR and leaves a short note on the issue. You answer questions in a comment and accept work by merging.

It's three small bash scripts over `gh`. The default agent is **Pi** (`pi -p`, as you run it through vc); any CLI
agent that takes a prompt works (`AGENT="claude -p --dangerously-skip-permissions" bin/loop`). macOS or Linux.

## How it works

```
Planned → Ready → Working → Needs me → Done          Executor: Agent | Me
```

- **A task is an issue** in your tasks repo, and a card on a GitHub Project over it.
- **`bin/loop`** takes the top `Ready` card with Executor `Agent`, moves it to `Working` and runs one attempt in
  `work/<issue>/`. The prompt is your `AGENTS.md` (house rules), `prompts/attempt.md` (how to work and finish) and the
  whole issue thread. The attempt writes its outcome to a file, and the loop routes the card:
  - `done` (a PR to try) or `needs-me` (a question) → **Needs me**
  - `continue` (progress, not finished) → back to **Ready** for a fresh attempt (at most `MAX_TRIES`, then Needs me)
  - no outcome (timebox ran out, crash) → **Needs me** with a comment
- **You** comment on a Needs-me card to answer; the loop sends it back to Ready on its next round. Merge the PR
  (`Closes <repo>#N`) and the issue closes; the board's built-in workflow moves it to Done.
- The thread is the memory: every attempt starts fresh and reads the earlier notes. Nothing lives only in a terminal.

## Setup (10 minutes)

You need `git`, `gh`, `jq`, `perl` and your agent (`pi` from vc) working in a terminal.

```sh
brew install git gh jq node                # or your package manager
gh auth login && gh auth refresh -s project   # boards need the project scope

# Pi on your PATH, the same version vc runs (skip if `pi --version` already matches)
v=$(jq -r .version ~/.void-code/runtime/pi/node_modules/@earendil-works/pi-coding-agent/package.json)
npm install -g "@earendil-works/pi-coding-agent@$v"
export VC_BOOTSTRAP_EXECUTABLE=~/.void-code/bin/vc   # how Pi finds your vc login (bin/loop sets it by itself)
pi -p "reply OK"                           # expect: OK, with no questions asked

gh repo clone makscee-agent/loop-board ~/loop-board && cd ~/loop-board
bin/init my-board                          # creates the private repo you/my-board, the board over it, board.env, AGENTS.md
```

Open the board link it prints and switch the view to **Board**, grouped by Status. A brand-new board can take a few
minutes before GitHub lists its cards to `bin/board list` and the loop. Then edit `AGENTS.md`: who you are,
your repos, how to test and deploy, what the agent may merge itself. It's the part that matters most; keep it short.

## Use it

```sh
cd ~/loop-board
bin/board add "Add hello.sh that prints hello" "In you/my-board. How to try: sh hello.sh" --ready
bin/loop                                   # a worker in this tab; Ctrl-C stops it
```

The worker prints what it takes, runs the agent (`pi -p` prints only its final reply, into the tab and
`work/<n>/log`; the real record is the note on the issue) and after the attempt moves the card. Then:

```sh
bin/board watch                            # what waits on you: Needs-me cards
bin/board list                             # every open card
bin/board move 3 Ready                     # move a card by hand
```

Read the note on the issue, try the PR, merge it. Or answer in a comment and let the loop pick it up again.

- `bin/board add "Title" "Body"` without `--ready` parks a task in Planned; `--me` makes it yours (the loop skips it).
- Tasks can point anywhere: "in you/some-app, fix …". The agent clones what it needs into its workspace.
- More workers: run `bin/loop` in more tabs. Workers on one machine never take the same card (a lock around pickup);
  across machines, give each its own board or accept a rare double pickup.
- `ATTEMPT_MIN=30` shortens the timebox (default 60), `ONCE=1` stops after one attempt, `IDLE_SEC` sets how often an
  idle worker looks at the board.

## Grow it

- **Better answers come from `AGENTS.md`.** When an attempt gets something wrong twice, add a line there.
- **Knowledge**: ask attempts for `Learned:` lines in the note (add it to `prompts/attempt.md`) and copy the good ones
  into `AGENTS.md` or a `wiki/` folder it points to.
- **Bigger work**: write a task "design X, don't build yet"; answer the design in a comment; then create the build
  tasks yourself.
- The loop never merges, deploys or changes your rules. Those are yours, unless you write otherwise in `AGENTS.md`.

## If something's off

- The worker stays idle with a Ready card: the card's Executor must be `Agent` and its Status exactly `Ready`
  (`bin/board list`).
- `no option 'Ready' in field Status`: the board was made by hand; run `bin/init` for a new one, or delete `.fields.json`
  after renaming the columns.
- The attempt ends at once: run your `AGENT` command by hand in a folder (`pi -p "reply OK"`): it must work with no
  terminal input.
- `void-code: managed Pi provider unavailable`: run `vc` once (it writes Pi's relay extension and your login), and
  check `~/.void-code/bin/vc` exists. If Pi picks some other provider, pin it:
  `AGENT="pi -p --provider void-codex" bin/loop`.
- `Error: … is not a function` from Pi: the `pi` on your PATH isn't the version vc runs; install it as in Setup.
