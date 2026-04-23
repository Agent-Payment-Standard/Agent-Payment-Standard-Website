# RFC: Know Your Agent (KYA) Standard

## Status: Draft
## Version: 1.0

---

## 1. Overview
The Know Your Agent (KYA) Standard establishes the mandatory baseline requirements for Payment Service Providers (PSPs) and autonomous software Agents. This standard operates as a foundational boundary—defining *what* must be achieved to ensure the security and integrity of the Agent payment process, while remaining agnostic to the specific technical implementation details.

The core objective of the KYA framework is to systematically verify that any Agent initiating a payment:
1. Represents a **Real User**
2. Holds explicit **User Authorization**
3. Operates within a **Clear Intent**

## 2. Mandatory Requirements

### Requirement 1: Protect Communications and Cryptographic Information
**Objective**: Ensure that all communication channels are secure and that the cryptographic keys—which form the basis of an Agent's identity—are never compromised.

*   **1.1 Secure Transmission channels:** All communications between the Agent, User, and the PSP must be transmitted over robustly encrypted and authenticated protocols (e.g., HTTPS, mTLS) to prevent eavesdropping and Man-in-the-Middle (MitM) attacks.
*   **1.2 Asymmetric Cryptography:** The identity and authorization mechanisms must rely on asymmetric cryptography.
*   **1.3 Key Generation & Local Storage:** The Agent must generate its cryptographic key pair entirely locally. The private key must be stored securely within the Agent's environment and **must never** be transmitted to the PSP, the User, or any third party.

### Requirement 2: Establish and Verify Agent Identity
**Objective**: Assign the Agent a verifiable, unforgeable identity that is recognized and trackable by the PSP.

*   **2.1 Agent Registration:** The Agent must complete a formal registration process with the PSP prior to any user interaction, providing its public key (or a Certificate Signing Request).
*   **2.2 Identity Binding:** The PSP must issue a unique Agent Identifier and securely, immutably bind this identifier to the Agent's public key or issued client certificate in its internal records.

### Requirement 3: Secure User Authentication and Authorization Binding
**Objective**: Guarantee that a real user is present, explicitly consents to the Agent's actions under defined constraints, and that sensitive payment credentials remain entirely isolated from the Agent.

*   **3.1 Real User Identity & Payment Capture:** The user must be authenticated securely. The collection of payment credentials (e.g., credit card numbers) **must** occur entirely within a secure environment hosted and controlled by the PSP.
*   **3.2 Zero Agent Exposure to Raw Data:** Sensitive payment data must be converted into a token (e.g., cross-merchant token). The Agent must never process, handle, or store raw payment credentials.
*   **3.3 Explicit Authorization & Intent:** The user must explicitly approve the Agent's access. The authorization interface must clearly display the Agent's identity and the strict boundaries of the authorization (e.g., payment limits, frequency, intent).
*   **3.4 Secondary Confirmation (Human-in-the-Loop):** The authorization process must establish explicit notification methods (e.g., Email, SMS, or Agent push notification). When a transaction exceeds the authorized limit or is flagged as high-risk by the PSP, these methods must be used to trigger a mandatory secondary user confirmation (Human present requirement).
*   **3.5 Strong Association Rule:** The PSP must create a persistent, secure data record that strongly binds together: `[Tokenized Payment Method + Agent Public Key/Certificate + User Authorization ID + User ID]`.

---

## 3. Best Practice Implementation (Non-Normative)

This section provides a detailed, compliant implementation of the KYA standard.

### Step 1: PSP Initialization
The PSP exposes secure endpoints for the Agent and User.
*   `register_endpoint`: `https://api.payments.com/agent/register`
*   `authorize_endpoint`: `https://api.payments.com/user/authorize`

### Step 2: Agent Registration (Establishing Identity)
The Agent generates a key pair and registers its public key.

**Agent Data Generation:**
*   `agent_private_key`: Securely stored in a Hardware Security Module (HSM) or encrypted local storage.
*   `agent_public_key`: `PUB_KEY_ABC`

**Registration Request (Agent -> PSP):**
```http
POST /agent/register
Content-Type: application/json

{
  "agent_name": "shopping-bot-v1",
  "public_key": "PUB_KEY_ABC"
}
```

**PSP Response:**
```json
{
  "agent_id": "agent_789",
  "client_cert": "CERT_DATA..."
}
```
*The PSP now stores the mapping: `agent_789` <-> `PUB_KEY_ABC`.*

### Step 3: User Binding and Tokenization
The Agent redirects the user to the PSP's hosted environment for card binding and authorization.

**1. Redirection:**
The Agent provides the user with a link: `https://api.payments.com/user/authorize?agent_id=agent_789`

**2. PSP-Hosted Interaction:**
*   The user performs real-name verification.
*   The user enters payment card details (e.g., Visa 4111...).
*   The PSP tokenizes the card into `tok_abc123`.

**3. Intent and Boundary Setting:**
The user defines limits (e.g., £100 per transaction) and chooses notification methods for high-risk challenges (e.g., SMS, Email, or Agent Push).

**4. Final Binding (PSP Internal):**
The PSP creates an `Authorization-ID` and stores the cryptographic bond:
```json
{
  "authorization_id": "auth_456xyz",
  "agent_id": "agent_789",
  "card_token": "tok_abc123",
  "limits": {
    "max_per_txn": 100.00,
    "currency": "GBP"
  },
  "notification_channel": "agent_push",
  "status": "active"
}
```

### Step 4: Handover to Agent
The PSP notifies the Agent (via callback or redirect) of the successful authorization.
**Completion Payload:**
```json
{
  "authorization_id": "auth_456xyz",
  "status": "authorized"
}
```
The Agent is now ready to initiate payments using `auth_456xyz` and its private key.
