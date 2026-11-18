# Paying with apps and checking the real transaction

My Tab never moves money. It opens the payment app ready to pay, then confirms the payment by
finding the transfer in the payer's own activity. The app side is built; the parts that need a
server are marked **Backend**.

## What happens

1. **Link an app** (Account → Payment apps): your handle (`@venmo`, `$cashtag`, PayPal.Me, Zelle)
   and, optionally, *Check my activity* (read-only).
2. **Pay** (an item, a friend, or Close tab) with Venmo / Cash App / PayPal. My Tab records the
   payment as *waiting*, with a short code like `MT-7Q2K`.
3. **Send with Venmo** opens the app with the friend, amount and a note `My Tab MT-7Q2K` filled in
   (`src/data/payapps.ts → payLink`). Cash App and PayPal.Me links fill in friend + amount but not
   the note. Zelle has no public link, so My Tab shows the friend's Zelle details instead.
4. **Back in My Tab**, it fetches your recent activity and matches it
   (`src/data/verify.ts → matchPayments`):
   - same app, money going **out**, **exact** amount (several tabs paid at once share one code and
     are matched against their total),
   - from 1 day before to 7 days after the payment was recorded,
   - and either the note contains the code, or the other side's name/handle matches and no other
     transfer fits equally well.
   A match confirms the payment and marks it *Verified in Venmo*; nobody has to tap "I got it".
   Anything unmatched stays *waiting* and can still be confirmed by hand.

`src/components/pay.tsx` has the picker, the hand-off block on the payment receipt, and a
`PaymentWatcher` that checks again whenever the app comes back to the foreground.

## Where the activity comes from

`src/data/transactions.ts` picks a source:

- **Demo** (no `EXPO_PUBLIC_API_URL`): a stand-in feed in which a payment appears after you tap
  *Send with …*. It's there so the whole flow can be tested now.
- **Backend** (`EXPO_PUBLIC_API_URL` set): `GET /activity?apps=venmo,paypal&since=…` returning
  `FeedTxn[]`:
  ```json
  [{ "id": "txn_123", "app": "venmo", "direction": "out", "amount": 2160,
     "date": "2026-10-03T18:04:00Z", "counterparty": "Maya Rivera", "note": "My Tab MT-7Q2K" }]
  ```

## Backend: reading each app

| App | How to read a user's activity | Notes |
|---|---|---|
| Venmo | **Plaid** (Link → `transactions/sync`), with the user's Venmo account as the institution | Venmo's own API is closed to new developers. Plaid gives amount, date and description; check in testing how much of the counterparty and note comes through. |
| PayPal | **PayPal Transaction Search API** for the user's account | Reading other people's accounts requires joining PayPal's partner program. |
| Cash App | **Plaid**, via the bank/card the Cash App transfers show up on | No public Cash App API for personal payments. Notes usually aren't available, so matching relies on amount + name. |
| Zelle | **Plaid**, via the user's bank | Zelle lives inside banks; no app link and no direct API. |

Keep Plaid/PayPal secrets on the server. Store access tokens per user, encrypted. Use Plaid's
`SYNC_UPDATES_AVAILABLE` webhook to re-check soon after a transfer instead of polling, and send a
push ("Maya's $21.60 is confirmed") when a match lands.

## Costs and rules

- Plaid charges for Transactions access (check current pricing); PayPal access depends on its partner program.
- The person whose activity is read must consent in the linking screen (Plaid Link shows this).
- Store only what's needed to match (amount, date, counterparty, note) and delete it after matching.
