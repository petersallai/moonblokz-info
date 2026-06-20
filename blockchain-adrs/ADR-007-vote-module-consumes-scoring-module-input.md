# ADR-007: Vote Module Consumes Scoring Module Input

### Status
Accepted

### Context
MoonBlokz creator selection involves two related but distinct layers.

The first layer is **scoring-module-based vote-target selection**: when a node creates a new transaction, the system must decide which other node to vote for inside that transaction. That decision depends on radio-observed behavior — specifically, on which other node has sent the most messages to the current node and therefore counts most strongly in that node's local network view. This selection is produced by the **scoring module**, which computes per-node scores from the senders of observed radio messages and provides the selected vote target to the caller that assembles the signed transaction before that transaction reaches the blockchain module.

The second layer is **blockchain-side vote accumulation and creator selection**: once transactions containing votes are part of blockchain state, the system must track each node's accumulated vote — each accepted transaction contributes one configured `vote_scale` credit to its target node, and each accepted block applies the integer-only anti-capture rule `accumulated_vote += floor(accumulated_vote × vote_interest / vote_scale)` to non-creators — and determine, from that accumulated state, who is currently expected to create the next block. This layer is owned by the **vote module** inside the blockchain module.

The blockchain responsibility boundary excludes radio-side observation and the scoring module, but it does include blockchain-side handling of votes that already exist in transactions and blocks.

### Decision
The blockchain module does **not** compute the score that determines **who a node votes for when it creates a new transaction**.

That computation belongs to the **scoring module**, which decides, from radio-observed message counts, which other node currently matters most in the local network view of the node creating the transaction. The scoring module exposes the selected vote target to the caller or upstream transaction-building layer; the blockchain module itself does not call the scoring module synchronously and instead receives the resulting fully assembled signed transaction with the chosen `vote` field already populated.

The blockchain module **does** own the blockchain-side vote logic via the **vote module**, including:
- per-node accumulated vote tracking,
- vote credits accumulated from votes recorded in transactions on the active chain,
- anti-capture interest growth applied on each accepted block via the integer rule `accumulated_vote += floor(accumulated_vote × vote_interest / vote_scale)`,
- creator accumulated-vote reset on successful block creation,
- grace-period accumulated-vote reset for the originally-top node on fallback,
- determination of the next expected block creator from the accumulated state,
- and reconciliation of accumulated-vote state during active-chain changes.

### Consequences
#### Positive
- Preserves the blockchain/radio boundary.
- Makes the distinction explicit between the scoring module choosing a vote target and the vote module accumulating blockchain vote state and naming the next creator.
- Keeps radio observation complexity outside blockchain logic.
- Keeps next-block creator calculation inside the blockchain domain where it belongs.
- Makes both sides easier to test independently.

#### Trade-offs
- Requires an explicit contract between the scoring module and the upstream local transaction-building path that embeds the chosen vote target into the signed transaction.
- Requires confidence that the scoring module is deterministic enough for blockchain use.
- Adds one more integration boundary where errors are possible.
- Requires careful naming so scoring-module-based vote-target selection is not confused with the vote module's blockchain-side vote accumulation and creator determination.

### Follow-up implications
This decision implies that later design work must define:
- the exact semantic shape of the scoring module's vote-target selection input,
- the update cadence and timing expectations for that input,
- the contract by which an upstream local transaction-building path embeds the chosen vote target into the signed transaction submitted to the blockchain module,
- the vote module's representation of accumulated votes and creator eligibility,
- and the interaction between scoring-module updates and chain-switch-driven vote reconciliation.
