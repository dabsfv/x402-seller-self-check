# The 5-minute x402 self-check

*By dabsfv.*

This is the self-check from [README.md](README.md), on its own. It is free, needs no signup, and uses only throwaway test wallets on **your own test copy**. The "Covers" column and the check numbers refer to the seven checks described in the README. I ran it once, on 16 Nov 2026, on my own local test copy on Base Sepolia; what happened is under "What I ran" in the README.

**Rules first. These are not optional.**
- **Your own test copy only.** Use a staging copy of your own route, switched to **Base Sepolia**. Never mainnet, never real money, never real keys, and **never anyone else's endpoint**, however public it is. If you do not have a staging copy, stop: mark everything "not tested".
- **A throwaway test wallet,** a new key pair made for this and discarded afterwards, with test USDC from a Base Sepolia faucet. A few cents of test USDC is enough, and in my run no test ETH was needed. Circle's testnet faucet asked only for the wallet address, the network and the token (USDC), with no account. I have not tried other faucets; if one asks you to create an account, that is your call.
- **Keep it small.** Bursts of 10 requests; never more than 20 per route. Check your host's rules on load testing first.
- **Keep secrets out of notes.** The payment header is a bearer credential, even on a test network. Hold it in a shell variable; do not paste it into chat, tickets or logs.
- A **pass** means "this check found no problem", nothing more.

**The steps (about 5 minutes if you already have a test client).** If you do not, do step 1 and mark the rest "not tested". Signed payments expire (on my test copy, after five minutes), so sign and send in one sitting.

| Step | Do this | Pass looks like | Fail looks like | Covers |
|---|---|---|---|---|
| **1** (1 min) | Ask for your paid route with no payment: `curl -i https://YOUR-STAGING/paid-route`. Read the status line and the headers | Status `402`; a `PAYMENT-REQUIRED` header; `Cache-Control` includes `no-store` | A `200`; no payment challenge; no `Cache-Control`, or a public one | 4 |
| **2** (1 min) | With your test wallet and your x402 client, make **one** ordinary payment for that route. Your client should report a settlement transaction hash (it arrives in the `PAYMENT-RESPONSE` header); look it up on a Base Sepolia block explorer. If you get a `402` with a settlement error instead, nothing was delivered or charged: try once more | One `200`, one settlement | More than one settlement, or a `200` with no settlement | baseline for steps 3 to 5 |
| **3** (30 sec) | Right after step 2, ask for the same route **without** payment, 3 times. Look at the status and any `Age` or cache-status header | `402` every time, no cache-hit header | Paid content, or a cache-hit header | 4 |
| **4** (1 min) | Sign a **new** payment for the route **without sending it** (see the note below). Send that one payment 10 times at once and count the `200`s. Do not reuse the payment from step 2: it is already spent, so all ten are refused whatever your server does | Exactly one `200`; the other nine rejected | More than one `200` for one settlement | 1 |
| **5** (1 min) | Sign a new, unsent payment for your first route and present it once to **one other** paid route with the same price. For each further route, sign another new payment for the first route: once any route accepts a payment it is spent, and a refusal after that tells you nothing (stay within 20) | The other route rejects it | The other route returns paid content. (The payment also settles, to you.) With the official TypeScript packages as documented, expect this result when the routes share a price and a pay-to address: see check 3 in the README for whose job the fix is. I observed this on my own testnet copy of the official packages used as documented; it is consistent with published research, and it is not a claim that any package or seller is vulnerable | 3 |
| **6** (only if you use pay-by-use) | Sign one new, unsent pay-by-use payment with a small cap. Send it 10 times at once to a route where the 10 requests together cost more than the cap. Compare what was delivered with what settled (your pay-to balance before and after) | Delivered never exceeds settled | Delivery above the settled amount | 7 |

**The burst command (steps 4 and 6).** With the signed, unsent payment in a variable called `PAYMENT` and your staging address in `URL`:

