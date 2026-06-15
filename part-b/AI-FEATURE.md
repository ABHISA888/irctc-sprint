# AI Feature Specification: Conversational Action & Query Navigator (IRCTC Mitra)

## Problem It Solves
This feature addresses **Problem 6: Mobile Navigation Complexity**. Currently, critical features like checking PNR status, cancellation, and refund tracking are hidden behind multiple levels of hamburger menus and nested navigation, causing frustration for users (especially elderly and less tech-savvy individuals).

---

## Proposed Feature — User Perspective
The user sees a floating "Ask IRCTC" search bar and microphone icon at the very top of the mobile homepage.
* **Interaction**: Instead of navigating hamburger menus, the user taps the microphone or types a query in plain English or regional languages (e.g., Hindi, Tamil, Telugu).
* **Example Query**: *"Check my PNR 4281920192 status"* or *"Where is my refund for yesterday's failed booking?"*
* **Response**: The AI processes the intent and immediately pulls up a simplified overlay card containing the real-time information or routes the user directly to the target page with all fields pre-filled, bypassing menus entirely.

---

## Model or API Choice
We will use the **Google Vertex AI Gemini 1.5 Flash API** (specifically utilizing its Function Calling / Structured Outputs capability).
* **Why Gemini 1.5 Flash**: It is highly cost-effective, offers extremely low latency (< 800ms response time), supports multi-lingual inputs (essential for the diverse Indian demographic), and provides robust JSON outputs for intent routing.
* **Why not alternatives**: OpenAI GPT-4o is too expensive for high-volume public utility traffic. A custom classifier (like BERT) would require extensive hosting infrastructure and training data, which IRCTC does not have ready for deployment.

---

## Training or Input Data
The model does not need custom training from scratch but requires structured context in its system prompt and input payload:
1. **User Input**: The text query or audio transcription (converted client-side using Web Audio API).
2. **User Context**: Logged-in state, recent 3 booking PNR numbers, and latest cancelled transactions fetched from the client session cache.
3. **Application Routing Map**: A JSON mapping of platform actions (e.g., `CHECK_PNR`, `TRACK_REFUND`, `CANCEL_TICKET`, `SEARCH_TRAINS`) and the parameters they accept.

All data is gathered in real-time from the active user session and is not stored externally, preserving user privacy.

---

## How Output Is Shown to the User

When the user types: *"Track the refund for the ticket I cancelled yesterday"*

The AI returns: `{ "intent": "TRACK_REFUND", "confidence": 0.98, "parameters": { "booking_id": "BK_901827" } }`

The frontend immediately renders an overlay showing the **Interactive Refund Timeline Widget** (from Specs 4) directly over the dashboard:

```
+-----------------------------------+
| [=] IRCTC Mobile          [Mitra] |
+-----------------------------------+
|  "Tracking your refund for PNR    |
|   4281920192..."                  |
|                                   |
|  [x] Ticket Cancelled (June 14)   |
|  [x] Refund Initiated (June 14)   |
|  [-] Bank Processing (Active)     |
|      ARN: 827182910 [Copy]        |
|                                   |
|   [ Close Mitra ]                 |
+-----------------------------------+
```

---

## Confidence Threshold and Fallback

* **Confidence Threshold**: 0.85.
* **Low-Confidence Scenario**: If the model return confidence is below 0.85, or the API fails/times out, the system triggers the fallback.
* **Fallback Behavior**: The user is shown a structured "Quick Site-Map Navigator" menu:
  ```
  "Sorry, I couldn't quite understand that. Were you trying to:
   - Check PNR Status? [Go]
   - Track a Refund? [Go]
   - Cancel a Ticket? [Go]
   - Search Trains? [Go]"
  ```
  The app logs the low-confidence query anonymously for offline analysis and refinement.

---

## Success Metrics
1. **Time-to-Task Completion**: Reduce average mobile PNR/Refund check time from 40 seconds to under 5 seconds.
2. **Feature Discoverability**: A 50% increase in the usage of refund tracking and PNR checking from the mobile homepage.
3. **Intent Recognition Accuracy**: Maintain a classification success rate of > 92% based on user confirmations ("Did this help you? Yes/No").

---

## Limitations and Risks
* **Language Variance / Accents**: Regional dialects or spoken accents might result in incorrect transcription or intent parsing.
* **Hallucination**: The model might misidentify a random 10-digit number as a PNR and query the database.
* **Risk Mitigation**: The system will *never* perform destructive actions (like actually executing a cancellation or payment) solely via voice/AI. The AI will only navigate the user to the final confirmation page, where they must manually click the final button to authorize any changes.
