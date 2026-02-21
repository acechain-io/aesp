# Agent Swarm / Agent array: manufacturer product form and subscription model

As a **manufacturer**, you provide users with a set of "**Agent Swarm**" (or **Agent Array**) solutions: skills similar to OpenClaw, presented as **menu options**. The user **checks the function array** they need → **owns and starts** the array → the agent in the array **self-runs**; the user pays a **monthly subscription** for this. This article describes the product form, how it interfaces with the Yallet framework, and how to pay for subscriptions.

---

## 1. Product form overview

| Dimensions | Description |
|------|------|
| **Name** | Agent Swarm / Agent Array (can choose one or both) |
| **Analogy** | OpenClaw skills: The user selects abilities from the "Skills Menu", which are scheduled and executed by the system after installation/activation; here is the "agent menu" → user selects function array → own and start → self-run. |
| **Provider** | You (manufacturer): Responsible for menu design, agent logic/hosting, protocol docking with user wallets, subscription and rights management. |
| **User** | User: See the "Optional Agent Array" menu in Yallet (Extension/Mobile), check the ones you need, and pay the monthly fee. After startup, these agents will run automatically under their identity and policy. |
| **Charge** | Monthly subscription (subscription): Users pay monthly to obtain "the right to use the array"; **automatic renewal** can be combined with existing agent capabilities (debited by the wallet if the user policy allows). |

---

## 2. User movement (from selection to self-operation)

1. **Look at the menu**
Users see the "Agent Swarm / Array" **function menu** (similar to the skills list) on Yallet or your landing page, for example:
- Lifestyle: fresh food/supermarket, subscription renewal, payment, health related
- Work: Invoice classification and payment, contracts/commitments, freelance payments, procurement price comparisons
- Finance: DCA, budget reminder, reimbursement assistant
Each item can include a brief description and required permissions (such as "authorization for automatic payment is required" and "access to certain types of data is required").

2. **Choose your own array**
The user checks the required items (multiple selections are possible) to form a **function array exclusive to the user**; unselected items will not be deployed and will not occupy their strategies and budgets.

3. **Subscription and Payment**
The user selects the subscription period (such as monthly payment/annual payment) and completes the first payment; the monthly fee can be automatically renewed if allowed by the policy through "in-wallet subscription" (see next section).

4. **Own and Launch**
- **Owned**: This array is bound to **User Wallet Identity**; policies (budget, whitelist, upgrade rules) are configured on the user side, or you provide a default template and take effect after user confirmation.
- **Startup**: When the user clicks "Start Array" or the first payment is successful, it is deemed to be activated; each agent in the array **self-runs** according to the policy and trigger conditions (timing, events, other parties' initiation, etc.), and those that require review are pushed or pop-up through Mobile/Extension.

5. **Everything comes around**
The array continues to run; users can **add or delete** array items in the menu at any time (which may trigger changes in subscription levels), adjust strategies, or shut down; monthly fees will continue to be deducted according to the subscription terms until cancellation.

---

## 3. Interconnection with Yallet Agent framework (you as the manufacturer)

Your role in the framework is that of a vendor: providing policy sources, optionally processing messages, and optionally running the agent backend; the user's wallet is responsible for identity, execution, and auditing.

| Framework capabilities | Your implementation |
|----------|--------------|
| **Policy** | Implement `IAgentPolicyProvider`: Based on the user's **selected array** + subscription status, return the user's policies under each scope (auto_payment, negotiation, commitment); a "default policy template" can be provided for user confirmation or fine-tuning. |
| **Menu/Array Configuration** | Maintain the "optional agent list" (id, name, description, required scope, default policy fragment); the user's selection result is saved as "the user's array configuration" and used as input for policy and equity judgments. |
| **Execution** | Does not directly activate the user's wallet; the framework calls the wallet core to execute (transfer, signature) according to the policy. You can run **agent logic** (recommendation, price comparison, negotiation with the other party's agent), and initiate `requestExecution` or `ReviewRequest` through the framework when payment/signing is required. |
| **Message** | If the agent in the array needs to communicate with the other party or your backend, implement `IAgentMessageHandler` to send and receive E2EE's `AgentMessage`; the user's wallet serves as the E2EE endpoint, and you do the business processing. |
| **Audit** | Use the framework's `IAgentAuditReader` / `recordExecution`; audit records can be checked on the user side or your side for reconciliation, disputes, and compliance. |
| **Subscriptions and Rights** | Monthly subscription determines "whether the user has the right to use this array/certain items"; if it is not renewed, your `getPolicies` returns empty or only returns the "trial/downgrade" policy, and the array is actually deactivated until renewal. |

