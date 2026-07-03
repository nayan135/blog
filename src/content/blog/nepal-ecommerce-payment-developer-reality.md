---
title: 'Building E-Commerce in Nepal: What Nobody Tells You'
description: 'The reality of building e-commerce in Nepal. Integrating eSewa, Fonepay, and Khalti, dealing with address parsing, SMS costs, and cash on delivery.'
pubDate: '2026-07-03'
---
You get a client who wants a custom e-commerce site for their store in Kathmandu. You think, "Easy. I will throw together a Next.js frontend, an Express backend, run some migrations on PostgreSQL, and hook up a payment gateway." Then you actually start writing the code, and you hit the local wall. 

Building e-commerce here is different from building for the US or Europe. You cannot just drop Stripe Elements into a form and call it a day. Here is the actual state of things when you build transactional apps in Nepal.

### The payment gateway circus

Stripe does not support Nepalese merchant accounts. BrainTree is out. Your options are local digital wallets and the Fonepay network. 

eSewa is still the giant. If you do not have eSewa, you do not have a business. Their integration process used to be a nightmare of exchanging signed PDFs over email. It is slightly better now with their developer portal, but the technical implementation is still clunky. They expect you to redirect the user to their portal, and then they redirect back to your success URL with a query string. Then, your backend has to make a SOAP-like XML or JSON request back to their validation endpoint to confirm the payment actually happened. Do not skip this validation step. If you do, users can easily spoof the success URL params and get things for free.

Khalti is much nicer to work with from a developer perspective. Their API documentation is cleaner, their SDKs actually get updated, and their test environment does not randomly go down on weekends. 

Then there is Fonepay. They dominate QR payments in physical stores. Integrating their online gateway for web apps requires dealing with corporate bank structures. You often have to work through partner banks, which means signing physical papers in three different offices before you get your merchant API keys. 

```javascript
// A typical verification payload you have to construct for eSewa
const verificationData = {
    amt: totalAmount,
    rid: referenceIdByEsewa,
    pid: internalOrderId,
    scd: merchantCode
};
```

### The address parsing nightmare

There are no zip codes that people actually use. If you put a mandatory "Zip / Postal Code" field on your checkout form, your conversion rate will drop to zero. 

People in Nepal write addresses based on landmarks. A typical checkout address looks like: "Koteshwor, near the yellow bridge, opposite to the grocery store, Kathmandu." 

If you try to integrate Google Maps Places API to autocomplete addresses, you will realize Google does not map inner galleys well in Nepalese cities. Plus, the API costs will eat your margin. 

The meta right now is to use a three-tier dropdown for State, District, and Municipality/Ward, followed by a free-text textarea for "Street / Landmark". Do not try to parse this data automatically. Just pass it raw to the delivery team. They will call the customer on the phone anyway. Which brings us to the next issue.

### SMS is your primary database index

In the West, email is the unique identifier. In Nepal, nobody reads transactional emails. If you send an order confirmation to a Gmail address, it will sit in the Promotions tab forever. 

You need phone numbers. Your user table should probably use the phone number as the primary identifier or at least a strict unique constraint. 

This means you need an SMS gateway. Sparrow SMS and Aakash SMS are the common choices. Their APIs are simple HTTP GET or POST requests, but you have to budget for the cost. Each OTP or order update costs around 1.2 to 1.5 NPR. If you have a high-volume store with lots of spam registrations, your SMS bill can quickly spiral. You must implement aggressive rate-limiting on your OTP endpoints. Limit it to one OTP per phone number every 2 minutes, and block IPs that request more than 5 OTPs an hour. 

### Cash on delivery (COD) is still king

Even with eSewa and Fonepay on every phone, around 70% of e-commerce transactions in Nepal are Cash on Delivery. 

As a developer, this changes your state machine. An order is not "Paid" when it is created. It goes through: `Pending` -> `Confirmed (via phone call)` -> `Dispatched` -> `Delivered & Paid` or `Returned`. 

You have to write admin dashboards that make it incredibly easy for operational staff to change these states in bulk. Delivery drivers need a mobile-friendly view where they can mark an order as "Paid" the moment they collect the physical cash. 

### What to do next

If you are starting a project today, use Khalti for your initial payment integration to get your prototype working fast. It will save you days of administrative back-and-forth. Keep your checkout forms dead simple. Do not force users to sign up with an email; let them log in with a phone OTP. And build your database schema expecting that payment status will change hours or days after delivery.