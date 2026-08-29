# aider-ollama

A working aider setup for local models, and the scripts for when the model has
to live on a different machine than you do.

## Pick your setup

Find your row. You only need that section.

| | Where you sit | Where the model runs | Go to |
|---|---|---|---|
| **A** | Your laptop | your laptop | [Setup A](#setup-a--everything-on-your-laptop) |
| **B** | A CAEN lab machine | that same machine | [Setup B](#setup-b--everything-on-a-caen-lab-machine) |
| **C** | Your laptop | a CAEN lab machine | [Setup C](#setup-c--aider-here-model-there) |

B is the simple way to use a lab GPU: sit at the machine and work there. C keeps
your own editor and files while the model runs on the lab machine's GPU. It has
the most moving parts, so take it only if you want it.

Nothing here is exclusive. Switching between them is one line in `.env`.

---

## First, get the files

Once, on each machine you will work on.

**1. Clone this repo.**

```bash
git clone https://github.com/eecs498-aase/aider-ollama.git
cd aider-ollama
```

**2. Install aider.**

```bash
python3 -m pip install aider-install && aider-install
aider --version
```

**3. Stay in this directory for now.** Everything is `bin/something`, run from
here. Nothing installs onto your PATH, so there is only ever one way to spell a
command. Moving the config into your own project is
[the last section](#using-this-in-your-own-project); do that once the checks
below pass.

Now go to your row's section.

---

## Setup A — everything on your laptop

**1. Install Ollama.** Download it from
[ollama.com/download](https://ollama.com/download) and run it. On macOS, opening
the app is enough.

**2. Start the server** (skip this if the macOS app is running):

```bash
ollama serve &
```

**3. Pull the model** named in `.aider.conf.yml`:

```bash
ollama pull qwen3.5:4b
```

**4. Tell aider where the model is.** Copy the template:

```bash
cp .env.local .env
```

That file is one line, `export OLLAMA_API_BASE=http://127.0.0.1:11434`, which is
where Ollama puts itself. aider reads `.env` from whatever directory you run it
in, so this is all it needs. Writing that line by hand does the same thing.

**5. Prove it works:**

```bash
bin/check
```

Read [what `bin/check` checks](#what-bincheck-checks) if any line is not `ok`.

**6. Run aider:**

```bash
aider
```

**Whether your laptop can hold a useful model is the catch.** A 9B wants about
16 GB of RAM to be comfortable. If yours cannot, use Setup B or C.

---

## Setup B — everything on a CAEN lab machine

Sit at the machine and run both halves there. It has an NVIDIA GPU, so the models
your laptop struggles with are comfortable, and there is no tunnel to go wrong.

**1. Install Ollama without root:**

```bash
bin/install-ollama
```

You are not root on those machines, and `ollama.com/install.sh` writes to
`/usr/local` and calls `sudo`. `OLLAMA_INSTALL_DIR` does not stop it, because it
also wants to register a systemd service. This script unpacks the release tarball
into `~/.local` instead. The tarball has exactly the layout of the prefix the
official installer wants (`bin/ollama`, `lib/ollama/*`), so that is the same
install without the root. It stages the download on local disk rather than in
your home, which matters when home is a network share and would otherwise carry
1.4 GB twice.

**2. Survive your own logout.** Once, ever:

```bash
loginctl enable-linger $USER
```

Without it, systemd kills `ollama serve` the moment you log out, which looks like
a crash rather than a logout.

**3. Start the server:**

```bash
ollama serve &
```

**4. Pull a model.** The GPU can take a bigger one than your laptop:

```bash
ollama pull qwen3.5:9b
```

If you pull something other than what `.aider.conf.yml` names, change its
`model:` line to match, and check that model has a block in
`.aider.model.settings.yml`. Every model shipped there already does.

**5. Point aider at it.** The model is on the machine you are sitting at, so this
is the same file as Setup A:

```bash
cp .env.local .env
```

**6. Prove it works, then run aider:**

```bash
bin/check
aider
```

One more CAEN fact: your home is a network share, so aider itself is slow to
start from it. Install it to local disk if that bothers you.

---

## Setup C — aider here, model there

Your files and your editor stay on your laptop; the GPU work happens on a lab
machine. The catch is that a CAEN machine can dial out and nothing can dial in,
so the lab computer pushes its Ollama port onto a third machine both ends can
reach, and your laptop pulls it back down.

```
lab computer  --ssh -R-->   CAEN node   <--ssh -L--  your laptop
bin/ollama-relay            (a meeting          bin/ollama-tunnel
ollama on the GPU            point, runs                aider
                             nothing)            127.0.0.1:11435
```

You need this repo on **both** machines, and Ollama installed on the lab one.

### Once, to set it up

**1. On the lab computer**, install Ollama exactly as in Setup B:

```bash
bin/install-ollama
```

**2. On your laptop**, pick the node and port the two halves will meet on:

```bash
bin/ollama-tunnel setup <your-uniqname>
```

It asks for your password and a Duo push, then writes
`~/.config/aider-ollama/tunnel.conf` in your CAEN home. That home is mounted on
every CAEN machine, so the lab computer reads it as a local file and needs no
configuring of its own.

### Each session, in this order

**3. On the lab computer**, publish the model:

```bash
bin/ollama-relay start
```

That starts `ollama serve` if it is not up, enables lingering for you, and opens
the relay. Run `bin/ollama-relay status` if you want to see each hop.

**4. On your laptop**, pull it down:

```bash
bin/ollama-tunnel start
```

The model lands on **`127.0.0.1:11435`**, not 11434, so it cannot collide with an
Ollama running on your laptop.

**5. On your laptop**, point aider at that port:

```bash
cp .env.caen .env
```

One line again, `export OLLAMA_API_BASE=http://127.0.0.1:11435`. Copying
`.env.local` over it switches back to a model on your own machine.

**6. Prove it, then run aider:**

```bash
bin/check
aider
```

`bin/check` says whether the answer came over the tunnel or from something
running locally, which is the failure this port split exists to prevent.

**7. When you are done**, on the lab computer:

```bash
bin/ollama-relay stop
```

Ollama has no authentication. While the relay is up, anyone logged into that
shared CAEN node can use your GPU and read what you send it.

### If you also run Ollama on your laptop

Nothing to do. That is why the tunnel is on 11435. Your laptop's Ollama keeps
11434, the lab computer's arrives on 11435, and which one aider talks to is the
`OLLAMA_API_BASE` line in `.env`. Both can run at once.

They used to share 11434, and that clash was the bad kind: a local Ollama answers
a probe exactly as the tunnel would, so nothing failed. aider just quietly used
the laptop's model while you wondered why the lab GPU felt slow.
`bin/ollama-tunnel` still refuses, and `status` still says which of the two is
answering, in case something else takes 11435:

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
with both ends reporting a healthy connection. Neither script ever dials that
name for the tunnel; `setup` pins one node by its real name.

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

Run it from the directory you run aider in. Every line is something that ruins a
session quietly if it is wrong.

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
of. aider truncates your files to fit and the model starts inventing functions
that exist three lines further down. It reads as a stupid model rather than a
misconfiguration, and it is the reason this repo exists. Every model named in that
file already has a block; if you add one, copy a block and change the name,
keeping `num_ctx` in step with `max_input_tokens` in
`.aider.model.metadata.json`.

**Which machine actually answered.** `bin/check` says whether the model came over
the tunnel or from something running here, and complains if that does not match
what `.env` asked for. A local Ollama sitting on the tunnel's port gets caught
rather than quietly serving you the wrong model.

To change models, edit the `model:` line in `.aider.conf.yml`, then run
`bin/check` again.

---

## The two variables, and why `source .env` is not enough

They are read by different programs, and mixing them up costs an afternoon.

| | read by | says |
|---|---|---|
| `OLLAMA_API_BASE` | aider | where to find the model |
| `OLLAMA_HOST` | the `ollama` command | where its server is, and what `ollama serve` binds to |

aider reads `.env` itself, so `OLLAMA_API_BASE` there is all it needs. The
`ollama` command does not read `.env` at all. It reads the environment. So to
point `ollama list` or `ollama pull` at a model arriving over the tunnel:

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

**1. Copy the config and the check into your project:**

```bash
cd /path/to/your/project
mkdir -p bin
cp ~/aider-ollama/.aider.conf.yml ~/aider-ollama/.aider.model.settings.yml \
   ~/aider-ollama/.aider.model.metadata.json ~/aider-ollama/AGENTS.md .
cp ~/aider-ollama/.env.local ~/aider-ollama/.env.caen .
cp ~/aider-ollama/bin/check bin/
```

**2. Pick an endpoint, the same way as before:**

```bash
cp .env.local .env     # the model on this machine       127.0.0.1:11434
cp .env.caen .env      # the model on a lab machine      127.0.0.1:11435
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

| | |
|---|---|
| `bin/check` | prove the setup works, before you waste a session on it |
| `bin/install-ollama [--prefix DIR]` | install Ollama into your home, no root |
| `bin/ollama-tunnel setup <user>` | pick the node and port, once, on your laptop |
| `bin/ollama-tunnel start\|stop\|restart\|status` | the laptop half |
| `bin/ollama-tunnel node [--next\|<name>]` | show or move the node both halves meet on |
| `bin/ollama-relay start\|stop\|restart\|status` | the lab-computer half |

## Licence

MIT.
