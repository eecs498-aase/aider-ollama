# aider-ollama

A working aider setup for local models, and the scripts to keep using it when
the model has to live on a different machine than you do.

Two things, and you may only want one of them:

- **A blank aider config for Ollama.** Model settings, context windows and
  pricing metadata that are already right, so aider does not silently truncate
  your files. Copy the dotfiles into any project and run `aider`. There is no
  wrapper command and nothing to learn.
- **A tunnel for a GPU you cannot SSH into.** Written for the University of
  Michigan CAEN lab machines, which can dial out but never in. Works for any
  pair of machines that can both reach some third host.

## Install

You need [Ollama](https://ollama.com/download) and aider
(`python3 -m pip install aider-install && aider-install`).

For the config alone, you only need the dotfiles - see below. `bin/install` is
for the tunnel, and puts its two commands on your PATH:

```bash
git clone git@github.com:eecs498-aase/aider-ollama.git
cd aider-ollama
bin/install          # symlinks ollama-tunnel and ollama-relay into ~/.local/bin
```

## The model is on this machine

Copy the dotfiles into your project, start Ollama, run aider:

```bash
cp .aider.conf.yml .aider.model.settings.yml .aider.model.metadata.json .env.example /path/to/project/
cd /path/to/project
cp .env.example .env

ollama serve &            # or just open the Ollama app on macOS
ollama pull qwen3.5:4b    # whatever .aider.conf.yml names
aider
```

That is the whole thing. Nothing wraps aider, and `.env` tells it where Ollama
is - the only line you change when the model moves to another machine.

To use a different model, edit the `model:` line in `.aider.conf.yml`.

**The one thing worth understanding.** A model with no entry in
`.aider.model.settings.yml` gets a 2048-token context, because that is what
litellm assumes for a model it has never heard of. aider then truncates your
files to fit and the model starts inventing functions that exist three lines
further down. It looks exactly like a stupid model, and it is the reason this
repo exists at all. Every model named in that file already has a block. If you
switch to one that does not, copy an existing block and change the name -
`num_ctx` there and `max_input_tokens` in `.aider.model.metadata.json` should
match.

## The model is on a machine you cannot SSH into

The CAEN lab machines have GPUs and can dial out, but nothing can dial in. So
the lab computer pushes its Ollama port onto a third machine both ends can
reach, and your laptop pulls it back down:

```
lab computer  --ssh -R-->   CAEN node   <--ssh -L--  your laptop
ollama-relay                (a meeting              ollama-tunnel
ollama on the GPU            point, runs                aider
                             nothing)            127.0.0.1:11434
```

Two commands, named for the machine you run them on. Neither will run on the
wrong one; they check and say so.

**Once**, on your laptop:

```bash
ollama-tunnel setup <your-uniqname>
```

That picks the node and the port the two halves will meet on and writes them to
`~/.config/aider-ollama/tunnel.conf` in your CAEN home. That home is mounted on
every CAEN machine, so the lab computer reads it as a local file and needs no
configuring of its own.

**Each session**, lab computer first, because it is the half that publishes:

```bash
ollama-relay start      # on the lab computer
ollama-tunnel start     # on your laptop
```

The tunnel lands on `127.0.0.1:11434`, which is what `.env.example` already
points aider at, so `aider` needs no telling that the model is elsewhere. `status` on either half reports every
hop separately, so you can see which one is down rather than guessing.

### Three things that make this harder than it looks

**`login.engin.umich.edu` is not a machine.** It is a round-robin over eighteen
of them - `caen-vnc-mi01` through `mi18` - handing out a different one per
connection. Both halves of a bounce have to meet on the *same* host, so two
halves that each "connect to CAEN" land on different computers and the tunnel is
dead on arrival, with both ends reporting a healthy connection. Neither script
ever dials that name for the tunnel; `setup` pins one node by its real name.

**A port on a shared node belongs to whoever got there first.** So your relay
port is not 11434 but a number derived from your uniqname - your two machines
compute the same one without coordinating. If it is taken anyway, `ollama-relay`
walks up until one is accepted and records where it landed; your laptop follows
on its next `start`. If the whole node is down, `ollama-tunnel node --next`
moves both halves to the next one in the ring.

**`ssh -L 11434:localhost:11434` can bind only `[::1]`.** Then `curl
localhost:11434` works, because curl tries every address a name resolves to, and
aider does not, because litellm resolves `127.0.0.1` and never falls back. You
get `APIConnectionError ... [Errno 61] Connection refused` from aider alone and
go looking for the fault in the wrong place. Both address families are bound
explicitly.

### Somewhere that is not CAEN

The relay is any host both machines can SSH to. Point the config at it by hand:

```bash
CAEN_NODE=relay.example.com RELAY_PORT=11787 ollama-tunnel setup <your-user>
```

`node --next` assumes the CAEN naming and will not help you elsewhere; the rest
does not care.

## Commands

| | |
|---|---|
| `ollama-tunnel setup <user>` | pick the node and port, once, on your laptop |
| `ollama-tunnel start\|stop\|restart\|status` | the laptop half |
| `ollama-tunnel node [--next\|<name>]` | show or move the node both halves meet on |
| `ollama-relay start\|stop\|restart\|status` | the lab-computer half |

## One warning

Ollama has no authentication. While the relay is up, anyone logged into that
shared node can use your GPU and read what you send it. `ollama-relay stop` when
you are done.

On CAEN specifically, `ollama-relay` also enables lingering for you
(`loginctl enable-linger`). Without it systemd kills everything you own the
moment you log out - the tunnel and `ollama serve` with it - which presents as
connections that keep dropping rather than as having logged out.

## Licence

MIT.
