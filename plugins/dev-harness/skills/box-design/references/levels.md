# Altitudes

The same system is one box or twenty depending on who is looking. Pick the altitude by audience, and do not mix them in one document — detail from a lower altitude derails the conversation the higher one is for.

## 50,000 ft — leadership, stakeholders, elevator pitch

One box. What it is, and why it is worth building.

- **What**: the capability in one or two sentences, in the caller's language.
- **Why**: the problem it removes or the flow it unblocks.

No integrations, no components, no technology. If a technology choice appears here, it is either the actual product or it does not belong.

## 10,000 ft — engineering managers, product, tech leads

The system plus the systems it touches. Enough to plan, staff, and sequence across teams.

- Each box is a system or a team-owned component, not a module.
- Each arrow is an integration, labelled with what flows across it.
- Mark which boxes exist already and which are new.
- Mark the owning team per box.
- Assumed existing infrastructure is stated once and not drawn.

This is the altitude where cross-team dependencies become visible: the conversations that need to happen, and which team is blocked on which contract. Implementation detail here inflates the apparent scope, because each team only ever takes a slice of it.

## Implementation — the engineers building it

One team's slice, decomposed into boxes with contracts. Covered by the main skill body: responsibility per box, payload types per box, arrows as contracts, build order, test strategy per box.

Other teams have their own implementation diagrams. Do not draw theirs; draw the contract you hand them.

## Choosing

If asked to present, pitch, or document a design, ask who the audience is before picking an altitude, unless it is already obvious from the request. When a document has to serve more than one audience, separate the altitudes into distinct sections rather than blending them.
