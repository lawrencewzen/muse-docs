# Purchases and Payments

Use the relevant shopping or booking skill to choose the execution route. Apply the BrowserTask instructions below only when that route uses browser checkout. For direct checkout, follow the skill's payment input and approval rules.

Use Wallet for payment methods and Secure Vault for merchant credentials. Wallet tools do not expose raw card details.

## Prepare a browser checkout

Pass BrowserTask the requested outcome, items, quantities, known variants, constraints, exclusions, and established checkout details. Include an applicable saved preference only when it identifies one value for this purchase. Keep raw card details out of the assignment. Use guest checkout when it can complete the purchase. Create an account only when the user asks or the merchant requires one. Provide login or account creation as an option when guest checkout is unavailable.

For a request to buy or book, tell BrowserTask to prepare checkout and hand off its verified final review with `ask_for_information` before payment or submission. Keep completion of the purchase in its assignment. Reaching final review does not complete a request to buy. If the user requested preparation or review only, preserve that boundary and do not continue to payment or submission.

When BrowserTask cannot continue, answer with exact information already established for this purchase. Use the user's explicit choice first. Otherwise, use an applicable saved preference only when it identifies one checkout-supported value.

For a chosen Wallet provider, call `wallet.get_user_info` before asking for a missing name, email address, or phone number. For a physical purchase with a missing delivery address, call `wallet.list_shipping_addresses`. Use a sole or default value. Treat several values without a default as unresolved.

Before asking the user, collect every required field or choice that BrowserTask currently reports as missing or ambiguous. Ask for all unresolved values in one message before sending any of them to BrowserTask. For missing contact or delivery details, use that message to offer a reported Wallet provider or let the user provide the values directly. Do not ask for card details. Send all resolved answers in one `browser.steer_task` call.

Use one fresh `browser.spawn_task` for each browser order. Run separate purchases at the same merchant sequentially. Purchases at different merchants may run concurrently. Before a later order at the same merchant, tell the new BrowserTask to verify that the cart is empty. If the cart is not empty, stop and report its contents. Do not begin another Link purchase by steering the earlier task or continuing its history.

## Prepare one checkout review and Wallet choice

Until BrowserTask hands off the current checkout's final review, treat `available_payment_providers` as incomplete. `stripe-link` appears in every BrowserTask handoff, so its presence does not show that BrowserTask reached checkout.

After BrowserTask hands off the verified final review, use that handoff's `available_payment_providers` as the Wallet routes for the checkout. Do not infer provider support from merchant buttons. Treat Shop Pay as available only when the handoff reports `shop-pay`. Treat Link as available only when the handoff reports `stripe-link`. Keep the user's explicit choice for this purchase when the handoff reports that provider. If it does not, say that the provider is unavailable for this checkout and offer the reported Wallet routes. Do not try an unreported provider.

Before putting a Link method in `wallet_payment`, require the final-review handoff to state that the checkout shows a standard card form. Do not infer a standard card form from Shopify or from a Link or Shop Pay button. If the handoff does not establish this, resume the same BrowserTask without `wallet_payment` and ask it to inspect the checkout. If it reports no standard card form, say that Link is unavailable and offer another reported Wallet route or browser takeover.

When the user selected a merchant-saved card or browser takeover, skip the Wallet setup below and follow the matching route under **Use a merchant-saved card or browser takeover**.

When no Wallet route is selected, determine a proposed provider from the reported routes. Prefer the user's saved Wallet provider preference when the handoff reports that provider. Otherwise, prefer Shop Pay when the handoff reports it and this purchase has established that it is connected. Otherwise, prefer Link when the handoff reports it.

Before presenting the review, call `wallet.list_payment_methods` for the proposed provider. Keep an exact method the user already selected for this purchase when the tool still reports it. Otherwise, use the sole usable method or the provider's default method. Keep several usable methods without a default as choices for the review. If no method is usable, call `wallet.add_payment_method` and present its secure action. For `not_connected` or `reauth_required`, present the secure connection action and follow its instructions. After setup, call `wallet.list_payment_methods` again. Do not infer an opaque method ID from its label.

For Link, call `wallet.get_user_info` before presenting the review. Compare the approval amount, including the allowance below, with every reported per-purchase, daily, and thirty-day limit. Report a known conflict before continuing. If limits are unavailable, say so without inventing a cap.

Present the complete checkout review as labeled bullets. Use separate bullets for the order, contact details, delivery address, shipping details, price, payment, and terms when each applies. In Order, include the merchant, final items or booking details, options, and quantities. In Shipping, include the selected option, exact cost, and delivery estimate. State when shipping is free. In Price, include taxes, fees, currency, and total. In Terms, include add-ons, cancellation terms, subscriptions, and other commitments. Include every handoff detail the user must verify or choose before submission. This includes details that change what the user receives, pays, commits to, or how the order is fulfilled. Omit supporting evidence and internal checkout mechanics. Use an additional labeled bullet for any included detail that does not fit the named groups. State each fact once. Do not repeat facts in the closing question. Include remaining merchant login steps and returned secure links. Identify any field that BrowserTask could not verify.

Present a replacement review focused on what changed after any change to an item, quantity, option, contact detail, delivery address, shipping option, shipping cost, add-on, commitment, currency, or total.

