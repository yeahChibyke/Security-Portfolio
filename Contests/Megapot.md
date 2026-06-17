![](../logo.png)
<img src="../img/megapot.png" alt="Megapot" height="320" />

# Security Report for the Megapot Contest

https://code4rena.com/audits/2025-11-megapot

### [M-1] Changing `referralFee` Mid-Drawing Impacts Some Users Negatively

**Summary:**

`referallFeee` is not snapshotted in the `DrawingState` when a drawing initializes, instead `Jackpot::buyTickets()` reads it directly from global state. This allows admin to change the referral fee percentage mid-drawing, causing users who purchase tickets at different times within the same drawing to contribute different amounts to the LP pool despite paying the same ticket price and having identical winning opportunities.

**Description and Root Cause:**

When a new drawing initializes via `Jackpot::_setNewDrawingState()` critical parameters are copied from global state into the `DrawingState` struct. `referralFee` is however not included in this struct.    

When users purchase tickets, `Jackpot::_validateAndTrackReferrals()` calcuates referral fees using the global `referralFee`, a value which can be updated during an active drawing by the admin when `::setReferralFee()` is called. This means users buying tickets at different times contribute different amounts to `lpEarnings`, even though they pay the same `ticketPrice`.

**Impact:**

- Users purchasing tickets at different times receive different economic treatment despite paying the same ticket price, competing for the same prize pool, and having identical winning probabilities.
- LP Pool Earnings becomes unpredictable as LPs cannot predict their returns when admin changes `referralFee` mid-drawing.

**Recommended Mitigation:**

The protocol can either add modifier to prevent admin from updating changes mid-drawing, or modify `DrawingState` struct to include `referralFee` and reflect these changes in the logic of `_setNewDrawingState()`, `_validateAndTrackReferrals()`, and `buyTickets()` functions.

