# Razorpay refunds with Parmana: a plain-language summary

Status: **As of 2026-11-08.** Test mode only. No live money has moved.

## What we are trying to show

An AI agent (or a person) can ask for a refund on a Razorpay payment. We want the refund to go ahead only when two things are true:

1. **Razorpay itself confirms** the payment is refundable for that amount. The agent cannot simply claim it is.
2. **A named manager has approved** that specific payment and amount, with a signed approval.

When both are true, the refund is made and a signed record is kept, so anyone can check later what was approved and by whom.

## Why it matters

Refunds are money leaving the business. An agent that can ask for a refund without a human decision, or on a claim nobody checked, is a risk. This setup puts the decision in a rule that a human wrote and approved, and it checks the facts with the payment provider.

## What is working today

- **The rules are in place.** The refund policy is approved and in effect. It requires the two checks above.
- **The approval chain works.** Every change to the rules was proposed by one person and approved by a different person, with a signature.
- **The payment provider check is set up.** Parmana can reach the service that asks Razorpay about a payment. The service is live and configured.
- **Requests without approval are stopped.** Two test requests without approval were refused. Nothing was refunded.

## What is not working yet

- **The Razorpay check is refusing Parmana's questions.** The service answers with an error rather than an answer. Until we know why, no refund can be approved end to end. We have added a log line to find out. It is waiting for the daily deployment limit to reset, which is about 24 hours.
- **No approved refund has run yet.** The full test, from approval to refund in Razorpay's test mode, has not happened.
- **Two shortcuts were taken and then corrected.** Two code changes were pushed without a review, which is against the team's rules. The rule-bypass was then switched off by the owner. One change still needs a review.

## What we can say honestly today

- We can say the **approval and rules are working as designed**: changes need two people and a signature, and unapproved requests are refused.
- We can say the **Razorpay check is live and reachable**, and that its key matches the one Parmana publishes.
- We **cannot yet say** that an approved refund runs end to end. That is the next test, and we expect to know within a day of the deployment limit resetting.

## What we need from you

1. **Confirm the approach** of a payment provider check plus a human approval before any refund, rather than one or the other.
2. **Decide on the code change** that is waiting for review (one log line). It can be kept after a review, or reverted.
3. **Choose when the first live-money refund may be considered**. Our recommendation is only after the test-mode run passes, a second person reviews the rules, and the remaining cleanup is done.

## Timeline

- **Today:** rules, approvals, the Razorpay check and the approval chain are set up. Two requests without approval were refused.
- **In about 24 hours:** deploy the diagnostic, fix the cause of the refused checks, and run the full test in Razorpay test mode.
- **After that:** review, clean up test leftovers, and decide on any live use.

## Risks

- The Razorpay check may need a fix that takes longer than expected.
- Logs are kept for only 30 minutes on the current hosting plan, so diagnosis must happen soon after a problem.
- Keys were shared in chat during the work. They have been replaced, but this is a process lesson for the team.
