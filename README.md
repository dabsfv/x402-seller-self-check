# Seven things to check before an AI agent pays your x402 route

**What it is:** a plain-English guide to seven things that can go wrong between "the payment looks valid" and "the money has arrived" when you sell through x402, with a short self-check.  
**Who it is for:** anyone who sells, or is building, a paid API route or tool on x402 and is not a security specialist.  
**What it costs:** nothing. It is free, there is no signup, and there is nothing to buy.  
**How the self-check works:** it uses only throwaway test wallets and test money, on your own test copy. Never real funds, and never anyone else's service.

Only want the checks? Go straight to [SELF-CHECK.md](SELF-CHECK.md).

Last checked against public sources on 16 November 2026. No seller is named, and nothing here says any particular seller has any of these problems.

*By dabsfv.*

---

## The gap to start with

**Your content can go out before the money has moved.** That is true for every x402 seller, whatever you sell and whatever you charge. In the current specification's default flow the order is "verify, run the resource, settle, respond": your server is told the payment looks fine, does the work, and only then collects the money. Research has found software that replies before the collection finishes, and cases where one payment was counted more than once.

This post walks through seven places where there is a gap between "your server was told the payment looks fine" and "the money has actually arrived". It says what the research found, and ends with a self-check you can run on your own test copy in about five minutes.

**One figure for scale.** On 16 November 2026 (counted at 14:58 UTC), **968 of the 33,455 paid routes** in the public Bazaar catalogue (2.9%, from about 35 sellers) listed a pay-by-use payment option called `upto`. Pay-by-use is the kind of pricing where the gap matters most (check 7). The figure counts routes that listed the option, nothing more.

## What an outside scan can and cannot see

Automated scanners for x402 sellers already exist, and they are cheap: some are free, and others cost a few cents a check. They send your route an unpaid request and read what comes back: the `402` reply, its headers, and how your route is listed in public catalogues. That is useful, because it shows whether a buyer can find you and pay you at all. By their own descriptions, these scanners do not pay, so they do not see what happens after a payment is presented. That is where the problems in this post sit: one payment delivering twice, content going out before the money has settled, and a payment for one route unlocking another. Only a test that presents real test payments, with a test wallet on your own copy, can show those. The self-check below is that second kind of test. It does not replace an outside scan, and an outside scan does not replace it.

## Why I wrote this

I'm dabsfv, based in the UK. I build with AI tools, and I'm looking into whether there's a useful tool or service to build for people selling through x402. I read the research on how x402 payments can go wrong and have written it up in plain English. I'm not asking you to buy anything. At the end there is one question for you.

## How an x402 payment works, in five lines

1. A buyer, often an AI agent, asks for a paid route. Your server answers "402 Payment Required" and a price.
2. The buyer signs a payment and asks again, carrying it in a request header. In the current specification that header is `PAYMENT-SIGNATURE`; the older version, and the research papers, call it `X-PAYMENT`.
3. Your server asks a payment service, the **facilitator**, to check the payment.
4. Your server delivers the content.
5. The money **settles** on the blockchain a few seconds later.

In the current specification's default flow the order is "verify, run the resource, settle, respond". So by design your content can be produced before the money has moved. The seven checks are about that gap.

## The seven checks

Each has the same parts: what could go wrong, what the research found, where it stands, and which step of the self-check covers it. A "pass" means only that the check found no problem.

### 1. One payment, one delivery

**What could go wrong.** The same payment is accepted many times at once, so you are paid once and deliver several times.

**What the research found.**
- A conference poster tested 7 public services with bursts of 10 identical requests. 2 delivered more than once, and one of those 2 was a reference or demo site.
- One paper's best round on a test endpoint gave 248 deliveries from 1,000 simultaneous replays. Its count of how many times the money settled is an estimate.
- Another paper, against the official server, saw duplicate delivery in 3 of 50 rounds of 20 requests, never more than 2 deliveries in a round.