In this way, **Agent Swarm / array** = "menu + policy template + agent logic + subscription rights" provided by you; **execution and identity** are always in the user's wallet, which is in line with the division of "the wallet is the base, and the manufacturer is the provider of strategies and capabilities".

---

## 4. Collection and payment of subscriptions (how to implement monthly fees)

- **Payee**: You (the manufacturer); the payment address or payment identity is fixed to facilitate user whitelisting and automatic renewal.
- **Trigger**: Before each month expires (such as D-1 or D-3), your backend or user-side "subscription agent" initiates a **renewal payment** (amount = monthly fee, payment = you, remarks = user id + period).
- **Automatic renewal**: If the user allows "automatic payment of your monthly fee" in his wallet policy (whitelist + fixed amount + once a month), the deduction can be completed without disturbing the user; otherwise, **ReviewRequest** is pushed to Mobile/Extension, and the payment is made after the user confirms.
- **Certificate**: After the payment is successful, you will record "This user has paid this cycle"; optional on-chain or verifiable certificates can be used for equity verification and disputes.
- **Cancellation**: After the user cancels the subscription, no renewal will be initiated in the next cycle; you stop or downgrade the array rights based on the "paid status".

That is: **The monthly fee itself can use the same set of agent capabilities** (automatic payment or push review), forming a closed loop.

---

## 5. Comparison with OpenClaw skills

| Dimensions | OpenClaw skills | Agent Swarm / Array |
|------|-----------------|---------------------|
| **Menu** | Skills list, CLI/configuration visible, searchable for installation | Agent function menu, user check to form a personal array |
| **Select** | `clawhub install <skill>`, enable/disable as needed | The user checks the required items and "owns" the array after subscribing |
| **Run** | The skill is scheduled for execution in the OpenClaw session | After the array is started, each agent runs automatically under the user policy (timing/event/initiated by the other party) |
| **Benefits** | Open source/community, no mandatory subscription | Monthly subscription, no benefits or downgrade if not paid |
| **Identity and Execution** | Local process, no unified wallet identity | Executed in the user's Yallet wallet, unified identity and payment |

Common features: **menu-based capabilities, user selection, available after selection, enable/disable**. Difference: Agent Swarm emphasizes wallet identity, payment and commitment, subscription fees, self-operation and strategy/auditing, and is more focused on "economic and compliance" scenarios.

---

## 6. Summary and landing points

- **Agent Swarm / Agent Array**: The "agent function menu" provided by you as the manufacturer; users **select their own array** → **monthly subscription fee** → **own and start** → agents in the array **self-run**, and push/Mobile review is required for confirmation.
- **Framework docking**: You implement Policy Provider (output policies based on array + subscription status), optional Message Handler and agent backend; execution and auditing are in the user wallet, conforming to the existing interface.
- **Monthly fee**: Through the same set of automatic payment/ReviewRequest capability deductions, you control the array rights and interests according to the payment status.
- **Analogy**: Consistent with the "menu + user selection + enable operation" of OpenClaw skills; the difference lies in the wallet base, payment and subscription, self-operation and policy closed loop.

The next step that can be implemented: add "**Manufacturer and product portal**" on the Yallet side (such as "Agent Swarm" product page + array selection UI); provide policy interface and subscription verification on your side; you can first make 1 to 2 array items (such as fresh food + subscription renewal) in a closed loop in the first phase, and then expand the menu and pricing range.
