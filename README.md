# Cross-chain Execution Confirmation Oracle

A standalone GenLayer Intelligent Contract primitive for the **Standalone Contracts** track.

## Purpose

A bridge or messaging protocol says a cross-chain transfer is "pending",
"relayed" or "done", and the only source for that is often the bridge's own
interface. This contract answers the question that matters to the receiving
side: did the transfer really execute on the destination chain, to the right
recipient, in the right token, for at least the right amount? A requester
states what they expect and points at a public explorer or node API response
for the destination transaction. Independent AI validators read that evidence
and the contract decides. The result is an on-chain record that insurance,
settlement or release logic can read without trusting the bridge.

Real-world use cases:

- **Bridge insurance and settlement**: release a payout only when the
  destination chain shows the funds arrived.
- **Cross-chain checkout and payroll**: confirm that a payment reached the
  merchant or employee in the agreed token.
- **Bridge monitoring**: a neutral second opinion on "stuck" transfers.

## Design properties

- **The model extracts, the code decides.** Validators are asked only to read
  the evidence and list facts: the transaction hash the response describes,
  whether it succeeded, and every transfer it credited (recipient, token, raw
  amount). They never see what the requester expects, so they cannot be nudged
  toward a match. The contract then decides in a fixed order and returns a
  reason code: `evidence_mismatch`, `tx_failed`, `wrong_recipient`,
  `wrong_token`, `amount_too_low` or `ok`.
- **Tokens are identified by contract address, never by symbol.** Lookalike
  tokens with a familiar symbol are the classic cross-chain trap, and one live
  run below is exactly that. EVM hex identifiers are compared
  case-insensitively, while Solana base58 and Cosmos denoms are compared as
  written, because those are case-sensitive.
- **Equivalence anchored to the decision.** Two extractions are equivalent only
  if the contract's decision for both is the same reason code, which also
  means the same confirmed or unconfirmed outcome. So the stored result never
  depends on who led, and an irrelevant extra fee transfer does not break
  consensus. Exhaustive offline tests cover every pair in a 628-extraction
  domain, and a soundness test shows that `ok` always rests on a real,
  matching transfer in the extraction.
- **Evidence is bound to the claim.** The evidence URL must contain the
  transaction hash, and the hash inside the response must match it, otherwise
  the decision is `evidence_mismatch`.
- **Not found is not a verdict.** A 404 (the transaction is not indexed yet)
  fails the check and the request stays `pending`, so it can be checked again
  later. Only evidence that is found and does not match produces `unconfirmed`.
- **Adversarial by design.** Explorer responses carry attacker-controlled text
  (token names, symbols, memos). The prompt fences the evidence as data, and
  since the model never produces the verdict, a fooled extraction can only
  yield facts that are then compared with the requester's own expectations.
- **No funds move.** The contract holds and transfers nothing.

## Design

### Lifecycle

```
create_request    (requester)  destination tx + evidence URL + expected recipient, token, minimum amount   -> pending
check             (anyone)     AI consensus                                                              -> confirmed | unconfirmed
update_evidence   (requester)  once, only after unconfirmed: another destination tx or explorer          -> pending
check             (anyone)     final answer
```

### Decision (computed by the contract, in this order)

| Reason | Meaning |
|---|---|
| `evidence_mismatch` | the response describes a different transaction |
| `tx_failed` | the transaction did not succeed |
| `wrong_recipient` | no transfer credits the expected recipient |
| `wrong_token` | the recipient was paid, but not in the expected token |
| `amount_too_low` | right token, but the total is below the minimum |
| `ok` | confirmed |

Several transfers of the right token to the right recipient are added up.

### Consensus

```
leader_fn():
    fetch the evidence URL (404 -> [EXTERNAL] error, request stays pending)
    ask LLM for {tx_hash, succeeded, transfers[], reasoning}, blind to the expectations
    return _validate_extraction(parsed JSON)

validator_fn(leader_result):
    leader_ex    = _validate_extraction(leader_result)
    validator_ex = leader_fn()                       # independently re-fetch + re-extract
    accept only if _decide(leader_ex) == _decide(validator_ex)
```

### Strict validation before any state write

| Field | Requirement |
|---|---|
| `tx_hash` | a non-empty string |
| `succeeded` | a real JSON boolean (the string `"false"` is rejected) |
| `transfers` | a list of at most 20 objects |
| `recipient`, `token` | non-empty strings |
| `amount_raw` | a non-negative integer or a digit string (no decimals, signs, exponents or booleans) |
| `reasoning` | a string, stored truncated to 500 characters |

These rules, the decision order and the equivalence check are covered by
offline tests that load the helper functions straight from the contract
source, so they test exactly the deployed code:

```
python3 tests/test_extraction_logic
```

The offline suite passed **17 tests** on commit `538a083a5fe23843b04f2d3c5f0e6828030e5d57`.

### Scope and limits

The oracle trusts what the explorer reports. A lagging, wrong or manipulated
explorer is the main risk, which is why the requester gets one chance to point
at a different transaction or explorer. "Succeeded" is as of the moment of the
check, so integrators that need finality should wait for it before asking. The
source side of the transfer is not verified: the requester declares what they
expect, and the oracle confirms what the destination chain shows.

## Deployed contract

