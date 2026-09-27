# Clean Code

## Summary

Clean code isn't about being clever. It's about conventions that make code predictable, so the next person to read it — or you, six months from now — doesn't have to think about anything except what the code actually does.

This doc collects conventions worth adopting everywhere: how to name things, when to give things their own types, how to shape functions, what comments are for, and why formatting should never cost you a thought. It will keep growing.

## Naming

### `get` or `find`?

How many times have you called a method named `getUser` and had to go read its implementation to find out what happens when there's no user? Does it return null? Throw? Hand back an empty user? The name told you nothing.

So let's make the name tell you:

- **`get`** returns **zero or one**: an optional. `users.Get(id)` gives you the user, or nothing.
- **`find`** returns **zero or more**: a list or collection of some sort, empty when nothing matches. `users.FindByTeam(team)` gives you every user on the team, or an empty list.

And the method's definition says so too, in its doc comment. The name tells you at a glance; the doc confirms it.

Notice what's missing: an error for "not found". Remember from [The Failure Path Is an Arrow](02-the-failure-path-is-an-arrow.md) — not finding something isn't an error, it's part of the method's contract. An empty optional and an empty list are perfectly good answers. Errors are for when something actually went wrong, like the database being unreachable.

A sketch in a few languages:

```go
package user

import (
	"context"

	. "github.com/go-playground/pkg/v5/values/option"
)

// Store reads and writes users.
type Store interface {
	// Get returns the user with the given ID, or None if there isn't one.
	Get(ctx context.Context, id ID) (Option[User], error)

	// FindByTeam returns every user on the team; the slice is empty if there are none.
	FindByTeam(ctx context.Context, team TeamID) ([]User, error)
}
```

```rust
pub trait Store {
    /// Returns the user with the given ID, or `None` if there isn't one.
    fn get(&self, id: Id) -> Result<Option<User>, StoreError>;

    /// Returns every user on the team; empty if there are none.
    fn find_by_team(&self, team: TeamId) -> Result<Vec<User>, StoreError>;
}
```

```java
public interface UserStore {
    /** Returns the user with the given ID, or empty if there isn't one. */
    Optional<User> get(UserId id);

    /** Returns every user on the team; empty if there are none. */
    List<User> findByTeam(TeamId team);
}
```

A quick word on Go. The traditional Go way is to return a pointer, and `nil` when there's nothing. It works, but it has two problems: nothing forces the caller to check for `nil` — hello, panic — and sometimes `nil` is itself a perfectly valid value. Now that Go has generics, there are plenty of Rust-like `Option` types that fix both. I use the one from [go-playground/pkg](https://github.com/go-playground/pkg/blob/master/values/option/option_common.go) (full disclosure: I wrote it :P), dot-imported so `Option`, `Some`, and `None` read as if they were part of the standard library.

One more thing: this is about *looking data up* — from a store, a service, a collection. A plain field accessor isn't a lookup and doesn't need to play by these rules.

### Let the container say it

Did you notice those methods are `Get` and `FindByTeam`, not `GetUser` and `FindUsersByTeam`? That's on purpose.

Code always lives in a container — a package, a crate or module, a class. The container already names the thing. In Go, the caller sees `user.Store`, so `GetUser` would say "user" twice. In Java, the caller sees `UserStore`, so `getUser` repeats it. Drop what the container already says. Go even has a name for this — stutter — and its official guidance is to avoid it.

The same goes for the variable holding the container: name it after the container, following the same conventions. A store full of users is `users`. Not `userStore`, not `userDAO` — `users`. Then the call site reads like a sentence: `users.Get(id)`. And because the store is plural, `user` is still free for the user you get back.

```go
func greet(ctx context.Context, users user.Store, id user.ID) (string, error) {
	found, err := users.Get(ctx, id)
	if err != nil {
		return "", err
	}
	if found.IsNone() {
		return "hello, stranger", nil
	}
	user := found.Unwrap()
	return "hello, " + user.Name, nil
}
```

