# Clean Code

## Summary

Clean code isn't about being clever. It's about conventions that make code predictable, so the next person to read it — or you, six months from now — doesn't have to think about anything except what the code actually does.

This doc starts with a few conventions worth adopting everywhere, and it will keep growing.

## Conventions

### `get` or `find`?

How many times have you called a method named `getUser` and had to go read its implementation to find out what happens when there's no user? Does it return null? Throw? Hand back an empty user? The name told you nothing.

So let's make the name tell you:

- **`get*`** returns **zero or one**: an optional. `getUser` gives you the user, or nothing.
- **`find*`** returns **zero or more**: a list or collection of some sort, empty when nothing matches. `findUsersByTeam` gives you every user on the team, or an empty list.

And the method's definition says so too, in its doc comment. The name tells you at a glance; the doc confirms it.

Notice what's missing: an error for "not found". Remember from [The Failure Path Is an Arrow](02-the-failure-path-is-an-arrow.md) — not finding something isn't an error, it's part of the method's contract. An empty optional and an empty list are perfectly good answers. Errors are for when something actually went wrong, like the database being unreachable.

A sketch in a few languages:

```go
import (
	"context"

	. "github.com/go-playground/pkg/v5/values/option"
)

type UserStore interface {
	// GetUser returns the user with the given ID, or None if there isn't one.
	GetUser(ctx context.Context, id UserID) (Option[User], error)

	// FindUsersByTeam returns every user on the team; the slice is empty if there are none.
	FindUsersByTeam(ctx context.Context, team TeamID) ([]User, error)
}

func greet(ctx context.Context, store UserStore, id UserID) (string, error) {
	user, err := store.GetUser(ctx, id)
	if err != nil {
		return "", err
	}
	if user.IsNone() {
		return "hello, stranger", nil
	}
	return "hello, " + user.Unwrap().Name, nil
}
```

```rust
pub trait UserStore {
    /// Returns the user with the given ID, or `None` if there isn't one.
    fn get_user(&self, id: UserId) -> Result<Option<User>, StoreError>;

    /// Returns every user on the team; empty if there are none.
    fn find_users_by_team(&self, team: TeamId) -> Result<Vec<User>, StoreError>;
}
```

```java
public interface UserStore {
    /** Returns the user with the given ID, or empty if there isn't one. */
    Optional<User> getUser(UserId id);

    /** Returns every user on the team; empty if there are none. */
    List<User> findUsersByTeam(TeamId team);
}
```

A quick word on Go. The traditional Go way is to return a pointer, and `nil` when there's nothing. It works, but it has two problems: nothing forces the caller to check for `nil` — hello, panic — and sometimes `nil` is itself a perfectly valid value. Now that Go has generics, there are plenty of Rust-like `Option` types that fix both. I use the one from [go-playground/pkg](https://github.com/go-playground/pkg/blob/master/values/option/option_common.go) (full disclosure: I wrote it :P), dot-imported so `Option`, `Some`, and `None` read as if they were part of the standard library.

One more thing: this is about *looking data up* — from a store, a service, a collection. A plain field accessor isn't a lookup and doesn't need to play by these rules.

### Name for the viewport

Short variable names get a bad rap. `u`, `r`, `n`... even single letters are fine — *if* the whole function fits on the screen. When you can see where a variable is declared and every place it's used, all at once, `u` is obviously the user. Call it about 40 lines. And loop counters? `i` is fine. It always has been.

But once a function scrolls past the screen, `u` stops meaning anything. You're scrolling back up to figure out what it was, and that's a wasted brain cycle, every single time. That's when names need to get specific.

And here's the thing: functions grow. So why not start with the most specific name for what's actually happening? Just... not too long! Nobody wants to read `counterOfTheBlahBlahBlah`. Be as specific as you can while staying short and sweet: `retries`, not `r`, and definitely not `numberOfRetryAttemptsSoFar`.

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

Good conventions are the ones you stop noticing. `get` or `find` tells you what comes back before you read a line of the implementation. Names are as short as their scope allows, and no shorter. Formatting and linting are decided once, by the defaults, and never again.

None of this is clever, and that's the point. Boring code is good code.

---

Previous: [The Failure Path Is an Arrow](02-the-failure-path-is-an-arrow.md)
