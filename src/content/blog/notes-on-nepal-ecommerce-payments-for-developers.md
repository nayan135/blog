---
title: 'Building payments for Nepal: what actually works'
description: 'Practical notes on eSewa, Khalti, ConnectIPS, and FonePay integration for developers building e-commerce in Nepal, plus why COD still wins.'
pubDate: '2026-07-03'
---
If you're building an online store for the Nepali market, the first thing to accept is that your payment gateway is the least important part of your checkout flow. Cash on delivery still moves more volume than every digital wallet combined for most small and mid-size merchants I've talked to. I learned this the hard way on a project last year where we spent three weeks polishing the eSewa integration and then watched 70% of orders come in as COD anyway.

That said, you still need the digital rails working, because the customers who do pay online expect it to just work. Here's what I've run into.

**eSewa** is the default choice because everyone's parents have it. The v2 API (the one using `epay-payment` and HMAC signature verification) is better than the old v1 form-post flow, but the documentation lags behind what's actually deployed. I had a case where the success callback URL fired before the payment status was actually marked complete on their backend, so if you hit their status-check endpoint too fast you get a stale `PENDING` response. Add a two-to-three second delay before your first status poll, or just retry with backoff instead of trusting the first response.

**Khalti** has a cleaner developer experience honestly. Their sandbox is more usable, the docs have real code samples in Python and PHP, and their support on Discord actually replies. The catch is their KYC requirements for merchants are stricter, and if you're integrating for a client who hasn't registered their PAN properly with Khalti's business side, you'll be stuck for days waiting on manual verification, not code.

**ConnectIPS** is what you use when a customer wants to pay directly from their bank account instead of a wallet. It's run by NCHL (Nepal Clearing House), and it's the closest thing to a proper bank-grade rail here. The integration is clunkier though. You're dealing with a token-based redirect flow and a shared secret you have to physically request from your bank, not just generate in a dashboard. If your bank is Nabil or NIC Asia this is usually quick. If it's a smaller bank, budget a week or two of back-and-forth with their IT department over email.

**FonePay** is worth mentioning separately because their QR code standard is what most physical POS and a lot of e-commerce sites use for scan-to-pay. If you're building anything with a physical pickup or hybrid checkout, supporting FonePay QR alongside eSewa and Khalti covers most people who don't want to type a mobile banking password into a browser.

A few things nobody tells you upfront:

Nepal Rastra Bank restricts cross-border card transactions in ways that will bite you if you're selling to the Nepali diaspora paying on behalf of family back home. International Visa/Mastercard processing exists through a handful of banks but the settlement and compliance overhead means most small merchants skip it entirely and just tell people to use Western Union or bank wire for anything cross-border.

SSL and domain trust matters more here than you'd think. A lot of customers still associate `.com.np` domains with legitimacy over generic `.com` for local businesses, especially older users who are wary of online payment in general. It's a small thing but it shows up in conversion numbers if you A/B test it.

Webhook reliability across all these gateways is inconsistent. I now treat every payment webhook as advisory, not authoritative. My order state machine never marks something paid purely off a webhook hit — it always cross-checks with a direct API call to the gateway's transaction lookup endpoint before finalizing. Doubles the API calls, but I've seen eSewa webhooks silently fail during their maintenance windows (usually late at night, no announcement) and orders would've stayed unpaid in my DB otherwise.

For testing, don't trust sandbox credentials to behave like production. eSewa's sandbox in particular has thrown 500 errors on webhook signature verification that never happened once we went live with real merchant credentials. Keep a manual test-order flow ready so you can sanity check with real, small-amount transactions before a launch, not just automated sandbox runs.