```rust
fn greet(users: &impl user::Store, id: user::Id) -> Result<String, user::StoreError> {
    let Some(user) = users.get(id)? else {
        return Ok("hello, stranger".to_string());
    };
    Ok(format!("hello, {}", user.name))
}
```

```java
static String greet(UserStore users, UserId id) {
    return users.get(id)
            .map(user -> "hello, " + user.name())
            .orElse("hello, stranger");
}
```

### Name arguments, then match them

Arguments deserve good names too — they're the first thing anyone reads when calling your function. And here's the part people miss: when you call a function, name your variables the same as its parameters. `Get` takes an `id`, so pass it an `id`, not an `x`, a `uid`, and definitely not a `theIdOfTheUser`. Follow a value through a call chain and it keeps the same name the whole way.

The exception is truly generic code. `slices.Contains(s, v)` names its parameters `s` and `v` because it has no idea what you're passing it — so there, your names win.

```go
id := request.UserID
found, err := users.Get(ctx, id)

isAdmin := slices.Contains(roles, Admin)
```

```rust
let id = request.user_id;
let found = users.get(id)?;

let is_admin = roles.contains(&Role::Admin);
```

```java
UserId id = request.userId();
Optional<User> found = users.get(id);

boolean isAdmin = roles.contains(Role.ADMIN);
```

### Booleans are questions

A boolean answers a yes/no question, so name it like one: `isActive`, `hasAccess`, `shouldRetry`. Read it out loud in an `if` and it just works — "if user is active". `active` could be a boolean, a timestamp, or a list of active things. `isActive` can only be one.

```go
type User struct {
	Name     string
	IsActive bool
}

// HasAccess reports whether the user may see the resource.
func (u User) HasAccess(resource Resource) bool {
	return u.IsActive && resource.IsPublic
}
```

```rust
pub struct User {
    pub name: String,
    pub is_active: bool,
}

impl User {
    /// Whether the user may see the resource.
    pub fn has_access(&self, resource: &Resource) -> bool {
        self.is_active && resource.is_public
    }
}
```

```java
public record User(String name, boolean isActive) {
    /** Whether the user may see the resource. */
    public boolean hasAccess(Resource resource) {
        return isActive && resource.isPublic();
    }
}
```

### Name for the viewport

Short variable names get a bad rap. `u`, `r`, `n`... even single letters are fine — *if* the whole function fits on the screen. When you can see where a variable is declared and every place it's used, all at once, `u` is obviously the user. Call it about 40 lines. And loop counters? `i` is fine. It always has been.

But once a function scrolls past the screen, `u` stops meaning anything. You're scrolling back up to figure out what it was, and that's a wasted brain cycle, every single time. That's when names need to get specific.

And here's the thing: functions grow. So why not start with the most specific name for what's actually happening? Just... not too long! Nobody wants to read `counterOfTheBlahBlahBlah`. Be as specific as you can while staying short and sweet: `retries`, not `r`, and definitely not `numberOfRetryAttemptsSoFar`.

### If it can crash, say so

Remember from [The Failure Path Is an Arrow](02-the-failure-path-is-an-arrow.md): some failures should stop the application, right there. A function that does that had better say so, because nobody expects a harmless-looking call to take the whole process down.

Each language has its own way of saying it. Go prefixes the name with `Must`. Rust uses `expect` and documents it under a `# Panics` heading. Java's convention is `require`. Either way, these belong at startup — wiring, configuration, compiling a pattern — where failing means the program itself is broken. Never on a request path.

```go
var slugPattern = regexp.MustCompile(`^[a-z0-9-]+$`)

// MustLoadConfig parses the embedded configuration and panics if it's invalid:
// it ships with the binary, so an invalid one is a bug.
func MustLoadConfig() Config {
	config, err := parseConfig(embeddedConfig)
	if err != nil {
		panic(fmt.Sprintf("embedded config is invalid: %v", err))
	}
	return config
}
```

