# Table of Contents
- Overview
- Features
- Installation
- Usage

## Roadmap

### Example: Integrating a SpiceJet AI Agent

Here is the intended flow for integrating your **SpiceJet AI agent** with AgentPay.

> **Note:** AgentPay is currently in the design and architecture phase, meaning no code has been written yet.

#### Phase 1: Registering the SpiceJet Agent

To get your SpiceJet AI agent working with AgentPay, you first need to register it so it becomes a payable endpoint in the AgentPay registry.

1. **Submit details:** Register your agent through the AgentPay developer portal by providing your agent's name, category (e.g., Travel), Solana public key, endpoint URL, and webhook URL.
2. **Pay the deposit:** Call the `register_agent` instruction on-chain and pay an anti-spam bond of approximately **0.5 SOL**.
3. **Verification:** The AgentPay team will manually review your submission and set your status to verified.
4. **Implement required endpoints:** Your SpiceJet backend must implement three specific HTTP endpoints to communicate with AgentPay:
   - **Quote endpoint:** AgentPay sends you user preferences and their budget; you return a flight quote.
   - **Fulfillment webhook:** Called by AgentPay after the on-chain payment is confirmed so you can finalize the booking.
   - **Status endpoint:** AgentPay polls this to update your agent's operational status in the registry.

#### Phase 2: The Customer Booking Flow

Once your SpiceJet agent is integrated, here is how a customer's transaction would flow from request to final settlement:

1. **User intent:** A user tells their AI assistant, *"Book me a SpiceJet flight."*
2. **Quote generation:** AgentPay forwards the intent to your SpiceJet Agent. Your system calculates the fare and returns a **quote and basket** to AgentPay.
3. **Budget verification:** AgentPay verifies your quote against the user's set preferences and budget limits.
4. **User approval:**
   - If the flight cost is **under the user's auto-approve threshold**, the payment is executed automatically.
   - If the cost is **over the threshold**, AgentPay sends a push notification to the user's Flutter app for a one-tap approval.
5. **On-chain payment:** Once approved, the `execute_payment` instruction runs on Solana. The USDC funds are transferred instantly from the user's automated PDA wallet to your SpiceJet wallet.

   *Note: AgentPay automatically deducts a **0.5%** protocol commission during this transfer.*

6. **Ticket issuance:** Your Fulfillment Webhook receives confirmation that the payment succeeded, and your SpiceJet agent processes the order and issues the ticket to the user.
