# NoNameCoin — Proof of Stake Consensus Simulator

A distributed system that simulates the consensus validation process of a
fictional digital currency called NoNameCoin. Three independent Flask services
cooperate to register clients and validators, select validators weighted by their
stake, collect their votes on a transaction, and settle balances when the network
reaches consensus.

Built as a Distributed Systems course project.

## Authors

- Kauê Galon Silva
- Leonardo Kenji
- Gabriel Ziliotto

## Architecture

The system is split into three services, each with its own HTTP port and its own
SQLite database.

| Service | Port | Role |
| --- | --- | --- |
| Management | 5000 | Ledger of record. Stores clients, selectors and transactions. Entry point for new transactions. |
| Selector | 5001 | Registers validators, picks three of them weighted by stake, tallies their votes and settles the transfer. |
| Validator | 5002 | Applies the validation rules to a transaction and returns an approval or rejection. |

### Transaction flow

```text
client script ──POST /transacoes/{from}/{to}/{amount}──► Management (5000)
                                                              │
                                        persists transaction with status 0
                                                              │
                                        ──POST /seletor/select──► Selector (5001)
                                                                       │
                                                        selects 3 validators by stake
                                                                       │
                                                   ──POST /validador──► Validator (5002)
                                                                       │
                                                        returns status 1 or 2 per vote
                                                                       │
                                                        majority vote decides outcome
                                                                       │
                                        ◄──balance updates via /cliente/{id}──
```

Every endpoint answers with a `status` field: `1` means success, `2` means
failure.

## Validator selection

Only validators not currently on hold are eligible, and the network refuses to
proceed with fewer than three of them. Each eligible validator is weighted by its
stake, then penalized according to how many flags it has accumulated:

| Flags | Weight multiplier |
| --- | --- |
| 0 | 1.0 |
| 1 | 0.5 |
| 2 | 0.25 |
| 3 | 0.0, and the validator is expelled |

No single validator may hold more than 20% of the total stake weight, which caps
the influence of a wealthy participant. Once a validator passes 10,000 total
selections its counter resets and one flag is forgiven. Three distinct validators
are then drawn at random from the weighted pool.

## Validation rules

A validator rejects a transaction when any of the following holds:

- the unique key presented does not match the one it registered
- the sender's balance is smaller than the amount plus the fee
- the timestamp lies in the future, or is not after the sender's previous transaction
- the sender is currently blocked for excessive activity
- the sender exceeded 100 transactions in the last minute, which triggers a block
  whose duration doubles on each repeat offense

## Consensus and fees

A transaction is approved when more than half of the returned votes approve it,
rejected when more than half reject it, and left without consensus otherwise.

On approval, a fee of 1.5% of the amount is withheld. One third of that fee goes
to the selector that coordinated the round, and the remainder is split among the
validators, credited directly to their stake.

## Requirements

- Python 3.x
- Dependencies pinned in `requirements.txt`: Flask, Flask-SQLAlchemy,
  Flask-Migrate, requests, SQLAlchemy and their transitive dependencies

## Setup

```bash
git clone https://github.com/kauegalons/nonamecoin-consensus.git
cd nonamecoin-consensus

python -m venv venv
# Linux / macOS
source venv/bin/activate
# Windows
venv\Scripts\activate

pip install -r requirements.txt
```

## Running

The three services must run simultaneously, each in its own terminal. Start them
in this order, because validator registration pushes a key to the Validator
service and will fail if it is not listening yet.

```bash
# terminal 1 — Validator, port 5002
python Validator/validador.py

# terminal 2 — Selector, port 5001, must run from the repository root
python Selector/seletor.py

# terminal 3 — Management, port 5000, must run from inside Management/
cd Management
python main.py
```

The working directory matters. The Selector resolves its database path from the
current directory as `seletor/banco/seletor.db`, while Management resolves
`site.db` relative to its own folder as `Management/instance/site.db`.

## Seeding and exercising the system

Helper scripts populate the services and trigger a transaction. Run them from the
repository root with all three services up.

```bash
python criacao_cli_sel.py   # creates clients and selectors in Management
python cria_validadores.py  # registers validators in the Selector
python cria_transacao.py    # submits a transaction and prints the outcome
```

`cria_validadores.py` intentionally includes one validator with a stake of 40,
below the minimum of 50, to demonstrate the registration rejection.

## API reference

### Management, port 5000

| Method | Route | Description |
| --- | --- | --- |
| GET | `/cliente` | list all clients |
| POST | `/cliente/{name}/{password}/{balance}` | create a client |
| GET | `/cliente/{id}` | fetch one client |
| POST | `/cliente/{id}?amount={delta}` | add `delta` to the client's balance |
| DELETE | `/cliente/{id}` | remove a client |
| GET | `/seletor` | list all selectors |
| POST | `/seletor/{name}/{ip}` | register a selector with zero stake |
| POST | `/seletor/{id}/{name}/{ip}/{stake_delta}` | update a selector and add to its stake |
| DELETE | `/seletor/{id}` | remove a selector |
| GET | `/transacoes` | list all transactions |
| POST | `/transacoes/{sender}/{receiver}/{amount}` | create a transaction and start consensus |
| GET | `/transacoes/{id}` | fetch one transaction |
| POST | `/transacoes/{id}/{status}` | set a transaction's status |
| GET | `/hora` | current server time |

### Selector, port 5001

| Method | Route | Description |
| --- | --- | --- |
| POST | `/seletor/register/{name}/{stake}` | register a validator, minimum stake 50.0 |
| POST | `/seletor/select` | JSON body, runs a consensus round |
| DELETE | `/seletor/delete/{id}` | remove a validator |

### Validator, port 5002

| Method | Route | Description |
| --- | --- | --- |
| POST | `/validador` | JSON body, validates a transaction |
| POST | `/validador/register_key` | stores the unique key for a validator id |
| GET | `/validador/keys` | list registered keys |
| GET | `/validador/accounts` | list simulated accounts and balances |

## Security notice

This is an academic simulation and is not safe to expose beyond localhost:

- No endpoint requires authentication. Anyone who can reach a service may create
  clients, mint balances, delete records or force a transaction.
- Client passwords are stored in plain text and travel in the URL path, where they
  end up in server logs and browser history.
- All three services start with `debug=True`, which enables the Werkzeug
  interactive debugger and therefore remote code execution.

## License

Academic work, shared for reference.