```rust
/// Parses the embedded configuration.
///
/// # Panics
///
/// Panics if the configuration is invalid: it ships with the binary, so an invalid one is a bug.
pub fn load_config() -> Config {
    parse_config(EMBEDDED_CONFIG).expect("embedded config is valid")
}
```

```java
public final class SqlUserStore implements UserStore {
    private final DataSource dataSource;

    public SqlUserStore(DataSource dataSource) {
        this.dataSource = Objects.requireNonNull(dataSource, "dataSource");
    }
}
```

## Types

### Give concepts their own types, where it pays

Here's a bug that compiles just fine. A method takes a user ID and an order ID, and both are plain integers. Swap them at the call site and... it compiles. It runs. It cancels the wrong order — maybe someone else's.

The fix is to give each concept its own type. A `UserID` is not an `OrderID`, even if both are integers underneath. Now the swap is a compile error, and in Go and Rust it costs nothing at runtime: the wrapper compiles away.

I'll be honest, though: this one isn't free, especially in Java. Every new type is more code, and in some languages more glue for every framework that touches it. So spend it where the mix-ups actually happen:

- **Values of the same primitive type that meet** — two or more different concepts in one signature, like a user ID and an order ID. That's where swaps happen.
- **Values with rules** — money, or an `Email` that validates itself once, in its constructor, so nothing downstream ever has to check again.
- **Units** — more on those in the next section.

In Go and Rust it's so cheap that it's worth defaulting to. Elsewhere, weigh it: a `UserID` that only ever travels on its own doesn't need a type of its own. And never retrofit a whole codebase just for this.

```go
type UserID int64
type OrderID int64

type Orders interface {
	// Cancel cancels the user's order.
	Cancel(ctx context.Context, userID UserID, orderID OrderID) error
}

func cancel(ctx context.Context, orders Orders, userID UserID, orderID OrderID) error {
	// Passing orderID first is a compile error: cannot use orderID (OrderID) as UserID.
	return orders.Cancel(ctx, userID, orderID)
}
```

```rust
use std::ops::Deref;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(i64);

impl From<i64> for UserId {
    fn from(raw: i64) -> Self {
        Self(raw)
    }
}

impl Deref for UserId {
    type Target = i64;

    fn deref(&self) -> &i64 {
        &self.0
    }
}

pub trait Orders {
    /// Cancels the user's order.
    fn cancel(&self, user_id: UserId, order_id: OrderId) -> Result<(), OrderError>;
}
```

```java
public record UserId(long value) {}

public record OrderId(long value) {}

public interface Orders {
    /** Cancels the user's order. */
    void cancel(UserId userId, OrderId orderId);
}
```

A few things worth knowing, per language:

- **Go:** that's a *defined type*, `type UserID int64`. Watch out for the `=`: `type UserID = int64` is an alias — just another name for `int64` — and protects nothing. Also, untyped constants still convert on their own, so `orders.Cancel(ctx, 42, 7)` compiles. The protection is between typed values, which is exactly where mix-ups happen. Bonus: `database/sql` accepts a `UserID` as a query argument and scans into one directly, because it looks at the underlying `int64`.
- **Rust:** a newtype. `From` gets a raw value in, and `Deref` gets it back out with `*user_id` when you hit the database. Deref also lets you call any `i64` method on the ID — harmless, but nothing stops you doing arithmetic on it. And often you don't need to deref at all: serde's `#[serde(transparent)]` and sqlx's `#[sqlx(transparent)]` take the newtype as-is. (`OrderId` gets the same `From` and `Deref`.)
- **Java:** a record. It isn't free today — each one is a small object — but the JIT often optimizes it away, and value classes are coming to make it zero-cost. There's no `Deref` in Java either: `.value()` is your way out, and each framework — Jackson, JPA, jOOQ — has to be taught the type once. That's the real cost, and why Java is where to be most selective.

Not one of those three? Most languages have a way to do this now: Kotlin value classes, TypeScript branded types, Python's `NewType` (checked by mypy or pyright). Use whatever yours has.

