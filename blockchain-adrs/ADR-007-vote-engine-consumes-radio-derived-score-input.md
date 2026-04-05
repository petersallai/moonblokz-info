# ADR-007: Vote Engine Consumes Radio-Derived Score Input

### Status
Accepted

### Context
MoonBlokz creator selection involves two related but distinct layers.

The first layer is **score-based vote-target selection**: when a node accepts a new transaction, it must decide which other node to vote for inside that transaction. That decision depends on radio-observed behavior, specifically on which other node has sent the most messages to the current node and therefore counts most strongly in that node’s local network view.

The second layer is **blockchain-side vote accumulation and creator selection**: once transactions containing votes are part of blockchain state, the system must track vote totals, resets, and creator-selection consequences in order to determine who may create the next block.

The blockchain responsibility boundary excludes radio-side observation and score calculation, but it does include blockchain-side handling of votes that already exist in transactions and blocks.

### Decision
The blockchain module does **not** compute the radio-derived score that determines **who a node votes for when it accepts a new transaction**.

That score calculation belongs to a separate module which decides, from radio-observed message counts, which other node currently matters most in the local network view of the node creating or accepting the transaction.

The blockchain module **does** own the blockchain-side vote logic, including:
- vote accumulation from votes recorded in transactions,
- vote resets,
- creator-selection consequences,
- and reconciliation of vote state during active-chain changes.

### Consequences
#### Positive
- Preserves the blockchain/radio boundary.
- Makes the distinction explicit between choosing a vote target and accumulating blockchain vote state.
- Keeps radio observation complexity outside blockchain logic.
- Keeps next-block creator calculation inside the blockchain domain where it belongs.
- Makes both sides easier to test independently.

#### Trade-offs
- Requires an explicit contract for the external vote-target selection input.
- Requires confidence that the external scoring layer is deterministic enough for blockchain use.
- Adds one more integration boundary where errors are possible.
- Requires careful naming so score-based vote-target selection is not confused with blockchain-side vote accumulation.

### Follow-up implications
This decision implies that later design work must define:
- the exact semantic shape of the external vote-target selection input,
- the update cadence and timing expectations for that input,
- the blockchain-side representation of accumulated votes and creator eligibility,
- and the interaction between score-input updates and chain-switch-driven vote reconciliation.