**Where it stands (last checked 16 November 2026).** No fix found. A public report of 4 of 5 simultaneous requests receiving the same settlement proof (issue #1805) was still open, and so was a request to add a rule against it to the specification (#3961). The specification still relies on the blockchain to stop a payment being reused. The changelogs of the latest official TypeScript packages (2.31.0, 12 November) show no change for duplicate payments.

**What you can do.** Write down that a payment has been used *before* you deliver, in one atomic step (not "look it up, then write it"). **Self-check:** step 4.

### 2. Deliver only after the payment has settled

**What could go wrong.** Content goes out as soon as the payment "looks valid", and collecting the money then fails.

**What the research found.**
- One paper found two third-party code libraries (Python, and Rust in an "optimistic" mode) that reply before settlement finishes. The Coinbase TypeScript reference library settles before replying.
- A paper that tried to empty a buyer's balance after delivery, against the official libraries on 8 test networks, reports that the attempt **failed**.
- On one streaming pay-by-use deployment, a burst of 50 requests a second against a 10-a-second settlement limit delivered all 50 and settled 5.

**Where it stands (last checked 16 November 2026).** A different failure, where the buyer is charged but refused because the payment service gave up too early, was fixed between August and September 2026 (its report, #1062, was closed on 21 September). A report that the Python Flask library settles only on a `2xx` reply (so a redirect is served unpaid; #3465) was still open.

**What you can do.** Wait for a successful settlement before releasing content; if the payment service may refuse, turn the request away before doing any work. **Self-check:** question A in the code table.

### 3. A payment is tied to the route it was for

**What could go wrong.** A payment for one route is accepted on any other route with the same price.

**What I observed, and what it is not.** On my own testnet copy of the official packages, used as documented, a payment for one route was accepted on another route with the same price and pay-to address. That is consistent with the published research below. It is not a claim that any package or any seller is vulnerable.

**What the research found.**
- In 100 of 100 test rounds, a payment for one route unlocked another route at the same price.
- The paper itself says the direct money loss is limited to routes of equal price; the harm is bypassed stock or access limits.
- Its June 2026 scan of the public catalogue found that 38% of hosts (331 of 868) had routes at the same price as one another. In the paper's own words that is exposure, not exploitation.

**Where it stands (last checked 16 November 2026).** No fix found for the standard payment type, and nobody is currently given the job. For the standard fixed-price payment on Base, the specification does not require a payment to be tied to a route, the signature does not cover the route, and the official TypeScript package (2.31.0) compares the payment's price, recipient, asset and network with the route's, but not the route itself. The documented examples do not add a check. A separate flaw in the sign-in extension, where a server trusted the route named by the client, has a published identifier (CVE-2026-113603, medium, published 20 October 2026; affected versions listed up to 2.28.0); the fix linked from the record (#3840) was still open.

**What you can do.** Today this is the seller's job, because nothing upstream does it. Do not rely on the route named inside the payment: the client fills that field in and the signature does not cover it. The signature does cover the recipient and the amount, so routes that must not unlock one another can be given different pay-to addresses or different prices. I tested this once on my own test copy: a payment for one route was refused on a route with a different pay-to address, and on a route with a different price. Routes that share both still unlock one another, so this separates groups of routes and does not tie a payment to one route. For limited items, such as stock, per-user data or quotas, count each delivery in your own application against the buyer or the order, not against "a valid payment arrived". A lasting fix needs the route to be part of what the buyer signs, which is for the protocol and the packages to change. **Self-check:** step 5.

### 4. Caches must not keep paid pages

**What could go wrong.** A web cache in front of your server stores the one paid reply and gives it to everyone.

**What the research found.**
- In the authors' own test with an nginx cache switched on and no protective header, all 1,000 unpaid requests got the paid content. With `Cache-Control: no-store, private` set, none did.
- One live test endpoint's paid replies were publicly cacheable.

**Where it stands (last checked 16 November 2026).** Partly fixed. From version 2.21.0 (4 August 2026), the changelog for the official TypeScript packages says they set `no-store` on `402` replies and settlement failures, and add `private` to paid `200` replies. On my own test copy (2.31.0) I saw exactly that: `no-store` on the `402`, and `private` without `no-store` on the paid reply. I did not check other languages.

**What you can do.** Send `Cache-Control: no-store, private` on every paid route and its `402` replies, and keep paid routes out of CDN caching rules. **Self-check:** steps 1 and 3.

### 5. Treat the payment header like a password

**What could go wrong.** The header is copied or changed on its way, or ends up in logs, and someone else uses it first.

**What the research found.**
- On a test network, an observer who saw a payment header used it first. The buyer was charged and the seller's own settlement then failed.
- A deliberately faulty proxy duplicated the header in every test; nginx and Caddy did not. The paper says duplication shows confusion, **not** a working bypass.

**Where it stands (last checked 16 November 2026).** No change found in the changelogs of the official TypeScript packages up to 2.31.0.

**What you can do.** Reject requests with more than one payment header. Keep the header out of logs and strip it before logging middleware. **Self-check:** question E in the code table.

### 6. Do not take the payment service's word for it

**What could go wrong.** You trust "paid" without checking the amount, the recipient and the network yourself.

**What the research found.** One paper shows that a seller who checks that the transaction is final on the blockchain itself keeps its protection even if the payment service is wrong.

**Where it stands (last checked 16 November 2026).** Nothing specific found for Base.

**What you can do.** Fix the facilitator in your configuration, check amount, recipient and network yourself, and treat any error or timeout as "not paid". **Self-check:** question D in the code table.

### 7. Pay-by-use prices cannot be overspent

**What could go wrong.** The buyer signs a small spending cap, then sends a burst. The content goes out and the settlement fails.

**What the research found.**
- On one pay-by-use deployment, 50 simultaneous requests delivered 47,277 units of work and settled 1,057. The paper's figures are for that one service.
- A $1.00 authorisation at $0.01 a call allowed up to 99 unauthorised calls (10 of 10 trials).

**Where it stands (last checked 16 November 2026).** Not verified fixed. Hooks for usage-based payments were added in August 2026; I did not check whether they close these cases. This check applies only if you use pay-by-use pricing; fixed-price sellers mark it "not applicable".

**What you can do.** Set the cap aside before doing the work, or settle in small steps and stop when one fails. **Self-check:** step 6, only if you use pay-by-use.

## The 5-minute self-check

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
| **5** (1 min) | Sign a new, unsent payment for your first route and present it once to **one other** paid route with the same price. For each further route, sign another new payment for the first route: once any route accepts a payment it is spent, and a refusal after that tells you nothing (stay within 20) | The other route rejects it | The other route returns paid content. (The payment also settles, to you.) With the official TypeScript packages as documented, expect this result when the routes share a price and a pay-to address: see check 3 for whose job the fix is. I observed this on my own testnet copy of the official packages used as documented; it is consistent with published research, and it is not a claim that any package or seller is vulnerable | 3 |
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

**Write down the date, the versions and each result, and "not tested" where you skipped.** A fail does not mean someone is attacking you. It means a behaviour the research describes is present in your setup, and the "What you can do" line under that check is where to start.

## What this post does not say

- That any seller has any of these problems. The research tested particular services and test setups, mostly other people's.
- That the seven checks find everything. Left out: how many block confirmations to wait for, smart-contract review, fake listings in catalogues, and how buyers' wallets and agents spend.
- That anything is fixed or unfixed for *your* version. That depends on what you have installed.
- Anything about a named seller. None is named.

## What I ran, and what I could not verify

**What I ran.** On 16 November 2026 I ran the self-check above, steps 1 to 6, on **my own test copy, not on any seller**: a small Next.js seller built with the official packages as documented (`@x402/next` 2.31.0), running on my own machine, on Base Sepolia, with the public test facilitator and a throwaway test wallet holding test USDC. I ran it twice: once as the official docs show it, and once with the `Cache-Control` header deliberately removed. One run each, with bursts of 10.

- **Step 1.** As documented: `402`, a `PAYMENT-REQUIRED` header, `Cache-Control: no-store` (pass). With the header removed: no `Cache-Control` (the "fail" look, as intended).
- **Step 2.** One `200` and one settlement, confirmed on the blockchain, in both runs. On the paid reply the official package sent `Cache-Control: private`, without `no-store`. My first attempt returned a `402` with a settlement error from the test facilitator; nothing was delivered and no money moved, and the second attempt worked.
- **Step 3.** `402` three times and no cache header in both runs. My copy has no cache or CDN in front of it, so this step could not show a cached paid reply. **The cache leak was not reproduced.**
- **Step 4.** One `200` and nine `402`s for one settlement, in both runs. **Duplicate delivery was not reproduced.**
- **Step 5.** In both runs, a payment signed for one route and not yet sent was accepted by another route with the same price and pay-to address, which returned its paid content. That is the "fail" look. It is what the specification and the package source lead you to expect (check 3), not a fault particular to my copy: the route is not part of what the buyer signs, and the package does not compare it. Each payment settled once, to my own pay-to address, so no money was lost. This is an observation on my own testnet copy of the official packages used as documented. It is consistent with published research, and it is not a claim that any package or seller is vulnerable.
- **The seller-side advice in check 3.** In a second small run the same day, I gave one route a different pay-to address, and then a different price. Each time a payment for one route was refused on the other, in both directions. One payment per case, once.
- **Step 6.** One `200` and nine `402`s; the one delivery settled for what it charged, in both runs. **Overspending the cap was not reproduced.**

That is one run, on one day, on one version, on a machine with no proxy or cache in front of it. "Not reproduced" does not mean "cannot happen": the research used far more attempts and other set-ups. And step 5's result is about my copy; it says nothing about any seller.

**What I could not verify.**
- **The research results.** Apart from the run above, every result in this post is from a research paper's own tests, on test setups or other people's systems. I did not reproduce any of them on anyone's service.
- **The papers.** Two are preprints and one is a conference poster. I read the text of all three but did not inspect their figures or open their data. The authors anonymised most of the services they tested, and the poster does not say whether the owners had agreed to the tests.
- **"Where it stands."** It comes from changelogs, public issues, advisories and the specification text. For check 3 only, I also read the parts of the official TypeScript source that sign a payment and match it to a route. Otherwise I did not review the source code, so "no fix found" means "none in the public record I read". I did not check the Python, Go or Java packages.
- **CVE-2026-113603.** I read the record itself (cve.org, the US vulnerability database and the GitHub advisory) on 16 November 2026. It lists affected versions up to 2.28.0 and no fixed version. I could not confirm whether later versions are affected.
- **The 968 figure.** My own count at one moment on one day; two counts earlier the same day each gave 966, and the catalogue changes while it is read. The number of sellers behind it (about 35) depends on how addresses are grouped into sellers; a count earlier the same day, grouped slightly differently, gave 36. I did not check which of those routes deliver before they are paid.
- **The commands on other set-ups.** They worked on my test copy with the official TypeScript client. I have not tried another client, another language, another operating system or a hosted staging copy.
- **Faucets.** The test USDC came from Circle's testnet faucet, which asked only for the wallet address, the network and the token (USDC), with no account. I did not try any other faucet.
- **Your exposure.** It depends on your versions, your hosting and any cache or proxy in front of you. Only a test on your own staging copy can show it.

## One question

If you sell through x402: **what's been the hardest part of getting x402 payments working for you?** To reply, open an issue on this repository (https://github.com/dabsfv/x402-seller-self-check), or tell me where you'd rather say it. If you'd rather not be quoted, say so and I won't. I'm asking so I can learn what people actually need.

---

Text licensed under CC BY 4.0 (see [LICENSE](LICENSE)). To report a problem with this post, or to answer the question above, open an issue on this repository (https://github.com/dabsfv/x402-seller-self-check) and see [CONTRIBUTING.md](CONTRIBUTING.md).
