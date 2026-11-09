# Chop

Chop is a CLI for saving lists of terminal commands (*chops*) and running them
later. It's meant for those sequences you repeat often —deploying, spinning up
an environment, clearing caches— that don't deserve a script of their own.

Each chop has a name, a color and, optionally, numbering. Each item stores a
command, a description and the color each one is shown in. Everything is kept
in a local SQLite database.

## Requirements

- Go 1.27.2 or later
- `sh` in your `PATH` (commands run with `sh -c`)

No CGO needed: the SQLite driver ([modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite)) is pure Go.

## Installation

```sh
go install github.com/JacobsM14/chop@latest
```

Or clone the repository and install with `make`:

```sh
git clone https://github.com/JacobsM14/chop chop
cd chop
make install   # installs the binary into $GOBIN or $GOPATH/bin
```

If you'd rather not install it, `make build` puts the binary in `bin/chop`.

## Quick start

```sh
chop new deploy --color green --ordered   # create a numbered chop
chop add deploy                           # add an item (interactive form)
chop add deploy
chop get deploy                           # show its items
chop run deploy                           # ask for confirmation and run them in order
```

Output of `chop get deploy` with two items:

```
deploy
──────
1. go test ./...: run the tests
2. git push: push the changes
```

## Item order

How items are ordered depends on whether the chop is numbered:

- **Numbered chops** (`--ordered`): items keep the order in which you added
  them.
- **Unnumbered chops**: items are grouped by the color of their command, sorted
  alphabetically by color name (`blue`, `green`, `orange`, `purple`, `red`,
  `white`).

This order is the one used everywhere: `chop get` shows it, `chop run` runs the
commands in it, and the item numbers for `rm-item` and `run --item` follow it.

## Commands

| Command | Alias | Description |
|---|---|---|
| `chop new <name>` | | Create a chop |
| `chop add <name>` | | Add an item to a chop |
| `chop ls` | | List all chops |
| `chop get <name>` | `ver` | Show a chop and its items |
| `chop run <name>` | | Run the commands of a chop (see [Item order](#item-order)) |
| `chop rm-item <name> [n]` | | Remove an item from a chop |
| `chop delete <name>` | `rm` | Delete a chop and all its items |

### `new`

```sh
chop new <name> [--color <color>] [--ordered]
```

- Without flags, opens a form that asks for the color and whether the list is
  numbered.
- With `--ordered` but no `--color`, it only asks for the color.
- With `--color`, it asks nothing; the list is unnumbered unless you pass
  `--ordered`.

`--ordered` doesn't only number the list: it also keeps items in the order you
added them. Without it, items are grouped by color (see [Item order](#item-order)).

Valid colors: `red`, `green`, `blue`, `purple`, `orange`, `white`.

### `add`

```sh
chop add <name>
```

Always interactive. Asks for the command, the description (neither can be
empty) and the color of each one. By default the command is shown in `purple`
and the description in `white`. In numbered chops the item is appended to the
end of the list; in unnumbered ones it shows up in its color group.

### `ls`

```sh
chop ls [--sort name|color|ordered]
```

Shows each chop's name in its color. Sorted by name by default.

### `get`

```sh
chop get <name>
```

Shows the title and the items. Numbered chops use `1.`, `2.`…; the others use
`-`. Items appear in the order described in [Item order](#item-order).

### `run`

```sh
chop run <name> [flags]
```

Before running, it shows the list of commands and asks for confirmation. The
commands run in the same order `chop get` shows them. Each command runs with
`sh -c` attached to your terminal, so interactive commands work. By default it
stops at the first command that fails.

| Flag | Description |
|---|---|
| `-i, --item N` | Run only item N (using the numbering from `chop get`) |
| `--dry-run` | Print the commands without running them or asking for confirmation |
| `--continue` | Keep going even if a command fails; at the end it reports how many failed |
| `-y, --yes` | Don't ask for confirmation |

`Ctrl+C` interrupts the running command and stops the run.

### `rm-item`

```sh
chop rm-item <name> [n]
```

Removes item `n` (using the numbering from `chop get`). If you don't give `n`,
it opens a picker to choose it.

### `delete`

```sh
chop delete <name> [-y]
```

Deletes the chop and all its items. Asks for confirmation unless you pass
`-y`/`--yes`.

## Configuration

The database is stored at `~/.local/share/chop/chop.db`. The directory is
created automatically the first time.

To use a different path, set `CHOP_DB`:

```sh
CHOP_DB=/tmp/test.db chop ls
```

Handy for testing without touching your real chops.

## Shell completion

`chop completion <shell>` generates the completion script. To load it in every
session, add this to your shell config:

```sh
source <(chop completion zsh)    # ~/.zshrc
source <(chop completion bash)   # ~/.bashrc
chop completion fish | source    # ~/.config/fish/config.fish
```

Besides subcommands and flags, it completes your chop names
(`chop run <TAB>`), item numbers for `rm-item` and `run --item` (with the
command as description), colors for `new --color` and values for `ls --sort`.

## Development

```sh
make            # build into bin/chop (default target)
make run ARGS="get deploy"
make test       # tests with -race
make check      # fmt + vet + test
make help       # list all targets
```

Project layout:

```
.
├── main.go
├── cmd/                 # one file per subcommand (cobra)
└── internal/
    ├── store/           # Store interface and SQLite implementation
    ├── render/          # colors and output formatting (lipgloss)
    └── runner/          # command execution with sh -c
```

Interactive forms use [huh](https://github.com/charmbracelet/huh).