**Convert at the edges.** Raw integers and strings become `UserID`s once, where they enter the system — the API box when it parses a request, the storage box when it reads a row — and go back to raw once, when the storage box writes them. Everywhere in between, values travel typed. A conversion in the middle of your business logic is a red flag, and an easy one to grep for.

**Where do they live?** These types are shared vocabulary, not payloads — every box talks about users — so they don't break the "every box owns its payloads" rule from [Everything's a Box in a Box](01-everything-is-a-box-in-a-box.md). But *where* they live depends on one question: who defines what the value means?

- **One box defines it.** A user ID is minted by whatever owns users. So `UserID` lives with that box's contract — `user.ID`, right next to `user.Store`. Anyone using it already imports that package to call `users.Get`, so it costs no extra dependency.
- **Nobody owns it, and everyone agrees what it means.** An email address is an email address everywhere. Same for money, a phone number, a URL. Those go in the same shared foundation library as `Retryable` from [The Failure Path Is an Arrow](02-the-failure-path-is-an-arrow.md): one per language, for the whole codebase or company.
- **Better yet, don't write it at all.** `time.Duration`, `java.time`, a well-known money library... a standard type costs you nothing.

Either way, the type itself stays dependency-free: no JSON annotations, no ORM annotations, nothing that drags a framework into every box that touches a `UserID`. The framework glue — a JPA converter, a Jackson mixin, a custom scanner — lives in the box that uses the framework: the storage box for the database, the API box for JSON. Right back to convert at the edges.

**Don't wrap everything, though.** Not every string needs its own type, and over-typing is the same trap as being too DRY.

### Units belong in types

`timeoutMs`, `delaySeconds`, `sizeKb`... every one of those is a unit smuggled into a name, where the compiler can't check it. Pass `5` where milliseconds were expected and nothing stops you. Put the unit in the type instead, and the type *is* the documentation.

```go
type Config struct {
	Timeout time.Duration
}

config := Config{Timeout: 5 * time.Second}
```

```rust
pub struct Config {
    pub timeout: Duration,
}

let config = Config { timeout: Duration::from_secs(5) };
```

```java
public record Config(Duration timeout) {}

Config config = new Config(Duration.ofSeconds(5));
```

## Functions

### Order arguments by variance

Here's one most people never think about. When you write a function's parameter list, order the arguments from the ones that vary least to the ones that vary most.

- **First, the ones that are always there** — Go's `context.Context`, for example. Every call passes one, and it's always called `ctx`.
- **Then the ones that barely change** — constants, configuration, an enum with a handful of values.
- **Last, the ones that change every time** — the actual payload.

Why? Because it keeps you, and your coding agent, on autopilot. You type the boring arguments without thinking, and your attention arrives exactly where it's needed: the argument that's different this time. And the call sites line up, so the thing that actually differs between them jumps out.

```go
// Send delivers the message at the given priority.
Send(ctx context.Context, priority Priority, msg Message) error

notifier.Send(ctx, Urgent, welcome)
notifier.Send(ctx, Urgent, passwordReset)
notifier.Send(ctx, Normal, weeklyDigest)
```

```rust
/// Delivers the message at the given priority.
fn send(&self, priority: Priority, msg: &Message) -> Result<(), SendError>;

notifier.send(Priority::Urgent, &welcome)?;
notifier.send(Priority::Urgent, &password_reset)?;
notifier.send(Priority::Normal, &weekly_digest)?;
```

```java
/** Delivers the message at the given priority. */
void send(Priority priority, Message msg);

notifier.send(Priority.URGENT, welcome);
notifier.send(Priority.URGENT, passwordReset);
notifier.send(Priority.NORMAL, weeklyDigest);
```

Where the language decides a position for you, the language wins: the receiver or `self` comes first, variadic arguments come last, and so do trailing closures in languages that have them.

### No flag arguments

`notifier.Send(ctx, msg, true)` — true *what*? Urgent? Silent? Retry? You have to go read the signature to find out, every single time. A boolean argument tells the reader nothing at the call site.

