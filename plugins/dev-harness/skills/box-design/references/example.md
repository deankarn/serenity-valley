# Worked example — Prospecting API

One system drawn at every altitude. The system is deliberately trivial: an API that queries a contact database and shapes the response by caller tier. Assume auth, user profiles, and contact sourcing already exist.

Match this shape, not this content. Real breakdowns are sized to the problem.

## 50,000 ft

```
+-------------------------------------------------+
|               PROSPECTING API                   |
|                                                 |
|  WHAT:  Query by multiple criteria to find the  |
|         right stakeholders at a company (or     |
|         companies) to reach out to.             |
|                                                 |
|  WHY:   Unblocks and accelerates go-to-market   |
|         flows.                                  |
+-------------------------------------------------+
```

One box. No technology, no integrations.

## 10,000 ft

```
  +-----------------+                 +-----------------+
  |     CRM UI      |                 |   COMPANY MCP   |
  |   (CRM team)    |                 |    (AI team)    |
  +--------+--------+                 +--------+--------+
           |  search + save results            |  tool calls
           +---------------+-------------------+
                           v
               +-----------------------+
               |    PROSPECTING API    |   <-- us (new)
               +-----------+-----------+
                           v
               +-----------------------+
               |   CONTACT DATABASE    |
               +-----------------------+

  existing systems (auth, user profiles, contact sourcing)
  are assumed and not drawn
```

What this altitude surfaces: two conversations to have — the CRM team for saving results, the AI team for the MCP integration — and that both are blocked on the Prospecting API's edge contract, not on its implementation.

## Implementation

```
                    callers: CRM UI / MCP
                              |
  ======================== our service ========================
                              v
        +--------------------------------------------+
        |  API (exposure)                            |
        |    - route + auth middleware (exists)      |
        |    - input validation                      |
        |    in : ApiSearchRequest                   |
        |    out: ApiSearchResponse                  |
        +---------------------+----------------------+
                              |  contract
                              v
        +--------------------------------------------+
        |  BUSINESS LOGIC                            |
        |    - resolve caller tier -> record cap     |
        |    - wiring only, no storage, no transport |
        |    in : ProspectQuery                      |
        |    out: ProspectResult                     |
        +------+------------------------------+------+
               |                              |
               v                              v
  +--------------------------+   +------------------------------+
  |  AUTH / USER PROFILE     |   |  STORAGE                     |
  |    (exists, other team)  |   |    - all CRUD + search       |
  |    own tests, mock here  |   |    in : ContactFilter        |
  +--------------------------+   |    out: ContactRecord[]      |
                                 +--------------+---------------+
                                                v
                                     +----------------------+
                                     |  real / dockerized   |
                                     |  contact store       |
                                     +----------------------+
```

| Box | Owns | Tested against |
|---|---|---|
| API | transport, validation, auth wiring, status mapping | mocked business logic |
| Business logic | tier → record cap, wiring | mocked storage and auth |
| Storage | query construction, driver concerns | real or containerized store |
| Integration | nothing new | the whole product, from the outside |

Four payload types, one per box edge. On day one `ApiSearchRequest` and `ProspectQuery` look identical — they stay separate anyway.

**Build order.** Define the API contract first — CRM and AI teams mock against it and start in parallel. Build storage first (leaf, no dependencies), then business logic, then API. Integration with CRM UI and MCP last.

## Composability check

| Change | Lands in |
|---|---|
| Enterprise tier gets more results | business logic + its test |
| Add rate limiting | API only |
| Replace the database backend | storage only; no upstream test changes if the contract held |
| New enrichment product on the same data | reuse storage as-is |
| Add a gRPC entrypoint | new exposure box; reuse business logic as-is |

If a change in this table would touch more than the listed box, the decomposition is wrong.
