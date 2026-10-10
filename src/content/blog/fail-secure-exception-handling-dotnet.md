---
title: "The Exception Table That Fails Open"
description: "A crash-handling middleware that looks rigorous but silently routes every unlisted error to a 500 — and why the default branch is the whole security posture."
pubDate: 2026-10-10
tags: [security, dotnet, error-handling, architecture]
draft: false
---

I reviewed an exception middleware that looked rigorous. Six typed cases, stable error codes, `ProblemDetails` responses.

It was still returning 500s on six different malformed inputs to one endpoint. The cause was two lines of ordinary code a service elsewhere was throwing.

## The middleware

```csharp
// ErrorHandlingMiddleware.cs
var (status, errorCode, detail) = ex switch
{
    PaymentStateValidationException =>
        (StatusCodes.Status400BadRequest, "PAYMENT_STATE_VALIDATION_ERROR", ex.Message),
    PaymentProofReusedException =>
        (StatusCodes.Status409Conflict, "PAYMENT_PROOF_REUSED", ex.Message),
    ArgumentException =>
        (StatusCodes.Status400BadRequest, "VALIDATION_ERROR", "The request contains invalid data."),
    InvalidOperationException when ex.Message.Contains("already been resolved")
        => (StatusCodes.Status409Conflict, "OFFER_ALREADY_RESOLVED", ex.Message),
    UnauthorizedAccessException =>
        (StatusCodes.Status401Unauthorized, "UNAUTHORIZED", "Authentication required."),
    _ => (StatusCodes.Status500InternalServerError, "INTERNAL_ERROR", "An unexpected error occurred."),
};
```

## The two lines that broke it

```csharp
// PaymentService.cs
throw new InvalidOperationException(
    "Payment proof can only be uploaded for cases in Payment_Pending state.");
```

Walk the match:

- `InvalidOperationException` **is** in the switch
- But that arm has a `when` guard, and the guard looks for different words
- No match → falls to `_` → **500**

A state conflict that should have been a 409. Nothing exotic — just a new error message nobody added to a list.

## The default branch is the security posture

Every named arm is a decision someone made deliberately. The `_` arm is what happens to everything nobody thought about.

- A switch this thorough looks complete **during review**
- The eye goes to the six tidy named cases and concludes the file is covered
- The list being long says **nothing** about the size of the complement

> **The table grows by accretion, but the exposure grows by everything absent from it.**

A reviewer reading that switch is reading the list of *known* errors. Calling the design robust because the list is long is the mistake.

### Make the unlisted case visibly wrong

Same client behaviour, completely different behaviour for the next developer:

```csharp
var (status, errorCode, detail) = ex switch
{
    // Known, deliberately-handled domain failures.
    PaymentStateValidationException => (400, "PAYMENT_STATE_VALIDATION_ERROR", ex.Message),
    PaymentProofReusedException       => (409, "PAYMENT_PROOF_REUSED", ex.Message),

    // A bare InvalidOperationException almost always means someone threw a
    // message string instead of a typed exception. Treat it as a bug in THIS
    // code, and make that visible instead of returning a plausible 500.
    InvalidOperationException =>
        (500, "UNMAPPED_OPERATION_FAILURE", "An unexpected error occurred."),

    _ => (500, "INTERNAL_ERROR", "An unexpected error occurred."),
};
```

The `Payment_Pending` throw now lands in an arm that says, in the source: *"this was never mapped"*. That is a review question, and the right one.

## Why message matching is the real smell

`ex.Message.Contains("already been resolved")` makes correctness depend on prose — and prose is the part of a codebase nobody versions with any discipline.

| Breakage | Result |
|---|---|
| Reworded to "has already been resolved" | 409 silently becomes a 500 |
| `Contains` → `StartsWith` for tidiness | A legitimate case stops matching |
| Message is localised | The guard breaks for non-English callers |
| Two exceptions share phrasing | You can't tell which one fired |

Exceptions already carry identity in their **type**:

- `throw new OfferAlreadyResolvedException(...)` — a compile-time fact
- `Contains("already been resolved")` — a runtime guess

Prefer the first. If you're stuck with the second, at least fail loudly when it misses.

## Fail-safe vs fail-secure

These get collapsed constantly, and the collapse is itself a bug — they point at **opposite** answers to "what should happen when it breaks?"

- **Fail-safe** — failure moves to the safe *physical* state. A fail-safe lock **unlocks** when power is cut. Loss of power means emergency egress; nobody gets trapped inside.
- **Fail-secure** — failure moves to the safe *security* state. A fail-secure lock **locks** when power is cut. Loss of power may be an attack; nothing gets in.

