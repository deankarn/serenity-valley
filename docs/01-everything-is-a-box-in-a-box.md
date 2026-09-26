# Everything’s a Box in a Box  
  
#engineering #wisdom  
  
  
### Summary  
  
This will demonstrate that everything including concepts, products, features, applications, pipelines, databases can all be though of as a box and outline how depending on the level or position of persons involved what level of depth or fidelity of the boxes your looking/working at applies.  
  
It then will further describe how boxes at the implementation level relate to good development practices and coding practices.  
  
  
  
### Talk  
  
So everything’s a Box in a Box! no really! let’s start from the top at the 50,000 foot view.  The 50,000 foot view is the level at which you discuss products with leadership, give elevator pitches from and convince stakeholders building whatever it is worth it without getting dragged down into the details of how.  
  
As an example, let’s assume for the entirety of the talk a contrived example that we will assume is new, novel, groundbreaking and change the industry :P building an API that hits a database, based on contents of row return different response… groundbreaking right! well no but it’s easy to follow for the purposes of this talk, but expands far beyond. We will also assume your business already has user profiles, auth and contact sourcing and storage with existing systems for that already.  
  
So with that, at the 50,000 foot view, this is just one box this box describes the system, what is this thing, what does it do and what problem is it solving? In this case we will describe it as a prospecting API which lets the caller query by multiple criteria to find stakeholders at a company or companies to communicate with and will be a game changer for go-to-marketing flows.  
  
```
                +-------------------------------------------------+
                |                                                 |
                |               PROSPECTING API                   |
                |                                                 |
                |  WHAT:  Query by multiple criteria to find the  |
                |         right stakeholders at a company (or     |
                |         companies) to reach out to.             |
                |                                                 |
                |  WHY:   Unblocks and accelerates go-to-market   |
                |         flows.                                  |
                |                                                 |
                +-------------------------------------------------+
```
  
As you can see it’s short, sweet and to the point of the problem it’s solving, this is how you sell or pitch an idea.  
  
Ok now the next level is the 10,000 foot view, this is the view by which more detail of how the product or feature would be structured and connect /integrate with other systems.  What about the boxes you say? I was promised boxes!!! give! alright alright in this case, we just zoom into the 50,000 foot box and lo and behold it becomes multiple boxes! what boxes  
  
```
          +-----------------+                 +-----------------+
          |     CRM UI      |                 |   COMPANY MCP   |
          |   (CRM team)    |                 |    (AI team)    |
          +--------+--------+                 +--------+--------+
                   |                                   |
                   |  search + save results            |  tool calls
                   |                                   |
                   +---------------+-------------------+
                                   |
                                   v
                       +-----------------------+
                       |    PROSPECTING API    |   <-- us
                       +-----------+-----------+
                                   |
                                   v
                       +-----------------------+
                       |   CONTACT DATABASE    |
                       +-----------------------+

          existing systems (auth, user profiles, contact sourcing)
          are assumed and not drawn
```
  
We have the API, database, CRM UI that calls the API and MCP that leverages the API to hook the functionality into agents. Whoah, 4 boxes now! You asked, I delivered!  So what’s the point of the 10,000 foot view? it for a high level understanding of the new system beyond the what and why diving into the how. Given the 4 boxes you can see oh, to make this work we’ll need to talk to the CRM team to add functionality to add results from prospecting to the existing CRM product. And we’ll also need to talk to the AI team about building a new or integrating into and existing company MCP.  
  
So the 10,000 foot view is for engineering managers, product managers, and engineering leads in order to facilitate conversations and make product plans and coordinate the building of this new product with multiple teams. The low-level how isn’t needed to facilitate these conversations, negotiate timelines and manpower needed to get things on the roadmap. Too low level details can cause such conversations to stall or make the requirements seem larger than they are in actuality, because each team will be taking only a portion of those details, so no need to know about all things especially those that another team owns and is fully responsible for.  
  
  
I how your starting to see the pattern, now lets dive into the implementation layer, whoah! many more boxes, and actually mutiple different teams have implementation boxes, for example MCP team has theirs, CRM has theirs, for brevity we will focus on the owning team for the new system whereby the functionality.  
  
```
                        callers: CRM UI / MCP
                                  |
  ============================ our service ============================
                                  |
                                  v
            +--------------------------------------------+
            |  API BOX                                   |
            |    - route + auth middleware (exists)      |
            |    - input validation                      |
            |    in : ApiSearchRequest                   |
            |    out: ApiSearchResponse                  |
            +---------------------+----------------------+
                                  |  interface contract
                                  v
            +--------------------------------------------+
            |  BUSINESS LOGIC BOX                        |
            |    - resolve caller tier -> record cap     |
            |    - wiring only, no storage, no transport |
            |    in : ProspectQuery                      |
            |    out: ProspectResult                     |
            +------+------------------------------+------+
                   |                              |
                   v                              v
      +--------------------------+   +------------------------------+
      |  AUTH / USER PROFILE     |   |  DATABASE BOX                |
      |    (exists, other team)  |   |    - all CRUD + search logic |
      |    own tests, mock here  |   |    in : ContactFilter        |
      +--------------------------+   |    out: ContactRecord[]      |
                                     +--------------+---------------+
                                                    |
                                                    v
                                         +----------------------+
                                         |  real / dockerized   |
                                         |  contact store       |
                                         +----------------------+

  every box owns its payloads + its own tests; each arrow is a contract
```
  
How many boxes this time? let’s see, one box for the database interaction layer, let’s assume it didn’t exist before for searching for contacts. This box contains all database CRUD logic and defined input & output payloads for each and dedicated tests against, preferably a real, dockerized or test database of some sort. This is the database interface contract.  
  
