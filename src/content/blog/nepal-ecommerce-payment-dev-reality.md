---
title: 'Building E-Commerce in Nepal: The Stuff Nobody Tells You'
description: 'The real state of building e-commerce and integrating payment gateways like eSewa, Khalti, and Fonepay in Nepal. No fluff, just the technical hurdles.'
pubDate: '2026-07-04'
---
If you are building an e-commerce site in Nepal, you are probably looking at integrating eSewa, Khalti, or Fonepay. On paper, it looks like any standard gateway integration. In reality, you are about to hit some weird, undocumented walls that will make you want to throw your router out the window.

I have spent the last three years building custom shopping carts and payment integrations for local businesses here. Here is the actual state of play, the engineering friction, and how to deal with it.

### The SDK wasteland

First, forget about installing a polished, community-maintained npm package or a clean Composer dependency. While some gateways have official packages, they are often outdated. You will find SDKs pointing to old endpoints, or written for Node v12. 

Your best bet is to bypass the helper libraries entirely. Just grab their raw HTTP API documentation, write your own clean Axios or Fetch wrappers, and handle the payloads yourself. 

Khalti is currently the most developer-friendly of the lot. Their documentation is relatively modern, and their test environment actually works most of the time. eSewa has improved with their newer EPAY APIs, but you still need to be careful with how you construct your signature. If you get a single character wrong in your HMAC generation, you will get a generic error screen with zero debugging context.

### The signature headache

Let's talk about that eSewa signature. They use an HMAC-SHA256 signature to verify transactions. The tricky part is the exact format of the message string you hash. It requires a specific sequence of parameters (total amount, transaction UUID, product code) joined by commas. 

Here is a tip that will save you three hours of debugging: make sure your total amount format matches exactly what you sent in the initial request. If you send `150.00` in the payload but hash `150`, the signature verification will fail on their redirect page. They also require a product code which they assign to your merchant account. If you use the sandbox merchant code (`EPAYTEST`) in production by accident, your checkout will silently fail or charge the wrong account.

### The callback reliability problem

In a perfect world, a user pays on the gateway's hosted page, the gateway redirects them back to your site, and your backend receives a webhook (callback) confirming the payment. 

In Nepal, you cannot rely on this flow. 

Users often close their mobile browsers the second they see the "Success" screen on Khalti or eSewa, before the redirect actually triggers. If your architecture only updates order status on the redirect route, you will end up with paid orders marked as "pending" in your database. This leads to angry customer phone calls.

To solve this, implement a two-step verification system:

1. Set up a background cron job or a queue worker.
2. For every pending order older than five minutes, query the gateway's transaction verification API directly using the unique transaction ID.
3. Update your database representation based on that API response, not just the frontend redirect.

Fonepay works a bit differently since it relies heavily on QR codes. If you are doing a web checkout integration with their merchant portal, the verification API is your only source of truth anyway.

### Address validation is a myth

If you are using Shopify or WooCommerce out of the box, the checkout fields will ask for a postal code. We do not use postal codes in Nepal. If you make that field required, your conversion rate will drop. 

Local logistics companies (like Upaya, Pathao Logistics, or Namaste Cargo) do not API-integrate with smaller shops. Most of them expect a phone call or a manual entry in their own portal. 

Because of this, keep your checkout forms stupidly simple. You need a name, a phone number, a city selection, and a free-text "closest landmark" field. "Right next to the yellow building behind the temple" is still the most accurate routing algorithm we have in Kathmandu. Don't waste time building complex address parsing engines. Just give them a large text area.

### The cash on delivery reality

Even with digital wallets everywhere, Cash on Delivery (COD) still rules. Depending on your niche, COD will make up 60% to 80% of your orders. 

This means your inventory system needs to handle holds. When someone orders via COD, you need to reserve that stock. Since fake orders are common, you also need an automated SMS verification step (using a local bulk SMS gateway like Sparrow SMS or Aakash SMS) before you actually dispatch the package. Send a quick OTP to verify the phone number. It costs a few paisa, but it saves you the cost of a failed delivery trip across town.