# aider-ollama

A working aider setup for local models, and the scripts for when the model has
to live on a different machine than you do.

## Pick your setup

Students work in one of three ways. Find your row; you only need that section.

| Where you sit | Where the model runs | What you need |
|---|---|---|
| Your laptop | your laptop | [the dotfiles](#a-everything-on-your-laptop) |
| A CAEN lab machine | that same machine | [the dotfiles, plus a no-root Ollama](#b-everything-on-a-caen-lab-machine) |
| Your laptop | a CAEN lab machine | [both, plus the tunnel](#c-aider-on-your-laptop-model-on-a-lab-machine) |

Row 2 is the simple way to use a lab GPU: sit at the machine and work there.
Row 3 is for keeping your own editor and files while the model runs on the lab
machine's GPU - more moving parts, so only take it if you want it.

You can be more than one of these on different days. Nothing here is exclusive,
and switching is one line in `.env`.

## Getting the files

```bash
git clone git@github.com:eecs498-aase/aider-ollama.git
cd aider-ollama
```

Everything is `bin/something`, run from that directory. Nothing installs onto
your PATH, so there is only ever one way to spell a command.

You also need aider (`python3 -m pip install aider-install && aider-install`).

## The dotfiles, which every row needs

```bash
cp .aider.conf.yml .aider.model.settings.yml .aider.model.metadata.json .env.example /path/to/project/
cd /path/to/project
cp .env.example .env
```

`.env` is the one file that differs between the three rows: it says where
Ollama is. Everything else is identical.

**The one thing worth understanding.** A model with no entry in
`.aider.model.settings.yml` gets a 2048-token context, because that is what
litellm assumes for a model it has never heard of. aider then truncates your
files to fit and the model starts inventing functions that exist three lines
further down. It looks exactly like a stupid model, and it is the reason this
repo exists at all. Every model named in that file already has a block. If you
switch to one that does not, copy an existing block and change the name -
`num_ctx` there and `max_input_tokens` in `.aider.model.metadata.json` should
match. To change models, edit the `model:` line in `.aider.conf.yml`.

## A: Everything on your laptop

Install [Ollama](https://ollama.com/download), then:

```bash
ollama serve &            # or just open the Ollama app on macOS
ollama pull qwen3.5:4b    # whatever .aider.conf.yml names
aider
```

`.env` as shipped already points at `127.0.0.1:11434`, which is where Ollama
puts itself. Nothing wraps aider. That is the whole thing.

Whether your laptop can hold a useful model is the catch - a 9B wants about
16GB of RAM to be comfortable. If it cannot, use row B or C.

## B: Everything on a CAEN lab machine

Sit at the machine, run both there. It has an NVIDIA GPU, so the models your
laptop struggles with are comfortable, and there is no tunnel to go wrong.

You are not root on those machines, and `ollama.com/install.sh` writes to
`/usr/local` and calls `sudo` - `OLLAMA_INSTALL_DIR` does not stop it, because
it also wants to register a systemd service. Use the installer here instead:

```bash
bin/install-ollama        # unpacks the release tarball into ~/.local
```

The tarball has exactly the layout of the prefix the official installer wants -
`bin/ollama`, `lib/ollama/*` - so putting it in `~/.local` is that same install
without the root. It stages the download on local disk rather than in your
home, which matters when home is a network share and would otherwise carry
1.4GB twice.

Then it is row A again, with `.env` unchanged:

```bash
ollama serve &
ollama pull qwen3.5:9b
aider
```

Two CAEN facts worth knowing. Your home is a network share, so aider itself is
slow to start from it - install it to local disk if that bothers you. And
`loginctl enable-linger $USER` once, or systemd kills `ollama serve` the moment
you log out, which looks like a crash rather than a logout.

**The one thing worth understanding.** A model with no entry in
`.aider.model.settings.yml` gets a 2048-token context, because that is what
litellm assumes for a model it has never heard of. aider then truncates your
files to fit and the model starts inventing functions that exist three lines
further down. It looks exactly like a stupid model, and it is the reason this
repo exists at all. Every model named in that file already has a block. If you
switch to one that does not, copy an existing block and change the name -
`num_ctx` there and `max_input_tokens` in `.aider.model.metadata.json` should
match.

## C: aider on your laptop, model on a lab machine

Your files and your editor stay on your laptop; the GPU work happens on a lab
machine. The catch is that a CAEN machine can dial out and nothing can dial in,
so the lab computer pushes its Ollama port onto a third machine both ends can
reach, and your laptop pulls it back down:

```
lab computer  --ssh -R-->   CAEN node   <--ssh -L--  your laptop
bin/ollama-relay            (a meeting          bin/ollama-tunnel
ollama on the GPU            point, runs                aider
                             nothing)            127.0.0.1:11435
```

Clone this repo on **both** machines, and install Ollama on the lab one with
`bin/install-ollama` as in row B. Then two commands, named for the machine you
run them on - neither will run on the wrong one; they check and say so.

**Once**, on your laptop:

```bash
bin/ollama-tunnel setup <your-uniqname>
```

That picks the node and the port the two halves will meet on and writes them to
`~/.config/aider-ollama/tunnel.conf` in your CAEN home. That home is mounted on
every CAEN machine, so the lab computer reads it as a local file and needs no
configuring of its own.

**Each session**, lab computer first, because it is the half that publishes:

```bash
bin/ollama-relay start      # on the lab computer
bin/ollama-tunnel start     # on your laptop
```

The tunnel lands on **`127.0.0.1:11435`**, not 11434. Point aider at it in
`.env` - the line is already there, commented:

```
OLLAMA_API_BASE=http://127.0.0.1:11435
```

That is the only change to a project, and switching back to a model on your own
machine is switching that line back. `status` on either half reports every
hop separately, so you can see which one is down rather than guessing.

### If you also run Ollama on your laptop

Nothing to do - that is why the tunnel is on 11435. Your laptop's Ollama keeps
11434, the lab computer's arrives on 11435, and which one aider talks to is the
`OLLAMA_API_BASE` line in `.env`. Both can run at once.

They used to share 11434, and that clash was the bad kind: a local Ollama
answers a probe exactly as the tunnel would, so nothing failed - aider just
quietly used the laptop's model while you wondered why the lab GPU felt slow.
`bin/ollama-tunnel` still refuses, and `status` still says which of the two is
answering, in case something else takes 11435:

```
ollama-tunnel: ollama is already serving 127.0.0.1:11435
  That is not the tunnel. Stop it first, or it will shadow the model
  on your lab computer and aider will quietly use the wrong one.
```

`OLLAMA_TUNNEL_PORT=11500 bin/ollama-tunnel start` moves it again if you need.

### Three things that make this harder than it looks

**`login.engin.umich.edu` is not a machine.** It is a round-robin over eighteen
of them - `caen-vnc-mi01` through `mi18` - handing out a different one per
connection. Both halves of a bounce have to meet on the *same* host, so two
halves that each "connect to CAEN" land on different computers and the tunnel is
dead on arrival, with both ends reporting a healthy connection. Neither script
ever dials that name for the tunnel; `setup` pins one node by its real name.

**A port on a shared node belongs to whoever got there first.** So your relay
port on the node is a number derived from your uniqname - your two machines
compute the same one without coordinating. If it is taken anyway, `bin/ollama-relay`
walks up until one is accepted and records where it landed; your laptop follows
on its next `start`. If the whole node is down, `bin/ollama-tunnel node --next`
moves both halves to the next one in the ring.

**`ssh -L 11435:localhost:11435` can bind only `[::1]`.** Then `curl
localhost:11435` works, because curl tries every address a name resolves to, and
aider does not, because litellm resolves `127.0.0.1` and never falls back. You
get `APIConnectionError ... [Errno 61] Connection refused` from aider alone and
go looking for the fault in the wrong place. Both address families are bound
explicitly.

### Somewhere that is not CAEN

The relay is any host both machines can SSH to. Point the config at it by hand:

```bash
CAEN_NODE=relay.example.com RELAY_PORT=11787 bin/ollama-tunnel setup <your-user>
```

`node --next` assumes the CAEN naming and will not help you elsewhere; the rest
does not care.

## Commands

| | |
|---|---|
| `bin/install-ollama [--prefix DIR]` | install Ollama into your home, no root |
| `bin/ollama-tunnel setup <user>` | pick the node and port, once, on your laptop |
| `bin/ollama-tunnel start\|stop\|restart\|status` | the laptop half |
| `bin/ollama-tunnel node [--next\|<name>]` | show or move the node both halves meet on |
| `bin/ollama-relay start\|stop\|restart\|status` | the lab-computer half |

## One warning, for row C

Ollama has no authentication. While the relay is up, anyone logged into that
shared CAEN node can use your GPU and read what you send it. Run
`bin/ollama-relay stop` when you are done.

In row C, `bin/ollama-relay` runs `loginctl enable-linger` for you, so the
tunnel and `ollama serve` survive you logging out of the lab machine. In row B
you are doing that yourself; see above.

## Licence

MIT.
