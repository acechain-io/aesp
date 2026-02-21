# Scenario: Weekly fresh food/supermarket automatic ordering (personal agent + supermarket agent + delivery robot)

Imagine an end-to-end scenario: **Supermarket sales agent, delivery robot agent, and personal agent** collaborate. **Every week** the personal agent automatically generates an order based on **historical consumption, nutritional combination, and doctor's advice**. **I can review and approve it once** at most, and then complete the order, payment, and delivery, and the cycle repeats**.

---

## 1. Participants and their respective Agents

| Role | Agent Responsibilities | Running State |
|------|------------|----------|
| **Personal** | Summarize historical consumption + nutritional goals + doctor's recommendations → Generate "This Week's Shopping List" → Submit for review/approval → Place orders, pay, and sign for confirmation on behalf of the user | Local/Extended + Optional backend (price comparison, nutrition calculation) |
| **Supermarket** | Provide product catalogs and prices, receive orders, confirm inventory and amounts, issue invoices/orders, and arrange delivery | Backend (interconnected with ERP, inventory, delivery) |
| **Delivery robot/delivery agent** | Receive delivery tasks, pick up goods, deliver them, request for signature (or take photos/verification), and return status | Backend + Internet of Things/robot terminal |

---

## 2. Main process once a week (simplified)

```
1. Trigger (such as every Monday at 8:00)
→Personal agent pull:
- Historical consumption (local or authorized access consumption records)
- Nutritional goals and taboos (user preset + optional doctor’s recommendations)
- Doctor's advice (obtained from clinic/health agent if authorized by user)
→ Generate "This Week's Recommended List" + estimated amount

2. Review (at most once)
→ Personal agent pushes the list to the user: "Please confirm or modify"
→ User: Pass / Fine-tune (delete a few items, change the quantity) / Reject
→ If rejected, no order will be placed automatically this week; if passed or fine-tuned, enter 3

3. Order placement and price negotiation (optional)
→ The personal agent sends the "confirmed list" to the supermarket agent as an E2EE message
→ Supermarket agent returns: available products, price, out-of-stock substitution, total price
→ If negotiation is supported: multiple rounds of agents from both parties (such as discounts, full discounts), and orders and invoices will be generated after reaching an agreement.

4. Payment
→ The supermarket agent issues encrypted invoices (amount, payee, order number)
→ Personal agent is judged by strategy:
- If the amount is within the "weekly budget" and the payee is on the whitelist → automatic payment can be made
- Otherwise → Play "Confirm Payment" to the user again
→ Payment completed → Both parties have audit records

5. Delivery
→ The supermarket agent sends the delivery task to the delivery robot agent (address, time window, order number)
→ Robot picks up, delivers, and requests for signature (such as user agent or person scanning/confirming)
→ Signature status is returned to the supermarket + personal agent

6. Over and over again
→ Run 1~5 again at the same time next week; historical consumption will automatically be included in "History" for next time recommendation
```

---

## 3. Correspondence to existing framework

| Links | Framework capabilities | Description |
|------|----------|------|
| List generation | Personal agent internal logic + optional "health/doctor" data source | Not necessarily implemented by Yallet; Yallet provides "identity + payment + communication" |
| Review once | Strategy: `allowAutoApprove` is true only for "orders that have been reviewed this week"; or `requestApproval` is forced to go through once before execution (total price of this order + list summary) | Guarantee "review at most once" is guaranteed by strategy/product |
| Personal ↔ Supermarket E2EE | `AgentMessage` (type: negotiation / invoice) + existing xidentity E2EE | Lists, quotations, orders, and invoices all use encrypted channels |
| Payment | `AgentExecutionRequest` (transfer) + `IAgentPolicyProvider` (this week's budget, supermarket is in the whitelist) | Automatic payment can be made if the policy is met; otherwise a pop-up window will confirm |
| Signing commitment | Off-chain signature or on-chain commitment (such as "I confirm receipt of order X") | `commitment` scope; optional personal_sign |
| Supermarket ↔ Delivery | Same as above, can be regarded as B2B agent message; delivery results are returned | The same message format and identity system |
| Audit | Every order, payment, and receipt `recordExecution` | Individuals can check "this week/historical automatic orders"; they can be exported in case of disputes |

---

## 4. Implementation of risks and responses in this scenario

| Risk | Response in this scenario |
|------|----------------|
| **Incorrect payment** | The supermarket payee is in the user's "supermarket whitelist"; payment must be made only when the order number is consistent with the invoice; the upper limit of a single transaction/single day/"weekly supermarket budget" |
| **Spend Out** | The strategy sets a "weekly food budget" cap (such as 500 USDC); `minBalanceAfter` retains the minimum living balance; if the budget exceeds the budget, a pop-up window must be confirmed |
| **Privacy** | History of consumption and doctor's recommendations should be visible locally or only to personal agents; only "product + quantity + delivery address" is provided to the supermarket, and complete health files are not provided; doctors recommend that if it is through a third party, use E2EE or desensitization |
| **Order placed by mistake** | "Review at most once" means showing the list to others before placing the order; if the user checks "Strictly follow the list, do not automatically replace", the product will not be automatically replaced when out of stock, and the user will be notified instead |
| **Dispute** | Signing for receipt is a commitment; if it is not received or delivered by mistake, you can go through the dispute process and audit records (order, payment, sign-in request) as evidence; connect with arbitration/supermarket customer service |

---

## 5. Strategy example (personal side)

```ts
// Concept example, non-final interface
{
  id: "weekly-grocery-001",
  scope: "auto_payment",
  vendorId: "supermarket-A",
  allowAutoApprove: true,
  conditions: {
maxAmountPerTx: "200", //Single order limit
maxAmountPerDay: "300", // Single day (if there are multiple orders in a week)
maxAmountPerWeek: "500", // Extension: Total budget for this week
    allowListXidentities: ["supermarket-A-xidentity"],
    token: "USDC",
minBalanceAfter: "100", // Keep at least 100 after payment
requireReviewBeforeFirstPay: true //Extension: There must be a review before the first payment this week
  },
  validUntil: endOfWeekTimestamp
}
```

"Review at most once" can be implemented as follows: **before the first execution this week**, send `requestApproval` (display list + estimated amount) first. After the user passes it, mark this policy as "reviewed this week". Only subsequent payments in the same supermarket and under the same policy in the same week can `allowAutoApprove`.

---

## 6. Summary

- **Supermarket sales agent + delivery robot agent + personal agent** form a closed loop: **Recommend → review → place an order → pay → deliver → sign for receipt**, repeated every week.
- Maximum of one review on the personal side** is guaranteed by strategy and product (must be approved before first payment; or the entire weekly list must be confirmed by the user before automatic payment is enabled).
- The whole process can fall on existing** messages (E2EE), execution (payment/commitment), strategy (limit, whitelist, reserved balance), audit**; risk response (mispayment, spent, privacy, disputes) can be specified in this scenario according to "AGENTIC_ECONOMY_RISKS_AND_MITIGATIONS".
- This scenario can be used as a **reference use case** to verify the framework interface (Policy, Message, Execution, Audit) and product form (review timing, budget cap, dispute entry).