| Field | Value |
|---|---|
| Network | GenLayer Studio (Studionet) |
| Deployed address | `0xCDFD404e132F86Bde3bfd26F0a5d7b5e28771554` |
| Explorer | [Contract in Explorer](https://explorer-studio.genlayer.com/address/0xCDFD404e132F86Bde3bfd26F0a5d7b5e28771554) |
| Tx (deploy) | [Finalized deployment](https://explorer-studio.genlayer.com/tx/0x98e3aa11cbf3c3cf9d9a8bfc858964326e9bb7700f20d260a10d3ba0596527bb) |
| Source | [`crosschain_execution_oracle.py`](./crosschain_execution_oracle.py) |

## Test inputs

The evidence documents are synthetic test files modeled on an explorer API
response (a Blockscout-style transaction with `token_transfers`), named by their
transaction hash so the evidence URL contains it, exactly like a real explorer
URL. They describe a fictional bridge delivery of 250 USDC on Ethereum, and
every input of every run can be read and replayed by anyone:

| Fixture | Transaction hash | What it shows |
|---|---|---|
| Delivered correctly | `0x8b8a872815cd8df89806f45cbfd4bdc6c24f9777b7a247cb804f1a95a9ce5fe2` | genuine USDC, expected recipient, 250000000 |
| Lookalike token | `0x379f8944385ea918637996d1869c994874010e1fa896ca6a41eac94f1e6d5df7` | right recipient and amount, token address differs in its last character, symbol still "USDC" |
| Injection in a token name | `0x89c930cae53254a83667be0f7c094e3561ffbe6f73747eed8dc1250fe758dc92` | paid to a different address; the token name contains an instruction aimed at the checker |
| Reverted | `0x355f4ed2fc9d646595756b8cc0aa7715ce065d5d3b91fb402588509b2e995d96` | status error, nothing credited |
| Retry that succeeded | `0xd8de8bdfe2421ff20156a02da3f1b0b3859a0f34c46c7bd571aa9487808dc8bf` | a second delivery, correct |

Each is served at `https://raw.githubusercontent.com/Perf89/Cross-chain-Execution-Confirmation-Oracle/main/fixtures/<transaction hash>.json`.

Expectations used in all requests:

| Field | Value |
|---|---|
| `dest_chain` | `ethereum` |
| `expected_recipient` | `0x5B38Da6a701c568545dCfcB03FcB875f56beddC4` |
| `expected_token` | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| `min_amount_raw` | `250000000` |

## Studionet results

These calls used synthetic Ethereum explorer fixtures. The status and reason are contract fields returned by `get_request`.

| Request | Scenario | Observed result | Explorer transactions |
|---|---|---|---|
| 0 | Successful transfer | `confirmed / ok` | [Create](https://explorer-studio.genlayer.com/tx/0x643e33ebeeeaafe262e163882ece6178f29c2503c1e64c37b69682d52995e230), [check](https://explorer-studio.genlayer.com/tx/0x1b219d4718eaab7e3158b2e41a0a669f260683c345756165866a799b611ddd88) |
| 1 | Lookalike token | `unconfirmed / wrong_token` | [Create](https://explorer-studio.genlayer.com/tx/0xbfe9da92c869b9f1aba3da689eff3f2c3e05d2ab9931c1da174dbd1fdfcf258d), [check](https://explorer-studio.genlayer.com/tx/0x02c714b428eeb7a7f4816c6c3f5b5cc20c27aff36ab2b2cd32562add9fbef41a) |
| 2 | Different recipient, with an embedded instruction in the token name | `unconfirmed / wrong_recipient` | [Create](https://explorer-studio.genlayer.com/tx/0x98bd85349d24d1588d9b4cc9b58bac818a3b8ed0b102745ca130d1b80320aebf), [check](https://explorer-studio.genlayer.com/tx/0xab5473f95878ac8b939dd9bd7945d3b452732f09798ecfff34a17ad63b581d9f) |
| 3 | Reverted execution followed by an evidence update | `unconfirmed / tx_failed` → `pending` | [Create](https://explorer-studio.genlayer.com/tx/0x2d4f31c2e503f132100a47b156b447984f874f326f5acb8323286d2ca10782dd), [check](https://explorer-studio.genlayer.com/tx/0xf7becaf1b14ec80c85ab152759c7311a19c7fb4d132ce471b1d8683a42b06c8b), [update](https://explorer-studio.genlayer.com/tx/0x74010c631c2606f6a7a03aa2c1d675882bd4ae8ebdd42d4a74ce09fd9e10ff3d) |
| — | Lookalike transaction hash with the successful-transfer fixture URL | Rejected with `evidence_url must contain the transaction hash` | [Rejected create](https://explorer-studio.genlayer.com/tx/0x5ef764455b6b1ac4bfd0b6cf12efa6939f2e31430ec70c727d6e96c9b4ddea25) |

## Reproducing

1. Open [GenLayer Studio](https://studio.genlayer.com) and deploy
   `crosschain_execution_oracle.py` (constructor takes no arguments).
2. `create_request(dest_chain, dest_tx, evidence_url, expected_recipient, expected_token, min_amount_raw)`
   with a fixture's transaction hash and its URL.
3. `check(request_id)`.
4. Optionally, after `unconfirmed`, `update_evidence(request_id, dest_tx, evidence_url)`
   and `check(request_id)` again.
5. `get_request(request_id)` reads back `status`, `reason`, `observed` (the facts
   the validators extracted) and `reasoning`.

## Files

- [`crosschain_execution_oracle.py`](./crosschain_execution_oracle.py): contract source
- [`tests/test_extraction_logic`](./tests/test_extraction_logic): offline logic tests
- [`fixtures/`](./fixtures): the evidence used as test inputs
- [`LICENSE`](./LICENSE): MIT