Use an enum instead. `Urgent` says what it means, it can grow a third value later without breaking every caller, and — as a value with only a handful of possibilities — it goes before the payload, by the variance rule above.

```go
type Priority int

const (
	Normal Priority = iota
	Urgent
)
```

```rust
pub enum Priority {
    Normal,
    Urgent,
}
```

```java
public enum Priority {
    NORMAL,
    URGENT,
}
```

### Return early

Handle the edge cases first and get out. Guard clauses at the top, one per condition, each one returning right away. What's left is the happy path, and it stays flat against the left margin instead of buried three levels deep in nested `if`s.

```go
func discount(customer Customer, total Money) Money {
	if !customer.IsActive {
		return 0
	}
	if total < minimumForDiscount {
		return 0
	}
	return total / 10
}
```

```rust
fn discount(customer: &Customer, total: Money) -> Money {
    if !customer.is_active {
        return 0;
    }
    if total < MINIMUM_FOR_DISCOUNT {
        return 0;
    }
    total / 10
}
```

```java
static long discount(Customer customer, long total) {
    if (!customer.isActive()) {
        return 0;
    }
    if (total < MINIMUM_FOR_DISCOUNT) {
        return 0;
    }
    return total / 10;
}
```

## Comments

### Say why, not what

`// increment the counter` above `counter++` tells me nothing the code didn't. The code already says *what* it does. A comment is for what the code can't say: *why*. Why this odd-looking order, why this magic number, why this workaround for that vendor's bug.

Doc comments are the other kind worth writing: they state the contract. Remember `get` and `find`? The doc comment is where "or None if there isn't one" lives.

And commented-out code? Delete it. Git has history. Nobody is ever going to uncomment it, and everyone who reads past it has to wonder whether they should.

## Formatting and linting

### Formatting: pick the default and never think about it again

This is the big one.

Pick a code formatter. If your language has one — `gofmt`, `rustfmt` — use it, with its defaults. And then NEVER THINK ABOUT IT AGAIN!

I mean it. Nobody should ever waste a single brain cycle on tabs vs spaces, brace placement, or line length. Not in code review, not in a team meeting, not in a pull request comment. Trust me, you'll get used to the default formatting. IT REALLY DOESN'T MATTER.

No official formatter for your language? Pick the most popular one, keep its defaults, and... you guessed it. Never think about it again.

### Linting is not formatting

Some people conflate formatting with linting. They're not the same thing!

Formatting is how the code looks. Linting is whether the code does something suspicious — an unchecked error, an unused variable, a bug waiting to happen. Picking lint rules can sometimes make sense, and it can be specific to your project.

However, if your language has a de facto linter that most of the community uses — `clippy`, `golangci-lint`, `eslint` — just use it. And just like formatting: NEVER THINK ABOUT IT AGAIN!

As an added bonus, your code now looks like everyone else's code, formatting-wise. New hires can read it on day one, and you can read everyone else's. Being unique for unique's sake has zero value. Remember: boring code is good code.

### Automate it

The best way to never think about formatting is to never do it by hand. Format on save in your editor, a pre-commit hook, a CI check that fails on unformatted code, a hook in your coding agent — pick whichever fits. That's a decision for your team or company to make, not something a tool should force on you. But make it once, and then... well, you know.

## Final thoughts

Good conventions are the ones you stop noticing.

- **Names** tell you what comes back before you read a line of the implementation, don't repeat what their container already says, and are as short as their scope allows, and no shorter.
- **Types** make the wrong call fail to compile where mix-ups actually happen: a `UserID` is never an `OrderID`, and a timeout is never just an `int`.
- **Functions** put the boring arguments first, say what they mean instead of passing `true`, and get the edge cases out of the way early.
- **Comments** explain why.
- **Formatting and linting** are decided once, by the defaults, and never again.

None of this is clever, and that's the point. Boring code is good code.

---

Previous: [The Failure Path Is an Arrow](02-the-failure-path-is-an-arrow.md)
