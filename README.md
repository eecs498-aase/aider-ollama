# aider-ollama

A working aider setup for local models, and the scripts for when the model has to
live on a different machine than you do.

## The shape of it

Three pieces, and it is worth knowing which machine each one is on.

```
Ollama    serves a model, over HTTP, on a port
.env      one line: which port that is
aider     reads .env, talks to whatever answers there
```

Ollama is the piece that wants a GPU. aider is a small program that sits with your
files. Where you put each one gives three setups.

**[Setup A](#setup-a--both-on-your-laptop)** puts both on your laptop. Start here.

**[Setup B](#setup-b--both-on-a-caen-machine)** puts both on a CAEN lab machine.
You sit at the machine and work there. It has an NVIDIA GPU, so the models your
laptop struggles with are comfortable. It is Setup A over again with one extra step
at the front.

**[Setup C](#setup-c--ollama-on-caen-aider-on-your-laptop)** leaves aider on your
laptop and puts only Ollama on the CAEN machine, reached over an SSH tunnel. Your
own files and editor against the lab GPU. It has the most moving parts, so take it
when you want it rather than to get started.

Do A and B. They are the same commands twice, and between them you have a model on
whatever machine you are sitting at. Switching is one line in `.env`, so you are
not choosing one forever.

---

## First, on whichever machine you are setting up

Once per machine.

**1. Clone this repo.**

```bash
git clone https://github.com/eecs498-aase/aider-ollama.git
cd aider-ollama
```

**2. Install aider.** On your own laptop, the normal installer:

```bash
python3 -m pip install aider-install && aider-install
aider --version
```

On a CAEN machine, **do not run that**: an aider installed that way is painfully
slow to start there. Use `bin/install-aider` instead, covered in
[Setup B](#setup-b--both-on-a-caen-machine).

**3. Stay in this directory for now.** Every command is `bin/something`, run from
here. Nothing installs onto your PATH, so there is only ever one way to spell a
command. Moving the config into your own project is
[the last section](#using-this-in-your-own-project); do that once the checks below
pass.

---

## Setup A — both on your laptop

Everything on one machine. Nothing here is CAEN-specific.

**1. Install Ollama.** Download it from
[ollama.com/download](https://ollama.com/download) and run it. On macOS, opening
the app is enough.

**2. Start the server** (skip if the macOS app is already running):

```bash
ollama serve &
```

**3. Pull the model** named in `.aider.conf.yml`:

```bash
ollama pull qwen3.5:4b
```

**4. Tell aider where it is:**

```bash
cp .env.local .env
```

That file is one line, `export OLLAMA_API_BASE=http://127.0.0.1:11434`, which is
where Ollama puts itself. aider reads `.env` out of whatever directory you run it
in, so this is all it needs. Writing the line by hand does the same thing.

**5. Prove it, then work:**

```bash
bin/check
aider
```

If any line is not `ok`, see [what `bin/check` checks](#what-bincheck-checks).

**How big a model your laptop can hold is the limit here.** A 9B wants about 16 GB
of RAM to be comfortable. Setup B is where you go for a bigger one, and it is the
same commands over again, so do it before you need it.

---

## Setup B — both on a CAEN machine

Sit at the machine and run both there. It has an NVIDIA GPU, so the models your
laptop struggles with are comfortable, and there is no tunnel to go wrong.

Your home directory is mounted on every CAEN machine, and two of the three pieces
live in it: Ollama goes in `~/.local`, and the models it pulls go in `~/.ollama`.
Pull a model at one machine and it is there at the next. aider is the exception. It
opens several hundred files to start, which from a network home takes tens of
seconds, so `bin/install-aider` builds it on the local disk of the machine you are
at and leaves a launcher in `~/.local/bin` that rebuilds it, about a minute, at any
machine that does not have it yet. That is the whole difference from Setup A, and
you do not manage any of it.

**1. Install aider:**

```bash
bin/install-aider
```

**2. Install Ollama:**

```bash
bin/install-ollama
```

You are not root on those machines, and `ollama.com/install.sh` writes to
`/usr/local` and calls `sudo`. `OLLAMA_INSTALL_DIR` does not stop it, because it
also wants to register a systemd service. This script unpacks the release tarball
into `~/.local` instead, and the tarball has exactly the layout the official
installer wants (`bin/ollama`, `lib/ollama/*`), so that is the same install without
the root.

Both scripts leave what they installed on your PATH. If either says to open a new
terminal, do that before going on.

**3. Survive your own logout.** Once, ever:

```bash
loginctl enable-linger $USER
```

Without it, systemd kills `ollama serve` the moment you log out, which looks like a
crash rather than a logout.

**4. Start the server and pull a model.** The GPU can take a bigger one than your
laptop:

```bash
ollama serve &
ollama pull qwen3.5:9b
```

If you pull something other than what `.aider.conf.yml` names, change its `model:`
line to match, and check that model has a block in `.aider.model.settings.yml`.
Every model shipped there already does.

**5. Point aider at it.** The model is on the machine you are sitting at, so this is
the same file as Setup A:

```bash
cp .env.local .env
```

**6. Prove it, then work:**

```bash
bin/check
aider
```

---

## Setup C — Ollama on CAEN, aider on your laptop

Your files, your editor and aider stay on your laptop; only the model is elsewhere.
Setup B already gets you the GPU, so take this one when you want your own machine's
files and tools against it.

The catch is that a CAEN machine can dial out and nothing can dial in. So the lab
machine pushes its Ollama port onto a third machine both ends can reach, and your
laptop pulls it back down:

```
CAEN lab machine  --ssh -R-->   CAEN node   <--ssh -L--   your laptop
bin/ollama-relay                (a meeting               bin/ollama-tunnel
ollama, on the GPU               point; runs                    aider
                                  nothing)              127.0.0.1:11435
```

**You need this repo on both machines.** If you have done Setup B, the lab machine
already has everything it needs.

### Once, to set it up

**1. On the CAEN machine**, get Ollama installed and a model pulled, exactly as in
[Setup B](#setup-b--both-on-a-caen-machine) steps 2 to 4. Skip this if you have
already done it. You do not need aider there; this setup runs it on your laptop.

**2. On your laptop**, pick the node and port the two halves will meet on:

```bash
bin/ollama-tunnel setup <your-uniqname>
```

It asks for your password and a Duo push, then writes
`~/.config/aider-ollama/tunnel.conf` in your CAEN home. That home is mounted on
every CAEN machine, so the lab machine reads it as a local file and needs no
configuring of its own.

### Each session, in this order

**3. On the CAEN machine**, publish the model:

```bash
bin/ollama-relay start
```

That starts `ollama serve` if it is not up, enables lingering so it survives your
logout, and opens the relay. `bin/ollama-relay status` reports every hop
separately.

**4. On your laptop**, pull it down:

```bash
bin/ollama-tunnel start
```

The model lands on **`127.0.0.1:11435`**, not 11434, so it cannot collide with an
Ollama running on your own machine.

**5. On your laptop**, point aider at that port:

```bash
cp .env.caen .env
```

One line again, `export OLLAMA_API_BASE=http://127.0.0.1:11435`. Copying
`.env.local` over it switches back to a model on your laptop.

**6. On your laptop**, prove it, then work:

```bash
bin/check
aider
```

`bin/check` says whether the answer came over the tunnel or from something running
locally, which is the failure this port split exists to prevent.

**7. When you are done**, on the CAEN machine:

```bash
bin/ollama-relay stop
```

Ollama has no authentication. While the relay is up, anyone logged into that shared
CAEN node can use your GPU and read what you send it.

### If you also run Ollama on your laptop

Nothing to do. That is why the tunnel is on 11435. Your laptop's Ollama keeps
11434, the lab machine's arrives on 11435, and which one aider talks to is the
`OLLAMA_API_BASE` line in `.env`. Both can run at once.

They used to share 11434, and that clash was the bad kind: a local Ollama answers a
probe exactly as the tunnel would, so nothing failed. aider just quietly used the
laptop's model while you wondered why the lab GPU felt slow. `bin/ollama-tunnel`
still refuses, and `status` still says which of the two is answering, in case
something else takes 11435:

```
ollama-tunnel: ollama is already serving 127.0.0.1:11435
  That is not the tunnel. Stop it first, or it will shadow the model
  on your lab computer and aider will quietly use the wrong one.
```

`OLLAMA_TUNNEL_PORT=11500 bin/ollama-tunnel start` moves it again if you need.

### Three things that make this harder than it looks

**`login.engin.umich.edu` is not a machine.** It is a round-robin over eighteen of
them, `caen-vnc-mi01` through `mi18`, handing out a different one per connection.
Both halves of a bounce have to meet on the *same* host, so two halves that each
"connect to CAEN" land on different computers and the tunnel is dead on arrival,
with both ends reporting a healthy connection. Neither script ever dials that name
for the tunnel; `setup` pins one node by its real name.

**A port on a shared node belongs to whoever got there first.** So your relay port
on the node is derived from your uniqname, and your two machines compute the same
one without coordinating. If it is taken anyway, `bin/ollama-relay` walks up until
one is accepted and records where it landed; your laptop follows on its next
`start`. If the whole node is down, `bin/ollama-tunnel node --next` moves both
halves to the next one in the ring.

**`ssh -L 11435:localhost:11435` can bind only `[::1]`.** Then `curl
localhost:11435` works, because curl tries every address a name resolves to, and
aider does not, because litellm resolves `127.0.0.1` and never falls back. You get
`APIConnectionError ... [Errno 61] Connection refused` from aider alone and go
looking for the fault in the wrong place. Both address families are bound
explicitly.

### Somewhere that is not CAEN

The relay is any host both machines can SSH to. Point the config at it by hand:

```bash
CAEN_NODE=relay.example.com RELAY_PORT=11787 bin/ollama-tunnel setup <your-user>
```

`node --next` assumes the CAEN naming and will not help you elsewhere; the rest
does not care.

---

## What `bin/check` checks

Run it on whichever machine aider is on, from the directory you run aider in. Every line is something
that ruins a session quietly if it is wrong.

```
  ok    .aider.conf.yml is here
  ok    model: ollama_chat/qwen3.5:4b
  ok    endpoint from .env: http://127.0.0.1:11435
  ok    Ollama answers at http://127.0.0.1:11435
  ok    served over the tunnel, from another machine
  ok    qwen3.5:4b is pulled
  ok    context is 32768 tokens, not 2048
  ok    aider is installed
  ok    the model replied: ready

All good. Run aider in this directory.
```

Two are worth knowing about before they happen.

**The context window.** A model with no entry in `.aider.model.settings.yml` gets
2048 tokens, because that is what litellm assumes for a model it has never heard
of. aider truncates your files to fit and the model starts inventing functions that
exist three lines further down. It reads as a stupid model rather than a
misconfiguration, and it is the reason this repo exists. Every model named in that
file already has a block; if you add one, copy a block and change the name, keeping
`num_ctx` in step with `max_input_tokens` in `.aider.model.metadata.json`.

**Which machine actually answered.** `bin/check` says whether the model came over
the tunnel or from something running here, and complains if that does not match
what `.env` asked for. A local Ollama sitting on the tunnel's port gets caught
rather than quietly serving you the wrong model.

To change models, edit the `model:` line in `.aider.conf.yml`, then run `bin/check`
again.

---

## The two variables, and why `source .env` is not enough

They are read by different programs, and mixing them up costs an afternoon.

| | read by | says |
|---|---|---|
| `OLLAMA_API_BASE` | aider | where to find the model |
| `OLLAMA_HOST` | the `ollama` command | where its server is, and what `ollama serve` binds to |

aider reads `.env` itself, so `OLLAMA_API_BASE` there is all it needs. The `ollama`
command does not read `.env` at all. It reads the environment. So to point `ollama
list` or `ollama pull` at a model arriving over the tunnel:

```bash
export OLLAMA_HOST=127.0.0.1:11435
ollama list
```

**`source .env` on its own will not do it.** A plain `KEY=value` that you source
becomes a shell variable, not an environment variable, and no program you launch
can see it. `ollama` would silently keep talking to 11434. Which is why every line
in `.env.local` and `.env.caen` says `export`: aider's parser strips the word, and
the files then work both ways.

```bash
source .env            # works, because the lines say export
set -a; . .env; set +a # the equivalent if yours do not
```

One trap. `OLLAMA_HOST` also decides what `ollama serve` binds to. Exporting
`127.0.0.1:11435` and then starting a server on your laptop puts it on top of the
tunnel. Set it to inspect a remote model, not before starting a local one.

---

## Using this in your own project

Once the checks pass here, move the config to whatever you are actually building.
This runs on whichever machine aider is on.

**1. Copy the config and the check into your project:**

```bash
cd /path/to/your/project
mkdir -p bin
cp ~/aider-ollama/.aider.conf.yml ~/aider-ollama/.aider.model.settings.yml \
   ~/aider-ollama/.aider.model.metadata.json ~/aider-ollama/AGENTS.md .
cp ~/aider-ollama/.env.local ~/aider-ollama/.env.caen .
cp ~/aider-ollama/bin/check bin/
```

**2. Pick where the model is, the same way as before:**

```bash
cp .env.local .env     # Ollama on this laptop            127.0.0.1:11434
cp .env.caen .env      # Ollama on a CAEN machine         127.0.0.1:11435
```

**3. Check it, then work:**

```bash
bin/check
aider
```

**4. Keep `.env` out of git.** The `.gitignore` here already does; copy that line
into yours.

`AGENTS.md` is loaded read-only into every session by the `read:` line in
`.aider.conf.yml`. Replace it with your project's own conventions.

---

## Commands

Which machine each one belongs on.

| | runs on | |
|---|---|---|
| `bin/check` | wherever aider is | prove the setup works, before you waste a session on it |
| `bin/install-aider` | a CAEN machine | install aider where it starts fast, with a launcher that keeps it working at the next machine |
| `bin/install-ollama [--prefix DIR]` | a CAEN machine | install Ollama without root |
| `bin/ollama-tunnel setup <user>` | your laptop | pick the node and port, once |
| `bin/ollama-tunnel start\|stop\|restart\|status` | your laptop | the laptop half of the tunnel |
| `bin/ollama-tunnel node [--next\|<name>]` | your laptop | show or move the node both halves meet on |
| `bin/ollama-relay start\|stop\|restart\|status` | a CAEN machine | the lab-machine half |

## Licence

MIT.
