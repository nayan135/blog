---
title: 'Notes on building e-commerce in Nepal: payments, COD, and the gaps nobody warns you about'
description: 'Practical notes for developers integrating payments in Nepal — eSewa, Khalti, FonePay, cash on delivery, and the quirks that don''t show up in docs.'
pubDate: '2026-07-06'
---
I've shipped three e-commerce projects for clients in Kathmandu over the past two years, and every single one hit the same wall around week two: payments. Not because the APIs are hard. eSewa's docs are fine, Khalti's are actually decent. The wall is everything around the API — the business reality of how Nepalis actually pay for things online.

Here's what I wish someone had told me before the first project.

**Cash on delivery is not a fallback option, it's the default**

If you're coming from a market where COD is a minority payment method, recalibrate. For a lot of merchants I've worked with, 60-70% of orders are COD. Daraz Nepal built its entire early growth on this. People don't trust putting card details into a site they've never heard of, and honestly, given how many half-built Nepali e-commerce sites exist with no SSL and a Gmail address as the "support contact," that distrust is earned.

So your checkout flow needs to treat COD as a first-class payment method, not an afterthought bolted on after you finish integrating the digital wallets. Build your order state machine assuming a chunk of orders will sit in "pending confirmation" for a day while someone calls the customer to confirm the address and phone number. Yes, calls. Not emails.

**The wallet trio: eSewa, Khalti, and now FonePay/ConnectIPS**

eSewa is the oldest and has the most users, especially outside Kathmandu valley. Khalti has better developer experience and a cleaner dashboard, in my opinion — their sandbox actually works reliably, which is not something I can say for every gateway here. FonePay integrates a lot of banks directly, so you're not asking users to have a separate wallet balance topped up; it can pull from their actual bank account.

ConnectIPS is the interbank one backed by Nepal Clearing House. It feels more "enterprise" and a lot of government and utility payments run through it, but the integration is clunkier and the UX for the end user involves more redirects and OTP steps than eSewa or Khalti.

My actual recommendation for a new project: integrate Khalti and eSewa first, add FonePay if your client's customer base skews toward bank-account users over wallet users (older, more established businesses tend to see this), and treat ConnectIPS as optional unless a client specifically asks for it.

**Verification is a two-step dance, always**

Every one of these gateways works the same way at a high level: you initiate the payment, redirect the user, they pay, they get redirected back to you with some token or transaction ID, and then — this is the part people skip — you have to hit a separate verification endpoint server-side to confirm the transaction actually succeeded before you trust the client-side redirect. eSewa's older v1 API had a known issue where the redirect could fire even on failed transactions in certain edge cases. If you don't verify server-side, you will eventually ship an order that was never actually paid for. I know this because it happened to a client's site before I got involved, and untangling the refund/reconciliation mess afterward took longer than the entire payment integration.

**Cards barely matter, but when they do, expect friction**

International cards go through gateways like PayU or direct bank merchant accounts, and getting a merchant account approved as a small business in Nepal is its own bureaucratic process involving your PAN registration, a company bank account, and sometimes a physical visit to the bank. If your client is a small shop selling to a local audience, don't bother building card support at all in v1. Nobody's asking for it.

**SMS OTP delays are a real UX problem**

NTC and Ncell OTP delivery for payment confirmation can lag by 10-30 seconds, sometimes more if the user's on a weak signal outside the valley. Design your payment waiting screen assuming this. Don't time out and cancel the transaction after 15 seconds like I did in an early version of one project — I had users texting the client asking why their money got deducted but the order still showed "failed." The fix was embarrassingly simple: bump the polling timeout to 90 seconds and add a "still verifying, don't refresh" message instead of a spinner that implies something's broken.

**One more thing on invoicing**

If your client is a registered business, they're legally required to issue VAT bills for transactions above a certain threshold, and IRD (Inland Revenue Department) has specific formatting rules for what counts as a valid bill number sequence. This has nothing to do with payment gateways directly but it'll come up in your order model design, so ask about it before you finalize your database schema, not after.