# koryu-attestation

A public, tamper-evident, timestamped record of Koryu's daily crypto
measurement board and regime dial, so the record is independently
falsifiable rather than self-asserted.

Koryu ([koryu.io](https://koryu.io)) is the fifth guardian of the Shishin
Research house: a momentum monitor and a rules-based regime dial computed
from published formulas over public market data. This repository is the
proof layer. It mirrors the mechanism of
[shishin-attestation](https://github.com/Azoq/shishin-attestation).

## How it works

Two events per day:

- **Commit** (same day): the day's payload (the slimmed board + dial, plus a
  random nonce) is hashed, chained with the previous day's hash, and the
  hash alone is published to `chain/<date>.json` and OpenTimestamped
  (`chain/<date>.json.ots`). A hash reveals nothing; the nonce makes it
  un-brute-forceable.
- **Reveal** (a session later): the full payload is published to
  `reveals/<date>.json`. Anyone can then recompute the hash and confirm it
  matches the earlier, timestamped commitment.

The hash chain links every day to the one before it, so the record cannot be
re-ordered or a day quietly removed:

```
H[t] = SHA256( canonical(payload) || "|" || H[t-1] )
```

`canonical()` is compact JSON with sorted keys, UTF-8. Every number in the
payload is a string, so the serialization is byte-for-byte reproducible in
any language. The genesis hash is `SHA256("koryu-attestation/v1")`.

### Payload contents

- `date`, `board` (the slimmed daily board), `dial` (the regime dial's
  verdict + inputs), `nonce` — since genesis.
- `picks` — since 2026-07-19: the day's Strategy v1.0 selection (gate
  state, the ranked picks, and each excluded major with its reason), so the
  published strategy record is attested, not just recomputable. Additive:
  earlier payloads never carried the field and verify unchanged;
  `verify.py` hashes the revealed payload verbatim either way.
- `shortboard` — since 2026-07-22: the SHORT board, the bear-side mirror of
  `board` (same measured columns) plus the frozen BEAR score, the
  perp-`shortable` flag, and `held` — which coins the v2.0 shadow is actually
  short at this snapshot.
- `positions` — since 2026-07-22: what the v2.0 shadow HOLDS at this
  snapshot, both sleeves (`long`, `short`), each entry as symbol + entry
  price + dollar amount. Only the held position is committed; its running
  mark-to-market return is an outcome and stays out.

  Both joined the payload the same day and are additive on the same terms as
  `picks`: a day missing either hashes exactly as it did before the field
  existed, and `verify.py` hashes whatever fields the revealed payload
  carries. They are committed on the completed-bar data, before any
  live-price display overlay, so the hash never depends on intraday prices.

## Verify it yourself

Standard library only, no install:

```
python verify.py 2026-07-14
```

It reads `reveals/<date>.json` and `chain/<date>.json`, recomputes the
chained hash, and reports OK or MISMATCH. To also check the timestamp,
install the OpenTimestamps client and run `ots verify chain/<date>.json.ots`.

## What this proves, and what it does not

It proves each day's board existed at the committed time and has not been
edited since: the history cannot be backfilled or improved after the fact.
It does not prove the measurements are useful or "right" — the board makes
no prediction. See [koryu.io/verify](https://koryu.io/verify).