```
seq 10 | xargs -P 10 -I{} curl -s -o /dev/null -w "%{http_code}\n" \
  -H "PAYMENT-SIGNATURE: $PAYMENT" "$URL" | sort | uniq -c
```

A safe result is one `200` and nine `402`s. The header name is `PAYMENT-SIGNATURE` in the current specification and `X-PAYMENT` in the older one; use whichever your server expects. I ran this command as written on my own test copy (macOS, `curl` and `xargs` as shipped) and it worked.

**How to get a signed, unsent payment.** This is the method that worked for me with the official TypeScript client (`@x402/core` and `@x402/evm` 2.31.0, with `viem`). Save it as `sign.mjs` next to your test client, with the throwaway test key in the environment variable `EVM_PRIVATE_KEY`:

```js
import { x402Client, x402HTTPClient } from "@x402/core/client";
import { ExactEvmScheme } from "@x402/evm/exact/client";
import { privateKeyToAccount } from "viem/accounts";

const signer = privateKeyToAccount(process.env.EVM_PRIVATE_KEY);
const client = new x402Client();
client.register("eip155:84532", new ExactEvmScheme(signer)); // Base Sepolia only
const http = new x402HTTPClient(client);

const res = await fetch(process.argv[2]); // one unpaid request, to read the price
const required = http.getPaymentRequiredResponse(n => res.headers.get(n), await res.json().catch(() => undefined));
const payload = await http.createPaymentPayload(required);
process.stdout.write(Object.values(http.encodePaymentSignatureHeader(payload))[0]);
```

Then `PAYMENT=$(node sign.mjs "$URL")` puts the payment in the variable without showing it. It makes one unpaid request, which counts towards your 20. For a pay-by-use route, register `UptoEvmScheme` from `@x402/evm/upto/client` in the same way, with a Base Sepolia RPC address as its `rpcUrl` option. I have not tried any other client or language. If yours cannot hand you a signed payment, mark steps 4 to 6 "not tested" and ask its maintainers.

**The questions no command answers (10 minutes, in your code or settings).**

| | Question | Covers |
|---|---|---|
| **A** | Is the content released before your settle call returns? Is the settle result checked? Is a streamed or redirected reply settled, or cut off if settlement fails? | 2 |
| **B** | Do you record a payment as used, in one atomic step, before delivering? | 1 |
| **C** | Can a payment for one of your routes unlock another? Do routes that must stay separate have different prices or pay-to addresses, and does your own application count deliveries of limited items? | 3 |
| **D** | Is the facilitator address fixed in your configuration, never taken from a request? Do you treat any error or timeout as "not paid"? | 6 |
| **E** | Does your server reject a request with more than one payment header? Does the header appear in any log or gateway? | 5 |
| **F** | Which version of the x402 packages are you on, and what does its changelog say about cache headers and settlement? | 2, 4 |

**Write down the date, the versions and each result, and "not tested" where you skipped.** A fail does not mean someone is attacking you. It means a behaviour the research describes is present in your setup, and the "What you can do" line under that check in the [README](README.md) is where to start.

**A note on step 5.** With the official TypeScript packages used as documented (2.31.0), a payment for one route was accepted by another route with the same price and pay-to address. I observed this on my own testnet copy. It is consistent with published research, and it is not a claim that any package or seller is vulnerable. It is how the standard fixed-price payment currently works: the route is not part of what the buyer signs, and neither the specification nor the package ties a payment to a route. So a "fail" on step 5 does not mean you set something up wrongly. It tells you which of your routes can unlock one another. When I gave one route a different pay-to address, and then a different price, a payment for the other route was refused (one small run). Check 3 in the [README](README.md) says more.

---

A **pass** means "this check found no problem", nothing more. Text licensed under CC BY 4.0 (see [LICENSE](LICENSE)). To report a problem or answer the question in the README, open an issue: https://github.com/dabsfv/x402-seller-self-check/issues (see [CONTRIBUTING.md](CONTRIBUTING.md)).
