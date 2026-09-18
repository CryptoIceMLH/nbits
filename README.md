# SHA256 Shitcoin Detector — How It Works (via `nbits`)

A way to tell, from the mining protocol alone, whether a rig is pointed at **real Bitcoin** or at some other SHA‑256 chain (BCH, BSV, or random alts). It can't *name* the coin — but it can tell "this is not Bitcoin‑grade work."

## The signal: `nbits`

Every Stratum job (`mining.notify`) carries an **`nbits`** field. This is the **compact encoding of the network target** — the difficulty a *block* must beat on that chain, right now. It's a hard consensus value: a pool can't fake it without the share math falling apart. We already receive it in every job (`job.nbits`).

That `nbits` *is* the live network difficulty — whatever it happens to be at that moment, up or down (Bitcoin retargets both directions every 2016 blocks).

## Why the *pool* difficulty does NOT work

The pool's share/vardiff (e.g. 1000–4000) is sized to **your hashrate**, not the coin. A Bitcoin pool and a shitcoin pool both hand you roughly the same share difficulty. Useless for this. **Only `nbits` (the network target) distinguishes the chains.**

## The math

```
nbits = compact target (4 bytes)
exponent = nbits >> 24
mantissa = nbits & 0x007fffff
target   = mantissa * 2^(8 * (exponent - 3))

difficulty = difficulty_1_target / target
           = (0xffff * 2^208) / target
```

`difficulty` is the number we compare. Decoding `nbits` is a handful of integer ops — no network calls, no extra data.

## The scale (why it's reliable)

| Chain            | Network difficulty (order of magnitude) |
|------------------|------------------------------------------|
| **Bitcoin**      | ~100 **trillion** (10¹⁴), and rising long‑term |
| Bitcoin Cash     | ~1–5 trillion (≈1–5% of BTC)             |
| Bitcoin SV       | lower still                              |
| Random SHA‑256 alts | millions–billions (10⁶–10⁹)           |

Bitcoin sits **1–6 orders of magnitude above everything else.** Even a big downward retarget (e.g. a −10% adjustment, or worse) leaves BTC near ~100T — nowhere near the others.

## The rule

```
if network_difficulty(nbits) < 100T  ->  NOT Bitcoin
```

`THRESHOLD = 100 trillion`. Bitcoin is ~139T today; BCH, BSV and every alt sit far below. 100T flags everything that isn't Bitcoin while leaving BTC clear. One constant (margin note below).

## Why it can't be faked or blocked

The check runs **in the firmware**, off the `nbits` in the job — which makes it un‑evadable two ways:

1. **Nothing to block.** There's no external lookup (no mempool.space, no API). The threshold lives in the binary; the difficulty comes from the job itself. A shitcoiner firewalling the rig achieves nothing.

2. **The pool can't fake `nbits`.** `nBits` is one of the six fields **inside the 80‑byte block header that gets hashed** (`version‖prevhash‖merkle‖time‖bits‖nonce`). The chip hashes whatever `nbits` the pool sent. If a pool lies and sends a Bitcoin‑grade `nbits` to spoof the check, every "block" the rig produces encodes the wrong difficulty bits and is **rejected by that chain's own consensus** — zero valid blocks. To get usable work they are *forced* to send the chain's real (low) `nbits`. The honest signal is the only one that mines.



## What it does

It **shames, and throttles** . after 2 nbits hits: a "shitcoin" frame appears in the OLED rotation + a banner in the web UI. The asics will roll back to 100mhz.

## Honest caveats

- **Testnet/signet** would also flag (low difficulty) — arguably correct, it isn't mainnet BTC.
- **Threshold margin:** 100T sits  under today's BTC (~139T). Difficulty retargets both ways; a normal −10% dip is fine, but if a major hashrate exodus ever pushed real BTC toward 100T, then we would need to lower the threshold to restore margin.
- The only structural gray zone is a future SHA‑256 chain whose difficulty climbs near Bitcoin's. None do today.
