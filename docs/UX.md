# UX Requirements

Usability requirements for the DataChat AI chat interface. Desktop only.

---

## Results Display

Every query response shows three components together:

1. **Text summary** — LLM-generated narrative highlighting key insights
2. **Data table** — paginated preview of raw results
3. **Chart** — auto-selected chart type based on data shape

### Chart Interaction
- System selects the default chart type (line, bar, pie, etc.) based on data
- User can override via:
  - Natural language ("show this as a bar chart")
  - UI toggle in the chart panel

### Large Result Sets
- Chat shows a paginated preview (e.g. first 50 rows)
- Download full result as CSV always available

### Export
- Export result (chart + table + summary) as PDF or image

---

## Error Handling

- When a question is out of domain or data is unavailable: show "I can't answer that"
- No explanation, no suggested alternatives

---

## Conversation

### History
- Full conversation history visible in sidebar
- User can click any past conversation and continue from where they left off

### Follow-ups
- Users can ask follow-up questions inline on any result in the same conversation
- System resolves co-references and context automatically (see ARCHITECTURE.md — Conversation Resolver)

### New Conversation
- User can start a new conversation at any time

---

## Hints

- Per-tenant example questions shown in the chat input area
- Helps users understand what they can ask without guessing
- Admin configures hints in the admin panel

---

## Loading & Abort

- Step-by-step progress indicator during query execution:
  - e.g. "Checking domain..." → "Resolving query..." → "Running query..." → "Generating response..."
- User can abort the request at any point during processing

---

## Flagging & Feedback

- Every response has a thumbs down (flag) button
- On flag: optional text field to type the correct answer or describe the issue
- Flagged responses are logged for admin review and model improvement

---

## Saved Queries

- Users can pin/save frequently used queries
- Saved queries appear as quick-launch shortcuts in the UI
- No scheduling — saved queries are manually triggered only

---

## Multi Data Source

- Users do not select which data source to query
- System auto-routes using the Knowledge Graph and admin-provided descriptions
- Transparent to the user

---

## Platform

- Desktop browser only
- No mobile or tablet support

---

## Out of Scope

- Sharing conversations or results with colleagues
- Scheduled / recurring queries
- Mobile / tablet UI
