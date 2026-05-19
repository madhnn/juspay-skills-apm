---
id: hyper-checkout
category: CHECKOUT
platforms: [android, ios, web, flutter, react-native, cordova, capacitor, iframe-web]
---

## What it is
Pre-built, customizable payment page UI that handles the entire checkout experience — card, UPI, netbanking, wallets, EMI, and more. The merchant creates a session on the backend, then launches the SDK which renders and manages the full payment UI.

## When to recommend
- User wants a ready-made payment page without building UI from scratch
- User wants Juspay to handle payment method display, orchestration, and redirects
- User's app is mobile (Android/iOS/Flutter/React Native) or web-based
- User wants the fastest path to accepting all payment methods

## Key concepts
- **Session**: Backend creates an order session; the SDK payload from the response is passed to the client SDK to launch checkout
- **Order Status API**: Always reconcile server-to-server after callback — never trust SDK result alone
- **Webhook**: Async notification for final payment status; idempotent handling required
- **iframe-web vs web**: Two separate web integrations — iframe embeds checkout in a div; web redirect is a full-page redirect

## Intent signals
checkout UI, payment page, payment screen, pre-built UI, hosted payment, accept payments, mobile payments, Android payment, iOS payment, Flutter payment, React Native payment, web checkout, iframe checkout, customizable checkout
