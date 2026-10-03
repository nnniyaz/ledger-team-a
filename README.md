# Ledger (Team A)

A crash-safe key-value storage engine in Ada and SPARK. Data the engine has acknowledged survives a crash, and the core of that claim is backed by proof.

## Status

Sprint 2: requirements and design are written; the Alire project builds. Engine code starts in Sprint 3.

## Documents

| Document | What it covers |
|---|---|
| [Requirements](docs/requirements.md) | What we build and how each requirement is verified |
| [Design](docs/design.md) | Architecture, file formats, write path, recovery, checkpoint |
| [Decision 0001: durability](docs/decisions/0001-durability.md) | Write-ahead log, and the alternatives rejected |
| [Decision 0002: index](docs/decisions/0002-index.md) | B+tree, and the alternatives rejected |
| [Decision 0003: proof scope](docs/decisions/0003-proof-scope.md) | What is proved in SPARK and what is tested |
| [Development process](docs/sdlc.md) | SDLC phases, sprint plan, Definition of Done, risks |

## Build

Requires [Alire](https://alire.ada.dev).

```
alr build
alr exec -- gnatprove -P ledger.gpr
```

## Team

Niyaz Nassyrov (Team Leader), Ameer Hamza Syed, Syed Hassan, Mohammed asad Khoja.
