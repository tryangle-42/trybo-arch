Use cases to complete for the next main demo for Atul Sir(Main Investor):

1. **OTP / Delivery Handling via WebRTC**
   - Give a WebRTC link (e.g., in Amazon delivery comments)
   - Delivery agent calls the link for OTP/coordination
   - Trygo/agent receives and handles the call instead of personal number

2. **Insurance Vendor Comparison**
   - Search ~5–6 insurance companies
   - Fetch contact numbers
   - Call them, gather benefits, negotiate, get best prices
   - Create a consolidated comparison/USP diagram and report back
   - Optionally: call again to close the chosen policy and handle information collection

3. **Salesperson / Lead Pitch with Docs**
   - Sales agent uploads or provides pitch material (docs, FAQ, past call, website, etc.)
   - Agent calls prospect, explains product, answers questions using that material
   - When stuck, escalates/ transfers call to the human agent
   - Later: use same flow for outbound sales follow‑ups and upsell pitches

4. **Field Sales / School Visit Tracking**
   - For each sales rep, in the evening:
     - Get list of schools visited today
     - Get schools planned for tomorrow
     - Collect target vs achievement
   - Send follow-ups via WhatsApp, and if needed, calls
   - Have execution timelines/SLAs and follow-up logic (no open‑ended tasks)

5. **Follow‑up & Task SLA Management**
   - Every task must have an end date / execution timeline
   - Automatic follow-up when deadlines approach or responses are missing
   - Show “things due today / overdue” and prompt user for next steps (e.g., missed WhatsApp → suggest a call)

6. **Bakery / Grocery + Coordination (multi‑step agentic flow)**
   - Search nearby bakeries/grocers (e.g., “within 1 km”)
   - Get phone numbers
   - Call them to check availability (e.g., 1 kg pineapple cake / 1 kg rice)
   - Optionally coordinate driver/Porter/own driver:
     - Call driver with pickup/drop details
     - Or (future) integrate with Porter API and handle booking

7. **Multi‑channel Communication & Escalation**
   - Single instruction like “contact them” → agent chooses channel (WhatsApp / call / email) based on skills available
   - Conditional logic (eventually): WhatsApp first, if no response in X time, then call
   - For calls/WhatsApp, if the agent is confused or lacks info, it:
     - Puts user on hold (for calls)
     - Or sends a consent/clarification request (for WhatsApp)
     - Resumes with the user’s input

8. **Web Search + Skill Orchestration**
   - Use web search to:
     - Find entities (bakeries, insurance companies, vendors, etc.)
     - Then chain further actions: fetch numbers → call → gather info → summarize
   - Replace the hardcoded “insurance only” agent with a more generic, skill‑based flow

9. **WhatsApp UX Improvements for Demo**
   - Fallback behavior: if only one open task from a user, a plain reply should map to that task without “select to reply”
   - Show end‑to‑end: Trigo messages the owner for consent/details → owner replies → Trigo continues the same task with the contact.

   
10. **Call latency we must reduce it but we must explain Sarvam AI**