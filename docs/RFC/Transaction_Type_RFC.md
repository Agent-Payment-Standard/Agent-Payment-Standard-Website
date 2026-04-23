# RFC: Agent Transaction Classification and Interaction Standard

## Status: Draft
## Version: 1.0

---

## 1. Overview
The Transaction Security Standard defines the mandatory boundaries for classifying and routing Agent-initiated payments. It differentiates between transactions that can be completed entirely autonomously by the Agent (**Unattended**) and those that require explicit human confirmation (**Attended / Human-in-the-Loop**).

The goal is to ensure full-link trust by verifying that the Agent's actions align with the user's initial intent and that appropriate insurance/risk mitigation is in place for autonomous machine decisions.

## 2. Mandatory Requirements

### Requirement 1: Transaction Classification
**Objective:** Ensure that every transaction driven by an Agent is correctly evaluated against KYA constraints and routed through the appropriate security flow.

*   **1.1 Dynamic Evaluation:** The PSP must evaluate every incoming Agent payment request to determine if it qualifies for autonomous execution or requires human intervention.
*   **1.2 Graceful State Transition:** A transaction initially submitted as autonomous must smoothly transition into a Challenge (Attended) state if risk or boundary checks are triggered. The transaction must not be processed autonomously once a flag is raised.

### Requirement 2: Boundaries for Autonomous Transactions (Unattended)
**Objective:** Establish the minimum security, traceability, and dispute-resolution requirements for payments executed without real-time human intervention.

*   **2.1 Strict Limit Adherence:** Autonomous transactions must strictly fall within the user's pre-authorized thresholds (e.g., maximum transaction amount, specific merchants) established during the initial KYA phase.
*   **2.2 Intent Verification & Retention:** The system must cryptographically or logically bind the user's original purchasing intent to the final transaction. The PSP must retain this intent and transaction data for a duration sufficient to cover standard consumer dispute windows (e.g., 30+ days).
*   **2.3 Consumer Protection Mechanism:** A defined financial liability or compensation mechanism (e.g., an insurance pool) must be established to protect the consumer in the event of Agent errors, hallucinations, or unauthorized purchases.

### Requirement 3: Mandatory Human-in-the-Loop (Attended)
**Objective:** Define the strict triggers and communication pathways required when a transaction demands explicit, secure user confirmation.

*   **3.1 Challenge Triggers:** A mandatory Human-in-the-Loop challenge must be initiated by the PSP if:
    *   The transaction amount exceeds the active Agent authorization limit.
    *   The PSP's risk-management system flags the transaction as high-risk.
    *   The user explicitly requested manual intervention for this specific type of transaction during the KYA phase.
*   **3.2 Secure Delivery Channels:** The challenge (e.g., a challenge URL) must be delivered via the notification methods explicitly agreed upon by the user (Email, SMS, or Agent push).
*   **3.3 Independent Human Verification:** The challenge must be resolved in a trusted, PSP-hosted environment (e.g., biometric, password) that cannot be simulated by the Agent.

### Requirement 4: Agent-Merchant Interaction Integrity
**Objective:** Guarantee that the order details negotiated between software Agents remain tamper-evident and consistent.

*   **4.1 Consistency Verification:** Final payment parameters (amount, merchant ID) must match the finalized order negotiated between User Agent and Merchant Agent.
*   **4.2 Abstracted Payment Sessions:** Agents must not exchange raw payment credentials. They must rely on PSP-generated session identifiers (Payment Request IDs) to securely bind orders to payments.

---

## 3. Best Practice Implementation (Non-Normative)

This section details the interaction flow between the User Agent, Merchant Agent, and PSP for an **Unattended Transaction**.

### Interaction Flow & Data Schema

#### 1. Order Negotiation (Agent-to-Agent)
The User Agent communicates its intent to the Merchant Agent.
*   **Protocol**: HTTP/JSON or Model Context Protocol (MCP).
*   **Data Example**:
    ```json
    {
      "action": "create_order",
      "items": [{"item_id": "sku_12345", "quantity": 1}],
      "agent_id": "user_agent_789"
    }
    ```

#### 2. Payment Session Creation (Merchant -> PSP)
The Merchant Agent requests a payment link from the PSP.
*   **Request Body**:
    ```json
    {
      "merchant_id": "merchant_amazon",
      "order_id": "order_abc_999",
      "amount": 50.00,
      "currency": "GBP"
    }
    ```
*   **PSP Response**:
    ```json
    {
      "payment_request_id": "pay_req_xyz123",
      "payment_url": "https://api.payments.com/pay/pay_req_xyz123"
    }
    ```

#### 3. Agent Execution (User Agent -> PSP)
The User Agent initiates payment using its `Authorization-ID` and the `payment_request_id`.
*   **Headers**: Includes KYA/VYA signatures (mTLS + SHA256 Signature).
*   **Data Body**:
    ```json
    {
      "payment_request_id": "pay_req_xyz123",
      "amount": 50.00,
      "authorization_id": "auth_456xyz"
    }
    ```

#### 4. Automated Risk and Insurance
*   **PSP Internal**: The PSP verifies that the amount matches the merchant's recorded `pay_req_xyz123`.
*   **Insurance Fee**: For unattended transactions, the PSP may deduct a small "Unattended Transaction Insurance Fee" (e.g., 0.5%) to cover potential "Agent Hallucination" risks.
*   **Storage**: PSP stores the User Intent vs. Final Receipt for 30 days for auditability.

#### 5. Handling Challenges (Transition to Attended)
If the amount were `250.00` (exceeding the `100.00` limit), the PSP returns:
```json
{
  "status": "challenge_required",
  "challenge_url": "https://api.payments.com/challenge/txn_999",
  "reason": "LIMIT_EXCEEDED"
}
```
The Agent then prompts the user via the pre-arranged notification channel (e.g., a push notification on the user's phone) to approve the specific transaction.