Next the API box which contains details of the API Input & Output payloads, input validation and auth hooking, normally as middleware as previously mentioned which already exists. This box defines the interface contract into the prospecting product itself and is maybe the most important to define because it’s at the edge of the product that other teams will interact with.  
  
Then theres a business logic box. I should explain what business logic is before diving in; it’s really just wiring of multiple boxes together to get the desired behaviour. It is it’s own box for a reason we’ll explain later but in our example let’s keep it simple, it defines its own input and output payloads which takes required search & auth information, determines which tier the caller uses and adjust the number of records they’re allowed to receive based on that for prospecting, calls the database box and returns the response output.  Now first thing before we dive deeper to note, it may be tempting to use the same API input payload and the input business logic payload, or the output from the DB to be the output of the business logic, maybe even the API too, but resist this urge! each should have their own! It seems like you can or should, especially when the product is new as the inputs and outputs look almost the same if not identical BUT they will diverge I promise you and if you don’t these pieces will not be able to evolve independently! and in the long run hurt the composibility of your system.  
  
So boxes! many boxes at this level, what are they used for. Well this is not only for building of the system, but after defining the interface contracts, now the other teams are unblocked that you need help from to build those pieces in parallel and can mock data while building their pieces, integration being the very last piece into your API.  
  
Ok let’s dive one aspect deeper, these boxes now naturally map to pieces or steps to building your project, highlights what pieces need to exist first before other pieces can be built guiding the order of what needs to be built, which in tern then guides which can be built in parallel vs blocked by previous tasks which then guides milestones for you project based on what’s is blocked by what needs to exist first. No matter the size of the project this rings true, think of this as setting up the dominos, seems tedious BUT remember what happens after setting up the dominos…you get to knock them down and that part is FAST! because it’s pure execution at that point!  Ok so a bunch of good reasons, but there’s one other concept that will make your code, systems and products composable in a way that keep you flexible, saves time and allows you to easily change or expand you products and boils down to composability.  We touched a little bit on it with the DB and tests so let’s explain it upwards. So the DB is the lowest level point, one at which we can most of the time test against the real thing in one form or another so you build this piece and tests that cover all the functionality.  
  
Now we get to the business logic we need to hit the auth/user profile info to know about tier and the database, and we need tests for it also, but here’s the thing I don’t need to test the DB nor auth services code, I can mock those in the tests, all I need to do is test the business logic is wired correctly and points hit. I see this mistake made all the time! but seriously you don’t need to test the other parts, they have their own dedicated tests already proving they work and so now what you have to test with the business logic is the wiring!  Ok then let’s go to the API it contains the auth middleware and  business logic, again the business logic has it’s own dedicated test, just like the auth service should and so again can be mocked away in the API tests only needing to test the API now.  
  
I hope in this very simple product you can see how we will save time writing tests not having to re-test what’s already tested but more importantly your code and tests are decoupled and can evolve independently unless the interface contract changes. If any of you are thinking but how do you know the product works end-to-end then, well that’s what integration tests are for vs the unit tests we’ve been talking about; there’s something new to learn also, that’s where the line between unit and integration tests and that integration tests much like business logic, is testing the wiring between multiple things, but from the product level/perspective.  
  
Ok and now finally the best part! Composability, ensuring your boxes are decoupled and no abstract leakage allows for easier refactoring, additions, removal and changing of existing components don’t believe me, here are a few examples:  example 1. leadership decides we want to let enterprise tier get more results than in did before, what do we do? Adjust that in the business logic and associated test…that’s it! recompile and deploy. Didn’t need to worry about the API nor the DB in any way.  example 2. We need to add rate limiting to the API, some people are abusing our liberal use policy. So we add rate limiting to the API and adjust the tests, recompile and deploy…never needed to even think about the business logic or db.  
  
example 3. we need to completely change the DB backend we use, our old one can’t handle the load and need something better so we do it, replace the entire db backend, adjust the DB code and so long as we didn’t break the interface contracts didn’t even need to think about the API nor business logic again, not ever adjust the tests.  
  
In example 3 we’re also touching on another composability benefit, reusability! here are some cool examples:  example 1. We decide we want to build an enrichment product on top of the contact data as well and need to pull data from the same DB. Sweet we can use the same DB box we already have! it’s decoupled from the prospecting business logic and entrypoint we can just pick that DB module up and use it if we’ve properly segregated it based on the different languages concepts of compilation composition.  
  
example 2. We decide we want to offer integration through API or more higher-performance GRPC, what do we do. Build the GRPC and pick the business logic up off the shelf for use in it.   
I hope with this you can see why building things in this manner solves not only the current requirements but also a lots of future unseen ones as well and honestly doesn’t slow you down and usually only takes a few extra minutes of planning the boxes you need.  
  
Now another warning, this may sounds like everything should be self-contained it’s it’s own structure and own tests, but there is such a thing as too DRY, leftpad I’m looking at you! The separate comes at separation of responsibility like storage/DB layer vs business logic vs exposing layer like the API.  
  
### Final Thoughts  
  
Which view you discuss depends highly on your audience. Leadership doesn’t want to hear about the low level code and the how, they are more concerned with the what and why which is the 50,000 foot view.  EL Managers, team leaders and product needs to know more, they need to also know the how to be able to discuss with other teams for integration, in this case the CRM team to add functionality on that side and AI team which manages your company’s MCP’s. So 10,000 foot view is everything in the 50,000 foot view plus a little bit of the general how mainly for high-level understanding of the system.  
  
The implementation level view is for engineers that will build the system, they need to know mare than just the high level details, they need to know and think about the different building blocks and interface contracts between boxes and edges between teams and technologies.  
  
Remember great software is built using multiple boxes like lego! and business logic is the wiring of multiple lower level boxes together.  