Neither is universally correct — which is exactly why you can't let the default decide:

- Wrong toward physical → someone is trapped
- Wrong toward security → an attacker walks in during a power cut

> **A component whose behaviour under failure nobody decided will default to whichever way its control flow falls. And that is decided by nobody — which means it is decided by accident.**

## The inverted condition

From the course material:

```csharp
bool permissionGranted = authorizationProcess(...);

if (permissionGranted == true) {
    // Authorization process failed.
    ...
}
else {
    // Authorization process passed.
    ...
}
```

The comments say one thing. The branches do the other. Every attacker who finds this gets in on the failure path.

### The real-world version never looks this silly

```csharp
if (await _store.CanMutateAsync(caseId, ct))
{
    await _store.MutateAsync(caseId, ct);   // allowed
}
```

Nobody writes comments that contradict the branch. But consider:

- `CanMutateAsync` throws, or times out
- Or the store returns something the compiler can't type-check
- Entity Framework returns `0` instead of throwing on a failed query — so **"found nothing" and "failed" look identical** from here

### Deny by default

```csharp
// Deny by default. Every path that does not positively establish permission
// must leave this false.
var allowed = false;

try
{
    var caseEntity = await _store.GetAsync(caseId, ct);
    allowed = caseEntity is not null
              && caseEntity.CreatorUserId == tenant.UserId
              && await _holdGuard.IsClearAsync(caseId, ct);
}
catch (OperationCanceledException)
{
    throw;                       // caller went away; not an authz decision
}
catch (Exception ex)
{
    // Fail closed: an unavailable policy store is not an authorisation.
    _logger.LogError(ex, "Authorization check failed closed for case {CaseId}.", caseId);
}

if (!allowed)
{
    return Forbid();
}
```

That is longer than the original — and that's the point. A few lines buy an unassailable decision; getting it wrong is an auth bypass. The costs aren't comparable.

## Crashing is public

A crash hands the attacker a gift.

**What a stack trace leaks**

- Framework version, namespaces, file layout
- `KeyNotFoundException` → you reference a key that doesn't exist (a bug, likely guessable from context)
- `NpgsqlException` → you have a database, and its schema does something unexpected

That is reconnaissance, and it is free.

**What a crash leaks in state**

- A connection that never got closed
- A session that never got invalidated
- A token still valid because the logout path died mid-run

On a hard crash the process is gone, so sockets close. But a token that was **minted and never revoked** is still out there, and whatever it was scoped to is still accessible to whoever holds it.

### The post-failure checklist

- [ ] Close connections and release pooled handles
- [ ] Clear sensitive values from memory and any cache outliving the request
- [ ] Invalidate sessions; revoke tokens mid-issue
- [ ] Return an opaque message to the caller; keep the detail in the log
- [ ] Give the caller a correlation id it can quote

That last one is the pattern I'd defend hardest:

- Client gets a correlation id and a generic message
- Log gets the full exception
- The caller can report the id; the log reader finds everything
- **The error body never becomes a data channel**

## Do this on Monday

Pick the endpoint where a single failure produces the least useful response. Ask:

> *If this branch throws, what does the client see?*

Trace it all the way to the HTTP response — **not** to the `catch` block.

Then:

```bash
# Find the fallback and every prose-dependent guard
rg -n '_ =>' --glob '*Middleware.cs'
rg -n 'ex\.Message\.Contains' --glob '*Middleware.cs'
```

Count how many of your domain's exception types are **never named** in that file.

- In the codebase this came from, the answer was a non-zero number
- Every one of them was a 500 waiting for the right input
- That number is your real error-handling coverage
- **Nobody tracks it anywhere**

## Recap

- In an accretion-style switch, the **default branch is the posture** — the list of known errors tells you nothing about the unknown ones
- Make the unlisted case visibly wrong; opt *into* status codes rather than falling into a plausible 500
- Message matching makes correctness depend on prose. Use exception types
- Fail-safe releases on power loss; fail-secure engages. Decide which, deliberately
- Deny by default — an unavailable policy store is not an authorisation
- Crashes leak stack traces *and* live credentials. Both are yours to clean up
- Count the unmapped exception types. That number is your real coverage

---

*The middleware excerpt is generalised from a real codebase — names and messages changed, structure and the bug preserved. The fail-safe/fail-secure distinction follows the standard one drawn from powered locking systems: fail-safe releases on power loss, fail-secure engages on power loss.*