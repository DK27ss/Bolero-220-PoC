# Bolero-220-PoC

>**Chain:** Base
>
>**Date:** 2025-12-02 10:44:15 (UTC+1)
>
>**Protocol:** Bolero Finance Marketplace
>
>**Loss:** ~$221.45 USDC drained from 29 user wallets
>
>**Root Cause:** Missing `msg.sender` validation in `acceptOffer()`
>
>**Related Incident:** [0x0689a Pool Exploit](https://github.com/DK27ss/0x0689a-330K-PoC) — same attacker, ~1h30 earlier, ~$330K stolen
>
>| Contract | Address | Role |
>|---|---|---|
>| Bolero Marketplace (Proxy) | `0x356ef3f7B12D990A087dDC0f4F2E944D1B60347b` | Marketplace proxy |
>| Bolero Marketplace (Impl) | `0x977273A9D8AB0E50c8Cf8E143FBBb1F220bA3470` | Vulnerable implementation |
>| USDC (Base) | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | Drained currency |
>| Uniswap V3 Pool | `0x0E635F8EeED4F7279d56692D552F034ECE136019` | Flash loan source |
>| Attacker Factory | `0x3493e5233f227Bad26f3Da39Bf8380438d36cB37` | Deployer |
>| Attacker Logic | `0xf08Eee7503199AB628C97Ff8da181B2BBace52A6` | Exploit executor |
>| Fake Token | `0xc93F36214d7D0670c3B5391a597Cc8E4D1f726A6` | Worthless token |
>| Attacker Wallet | `0xD79E57C2800B7e1E575e8A72494FFea6C53850d6` | Final destination |

---

## Summary

Bolero Finance operates a marketplace on Base where users can trade tokenized music assets, the marketplace contract (`0x356ef3f7B12D990A087dDC0f4F2E944D1B60347b`) acts as a proxy, delegating calls to its implementation at `0x977273A9D8AB0E50c8Cf8E143FBBb1F220bA3470`.

Users who wish to trade on Bolero approve the marketplace contract to spend their USDC, this is standard ERC-20 workflow, the user calls `USDC.approve(bolero, amount)`, and the marketplace later calls `USDC.transferFrom(user, ...)` when a trade executes.

The vulnerability lies in **who** can trigger that trade execution.

---

## Root cause

## function `acceptOffer()`

```
acceptOffer(uint256 offerId, uint256 amount, (uint256,uint256)[] data)
```

selector: `0x8a47e524`

The `acceptOffer()` function executes a trade between an offer maker and the designated counterparty, during execution, it calls `USDC.transferFrom(counterparty, maker, price)` to move funds from the counterparty to the maker.

**The function does not verify that `msg.sender` is the counterparty** any address can call `acceptOffer()` and force the counterparty to pay, as long as the counterparty has previously approved the marketplace.

// Expected behavior

```
acceptOffer() should enforce:
    require(msg.sender == offer.counterparty, "unauthorized");
```

// Actual behavior

```
acceptOffer() performs:
    1. Check counterparty has enough balance and allowance   ✓
    2. Transfer fake tokens from maker to counterparty       ✓
    3. Transfer USDC from counterparty to maker              ✓  ← no sender check
    4. Transfer royalties                                    ✓
```

Any third party can trigger step 3, draining the counterparty funds

The vulnerability is a textbook **missing access control** on a state-changing function that moves funds on behalf of a third party.

The marketplace pattern is

```
User approves marketplace → marketplace calls transferFrom on user's behalf
```

This is safe **only if** the marketplace validates that the user consented to each specific trade, In Bolero case

- `makeOffer()` allows anyone to create an offer designating any address as the counterparty
- `acceptOffer()` allows anyone to accept the offer, triggering `transferFrom` on the counterparty
- **Neither function validates that the counterparty initiated or consented to the trade**

The approval granted by users to the marketplace was intended for legitimate trading, the attacker weaponized these approvals by creating offers for worthless tokens and forcing victims to "buy" them.

- **Fake token as attack vector**: The marketplace does not maintain a whitelist of tradeable tokens, allowing the attacker to deploy a token specifically crafted to bypass royalty checks and report infinite supply.
- **Flashloan for capital efficiency**: The attacker needed USDC balance to pass `makeOffer()` validation checks, A flash loan provided this capital at near-zero cost (0.01% fee).
- **No royalties extracted**: The fake token's `secondaryRoyalties()` returns `(0, 0)`, ensuring 100% of the victim's payment goes to the attacker with no protocol fee deducted.

---

## Exec Flow

The attacker exploited this in a single transaction with the following steps

## Stage 1 — Flashloan

The attacker deploys two contracts: a factory (`0x3493...cB37`) and the exploit logic (`0xf08E...52A6`). The exploit contract initiates a **20,000 USDC flash loan** from a Uniswap V3 pool (`0x0E63...6019`) with a 0.01% fee (2 USDC).

<img width="1414" height="341" alt="image" src="https://github.com/user-attachments/assets/eb93d2d9-c584-4dda-afec-002581b5a895" />

### Stage 2 — Fake Token Deployment

Inside the `uniswapV3FlashCallback`, the attacker deploys a **malicious ERC-20 token** (`0xc93F...26A6`, 650 bytes of bytecode), this token is designed to

| Function | Return Value |
|---|---|
| `balanceOf(any)` | Enormous value (~1e24) |
| `allowance(any, any)` | `type(uint256).max` |
| `transferFrom(any, any, any)` | `true` |
| `secondaryRoyalties()` | `(0, 0)` |
| `beneficiaries()` | `(marketplace, marketplace)` |

The token always reports infinite supply and allowance, succeeds on any transfer, charges zero royalties, and directs all fee recipients back to the marketplace (effectively nowhere), It costs nothing to "send" and generates no marketplace fees.

<img width="1223" height="162" alt="image" src="https://github.com/user-attachments/assets/9a17be3f-29b4-4f57-8ba3-129209a8f56c" />

## Stage 3 — Victim Recon

The attacker scans a pre-compiled list of addresses, checking each one for

1. **USDC balance** via `USDC.balanceOf(victim)`
2. **Allowance** to the marketplace via `USDC.allowance(victim, marketplace)`

Only victims satisfying `allowance >= balance > 0` are targeted, this ensures `transferFrom` will not revert during the forced trade.

<img width="1727" height="307" alt="image" src="https://github.com/user-attachments/assets/62994250-bbb8-45ef-853c-9d664ec74750" />

## Stage 4 — Drain Loop (30 iterations)

For each qualifying victim, the attacker executes two calls

// Step A: `makeOffer()`

```
makeOffer({
    amount:       1,000,000        // units of fake token
    price:        victim.balance   // exact USDC balance of victim
    currency:     USDC
    token:        fakeToken        // the worthless deployed token
    counterparty: victim           // the target address
    deadline:     future timestamp
})
```

<img width="1318" height="194" alt="image" src="https://github.com/user-attachments/assets/334c906a-015c-4b7f-8518-423cfd5abc04" />

This creates an offer on the marketplace: "I will sell 1,000,000 fake tokens, and the counterparty must pay their entire USDC balance."

// Step B: `acceptOffer()`

```
acceptOffer(offerId, 1_000_000, [])
```

<img width="1458" height="161" alt="image" src="https://github.com/user-attachments/assets/e4700d7c-09ef-4f5b-bc85-1c270652f46a" />

The attacker **accepts their own offer** on behalf of the victim, the marketplace

1. Transfers 1,000,000 worthless fake tokens from attacker to victim
2. Transfers the victim's **entire USDC balance** from victim to attacker
3. Transfers 0 USDC in royalties (fake token returns 0)

The victim receives worthless tokens and loses all their USDC, the attacker pays nothing because the fake token's `transferFrom` is a no-op.

## Stage 5 — Repay & Profit

After draining all 29 qualifying victims

1. **Repay** the flash loan: 20,000 USDC + 2 USDC fee = 20,002 USDC
2. **Remaining profit**: ~219.45 USDC
3. **Swap** 223.33 USDC to ~0.0794 ETH via Uniswap V2
4. **Send** ETH to attacker wallet `0xD79E57C2800B7e1E575e8A72494FFea6C53850d6`

5. <img width="1058" height="161" alt="image" src="https://github.com/user-attachments/assets/8cf9a085-4bc4-49e0-b38c-bb093efd9ca8" />

---

## Impact

29 wallets were drained of their entire USDC balance. Individual losses ranged from ~$5.34 to ~$10.04

| # | Victim | USDC Lost |
|---|---|---|
| 1 | `0x08F7...bA48` | 10.04 |
| 2 | `0xE92E...96D1` | 9.97 |
| 3 | `0x5C2C...E711` | 9.90 |
| 4 | `0x5Ca1...2DCb` | 9.78 |
| 5 | `0xBb61...16A2` | 9.27 |
| 6 | `0x2ec9...a935` | 9.24 |
| 7 | `0xE291...68F9` | 8.92 |
| 8 | `0xc1aA...ecfe` | 8.84 |
| 9 | `0xb673...f66a` | 8.70 |
| 10 | `0x3Dd4...d403` | 8.47 |
| 11 | `0xcEfa...221b` | 8.29 |
| 12 | `0xf635...92A9` | 8.26 |
| 13 | `0x97FC...4587` | 8.24 |
| 14 | `0x0c2F...4BB9` | 8.15 |
| 15 | `0x12aD...AfFD` | 8.08 |
| 16 | `0x553D...9ccc` | 7.97 |
| 17 | `0x6A35...FA69` | 7.38 |
| 18 | `0xE6DB...C642` | 7.25 |
| 19 | `0xb28B...6756` | 7.00 |
| 20 | `0x8Bd5...EE7b` | 6.58 |
| 21 | `0xf3A4...eDAd` | 6.31 |
| 22 | `0x218b...464C` | 5.96 |
| 23 | `0x5B3a...425a` | 5.91 |
| 24 | `0xD143...8AdE` | 5.81 |
| 25 | `0xfe72...69c9` | 5.59 |
| 26 | `0x8a30...1394` | 5.43 |
| 27 | `0x3202...4D25` | 5.39 |
| 28 | `0xAfF0...C2Cf` | 5.36 |
| 29 | `0xFB48...1158` | 5.34 |
| | **Total** | **$221.45** |

---

## Output

```
Total USDC at risk : 221449750 (raw 6 dec)

============= RESULTS
Victims drained    : 29
Total USDC stolen  : 221449750 (raw 6 dec)
Flash loan repaid  : 20,000 USDC + fee
Net USDC profit    : 219449750 (raw 6 dec)

  [1] 0x08F7Fd04e76daC7e4d9e343e0B5510379f5bbA48  lost 10041310 USDC (raw)
  [2] 0xE92EC418Ca707e6728040E519Ed7f2b5Dd8a96D1  lost 9970000 USDC (raw)
  ...
  [29] 0xFB487b81c51913B733E99b1927eb03fd27d01158  lost 5341405 USDC (raw)

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

---

## Attacker link to 0x0689a Pool Exploit

Approximately 1 hour and 30 minutes before the Bolero exploit, the same attacker drained **~$330,999 USDC** from Pool contract `0x0689aa2234d06Ac0d04cdac874331d287aFA4B43` on Ethereum mainnet, the full analysis of that incident is available at [github.com/DK27ss/0x0689a-330K-PoC](https://github.com/DK27ss/0x0689a-330K-PoC).

// Chronology

| Time (UTC+1) | Chain | Target | Method | USDC Stolen |
|---|---|---|---|---|
| **2025-12-02 09:13:23** | Ethereum | Pool `0x0689a...` | `collectInterestRepayment()` abuse | ~$330,999 |
| **2025-12-02 10:44:15** | Base | Bolero Marketplace | `acceptOffer()` abuse | ~$221 |

Two different protocols, two different chains, **91 minutes apart**.

Attack TX Bolero : https://app.blocksec.com/phalcon/explorer/tx/base/0x886b14e21f8ea7735b64b4b75e2cf11493dad636fefb1ea5a4c7c27160df6b3b
Attack TX 0x0689a : https://app.blocksec.com/phalcon/explorer/tx/eth/0x3a8bde0a17e04f6a119ae2f28e6b56ac736feb70761e2fa97cac25f816f751c2

## Identical Fingerprint

Both exploits share a methodology so specific that coincidence can be ruled out

| Dimension | 0x0689a Pool (Ethereum) | Bolero Marketplace (Base) |
|---|---|---|
| **Vulnerability class** | Missing `msg.sender` validation | Missing `msg.sender` validation |
| **Core mechanism** | Abuses `transferFrom` via victim's existing approval | Abuses `transferFrom` via victim's existing approval |
| **Targeted asset** | USDC | USDC |
| **Capital source** | Uniswap V3 flash loan | Uniswap V3 flash loan |
| **Attack pattern** | Call a protocol function with a victim's address as parameter, forcing `transferFrom` on their behalf | Call `acceptOffer()` with victim as counterparty, forcing `transferFrom` on their behalf |
| **Reconnaissance** | Scan for wallets with `allowance > 0` to the target contract | Scan for wallets with `allowance >= balance` to the target contract |
| **Execution** | Single atomic transaction | Single atomic transaction |
| **Profit extraction** | Swap USDC to ETH | Swap USDC to ETH |

1. **Same vulnerability pattern hunted.** Both attacks exploit the exact same class of bug: a protocol function that calls `transferFrom(victim, ...)` without verifying that `msg.sender == victim`, this is not a common exploit vector — it requires specifically scanning for contracts where an arbitrary caller can move another user's approved funds, the attacker was systematically hunting this pattern across chains.

2. **Same day, same session.** The 91-minute gap between attacks is consistent with a single operator pivoting from one target to the next, after draining $330K from the Pool contract on Ethereum at 09:13, the attacker moved to Base and hit Bolero at 10:44, this timing suggests the attacker had pre-identified both targets and executed them in sequence.

3. **Same operational playbook.** Both attacks follow an identical structure: flash loan USDC from Uniswap V3, exploit a missing access control to call `transferFrom` on victims, repay flash loan, swap profit to ETH, the code architecture, funding strategy, and exit mechanism are the same.

4. **USDC-only targeting.** Both attacks exclusively drain USDC, despite the target protocols handling other assets, this indicates a focused attacker with a single asset pipeline, likely funneling proceeds through a common laundering infrastructure.

5. **Cross-chain capability.** The attacker operated on Ethereum and Base within the same session, demonstrating multi-chain infrastructure (RPC access, gas funding, tooling) that is pre-prepared rather than opportunistic.

// Attacker Wallets

| Chain | Role | Address |
|---|---|---|
| Ethereum | Profit recipient (0x0689a) | `0x716420E787D20B255EeEd9f6D442fa454f1d7d5B` |
| Base | Profit recipient (Bolero) | `0xD79E57C2800B7e1E575e8A72494FFea6C53850d6` |

While the profit destination addresses differ (expected operational security), the behavioral fingerprint — vulnerability class, execution timing, code structure, asset targeting, and flash loan sourcing — forms a correlation too precise to dismiss. **This is the same operator conducting a coordinated approval-abuse campaign across multiple chains and protocols.**

---

// Timeline

| Time (UTC+1) | Event |
|---|---|
| 2025-12-02 09:13:23 | **First attack**: 0x0689a Pool drained on Ethereum (~$330K USDC) |
| 2025-12-02 ~09:15–10:43 | Attacker pivots to Base chain, prepares Bolero exploit |
| 2025-12-02 10:44:15 | **Second attack**: Bolero Marketplace drained on Base (~$221 USDC) |
| Same tx | Flash loan + 29 victims drained + swap to ETH |

Both exploits executed as **single atomic transactions** on their respective chains.

---