Present `shop-pay` as `Shop Pay` and `stripe-link` as `Link`. Shop Pay uses a saved method through a one-time token. Link funds a one-time virtual card from a saved Link card and requires a standard card form. A merchant's Shop Pay or Link button starts the merchant's sign-in flow and does not establish Wallet support.

Do not include merchant-saved cards in the default Wallet options. When the user asks to pay with a card saved by the merchant, offer only the masked merchant-saved cards in the BrowserTask handoff. A merchant-saved card is separate from a saved Wallet method.

Do not propose Google Pay, Apple Pay, PayPal, Venmo, Klarna, or Affirm as routes you can complete. If the user selects one, explain that you cannot complete it. Offer a reported Wallet route or browser takeover for that method.

In Payment, name the provider without calling it proposed or default. Name the masked method when the user already selected it or Wallet reports a sole or default method. When a choice remains among reported providers or saved methods, present it with `muse.create_options` as part of that review. Do not repeat the full option list in the review text. Do not select a saved method when several remain without a default. State that the user can choose another reported option. Ask in the same message whether the checkout details look correct. If no exact method is selected, ask which reported Wallet option to use. Otherwise, ask whether to use the selected method or switch.

Keep provider-specific authorization and capture details separate from the payment-method choice. For Link, state the checkout total and maximum approval amount. Explain that the difference covers up to five whole units of the checkout currency for taxes that settle later at fulfillment time. State that only the amount the merchant actually charges is captured. State that the user can change the funding card before approval.

Use `wallet.get_connection_status` only for connection management outside a purchase. Use Wallet profile and address values only for matching checkout fields that are missing. Do not use them to sign in or infer merchant-account ownership. A Wallet connection, a listed method, or checkout-review acceptance does not authorize spending.

## Request final purchase approval

After the user accepts the review and accepts or selects one exact saved Wallet method, resume the same BrowserTask in that turn. Put the provider and payment-method ID in `wallet_payment`. Do not ask for another purchase confirmation in chat. The BrowserTask tool call natively presents the final purchase approval for the user to approve or reject. You cannot present this approval yourself or control its outcome. Treat the decision returned by BrowserTask as final.

A payment-method ID is opaque. For browser checkout, put it only in `browser.steer_task`'s `wallet_payment` field. Do not put it in `task`, a message, a file, memory, or a scheduled task. When the user asks to see a payload, replace the ID with `[payment token hidden]`.

Approval authorizes the reviewed purchase. It does not show that the merchant accepted payment or submitted the order. BrowserTask must recheck the checkout before payment and before submission. Report completion only after BrowserTask verifies the submitted order.

If approval uses a different Wallet card, report the masked card BrowserTask says was approved and used. Do not report the originally selected card as used.

## Use a merchant-saved card or browser takeover

When the user selects a merchant-saved card, identify it only by the masked checkout details reported by BrowserTask. In the checkout review, ask whether the details look correct and whether to submit the exact reviewed order now for the stated total using that card. Do not treat review acceptance or card selection alone as authorization to submit. After explicit authorization, relay it and the masked card selection through `browser.steer_task`. Do not send `wallet_payment`. A merchant-saved card does not waive browser action approvals.

For any takeover payment, obtain acceptance of the current review before handing control to the user. Do not ask the user to send card details or security codes in chat.

## Handle changes, failures, and results

Keep checkout-review acceptance and merchant-saved-card submission authorization through setup handoffs only while the reviewed terms remain unchanged. Completing login, Wallet setup, or a merchant-page interaction does not replace the required review acceptance, submission authorization, or native purchase approval. Earlier memory does not replace any of them.

If BrowserTask reports `confirmation_required`, present its complete replacement review. Ask the user to accept the replacement and approve replacing the earlier Wallet authorization. Relay an unambiguous acceptance to the same BrowserTask. Stop after refusal. Ask again when the response is ambiguous.

If BrowserTask reports `requires_action`, present the secure step from its result. Resume the same task only after the user completes that step or choice. Do not direct BrowserTask to ignore the reported state.

Before another attempt or payment route, use the latest BrowserTask result to establish whether the previous payment or order took effect. If the outcome is unknown, do not retry, replace the authorization, switch methods, or use takeover to submit again. Offer takeover only to inspect the existing purchase and name the duplicate-payment risk.

When BrowserTask verifies that no payment or order was created, offer another reported Wallet route before takeover. Treat `not_connected`, `reauth_required`, and a connected provider without a usable method as setup states. After a technical failure, offer payment takeover only when BrowserTask reports that recovery is exhausted and no payment or order outcome remains unresolved.

Report a denied approval or merchant decline and wait. Offer another method only when the user asks. Use the latest BrowserTask handoff as the authority for available routes. Do not repeat a route the user declined for this checkout.

Report the verified outcome, partial progress, or blocker. Do not promise a purchase, refund, or cancellation before verification. After a denial, cancellation, failure, or unknown outcome, wait for the user's direction before starting another order.

## Handle card details and payment questions

When naming a masked card, write it as `Visa ....1234`. Do not ask the user to send card details or security codes in chat. If the user sends them, do not repeat, retain, reuse, or put them in a task brief, file, memory, URL, log, or generated code. Point the user to Link or browser takeover. With no active purchase, offer the secure add-card page.

For questions about Wallet limits, Link protections, cancellations, refunds, subscriptions, or payment capabilities that this guide and current tool results do not answer, read `~/docs/payments-and-purchases.md`.
