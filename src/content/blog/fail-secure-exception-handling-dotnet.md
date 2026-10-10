---
title: "The Exception Table That Fails Open"
description: "A crash-handling middleware that looks rigorous but silently routes every unlisted error to a 500 — and why the default branch is the whole security posture. Plus the fail-safe/fail-secure distinction everyone conflates."
pubDate: 2026-10-10
tags: [security, dotnet, error-handling, architecture]
draft: false
---

Here is an exception middleware. It reads like careful engineering — a typed case per domain error, stable error codes, `ProblemDetails` responses:

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

I reviewed this file as part of a security sweep. Six of its cases were correct. It was still producing 500s on six different malformed inputs to a single endpoint, and the reason was two lines of ordinary code that a service somewhere else was throwing:

```csharp
// PaymentService.cs
throw new InvalidOperationException("Payment proof can only be uploaded for cases in Payment_Pending state.");
```

`InvalidOperationException` **is** in the switch. But the arm has a `when` guard, and the guard checks for a different phrase. So the arm doesn't match, the match falls to `_`, and the caller gets a 500 for what is plainly a state conflict that should have been a 409.

That is the whole bug. Nothing exotic — just a new error message nobody added to a list.

## The default branch *is* the security posture

Every one of those listed arms is a decision someone made deliberately. The `_` arm is what happens to everything nobody thought about. In a switch that looks this thorough, the default branch is invisible during review — the eye goes to the six carefully-named cases and concludes the file is covered.

That is the structural problem: **the table grows by accretion, but the exposure grows by everything absent from it.** A reviewer reading the switch is reading the list of known errors, and calling the design robust because the list is long. The list being long says nothing about the size of the complement.

The fix inverts the pressure. Make the unlisted case the one that is visibly wrong, and make the interesting status codes something you opt into:

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

Same behaviour for the client, entirely different behaviour for the next developer. The `Payment_Pending` throw now lands in an arm that says, in the source, "this was never mapped" — which is a review question, and the right one.

## Why I stopped using message matching

The `when ex.Message.Contains("already been resolved")` line is the actual smell. It makes correctness depend on prose, and prose is the part of a codebase nobody versions with any discipline:

- someone rewords the message to "has already been resolved" and the 409 silently becomes a 500
- someone changes `Contains` to `StartsWith` for tidiness and a legitimate case stops matching
- the message is localised, and the guard breaks for non-English callers
- two different exceptions share a phrasing and you cannot tell which one fired

Exceptions already carry their identity in their type. `throw new OfferAlreadyResolvedException(...)` is a compile-time fact; `Contains("already been resolved")` is a runtime guess. Prefer the first, and if you are stuck with the second, at least fail loudly when it misses — which is what the arm above does.

## Fail-safe and fail-secure are not the same word

The distinction gets collapsed constantly, and the collapse is itself a security bug, because the two point at opposite answers to "what should happen when the thing breaks?"

**Fail-safe** means the failure moves the system toward the safe *physical* state. A fail-safe lock **unlocks** the door when power is cut — it assumes loss of power is an emergency egress, so a person is never trapped inside.

**Fail-secure** means the failure moves the system toward the safe *security* state. A fail-secure lock **locks** the door when power is cut — it assumes loss of power may be an attack, so nothing gets in.

Neither is universally correct, which is exactly why you cannot let the default decide for you. Choosing wrong in the physical direction gets someone trapped. Choosing wrong in the security direction lets an attacker walk in during a power cut. Both are real outcomes of the same ambiguity.

The failure mode to avoid is the accidental default. A component whose behaviour under failure nobody decided will default to whichever way its control flow happens to fall — and that is decided by nobody, which means it is decided by accident.

## The inverted condition is the one to memorise

The quiz attached to this material gives you this:

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

The comments say one thing, the branches do the other, and every attacker who finds this gets in on the failure path. Note the shape, because the real-world version almost never looks this silly:

```csharp
if (await _store.CanMutateAsync(caseId, ct))
{
    await _store.MutateAsync(caseId, ct);   // allowed
}
```

Nobody writes the comments that contradict the branch. But consider what happens when `CanMutateAsync` throws, times out, or the store returns something the compiler cannot type-check — the entity framework returns `0` instead of throwing on a failed query, so a lookup that finds nothing and a lookup that fails look identical from here.

The defensive form makes the decision explicit at the point where it is made, and inverts the branch so the safe state is what happens when you don't get a clear answer:

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

That is longer than the original and that is the point. The cost of a security decision being unassailable is a few lines; the cost of getting it wrong is an authentication bypass, and the two costs are not comparable.

## The part people forget: crashing is public

When the process dies it hands the attacker a gift. A crash dumps a stack trace; a stack trace names your framework version, your namespaces, and your file layout. It distinguishes a `KeyNotFoundException` (you referenced a key that doesn't exist — a bug, likely guessable from context) from an `NpgsqlException` (you have a database, and the schema is doing something you didn't expect). That is reconnaissance, and it is free.

The other thing a crash leaks is *state*. A connection that never got closed, a session that never got invalidated, a token still valid because the logout path died before it ran. On a hard crash the process is gone, so the sockets close — but the token that was minted and never revoked is still out there, and whatever it was scoped to is still accessible to whoever holds it.

So the checklist after an unhandled failure is not cosmetic:

- close connections and release pooled handles
- clear sensitive values from memory and any cache that outlives the request
- invalidate sessions and revoke tokens that were mid-issue
- return an opaque message to the caller, keep the detail in the log, and correlate them with an id the caller can quote

That last one is the pattern I would defend hardest. The client gets a correlation id and a generic message; the log gets the exception. The caller can report the id, and the reader of the log can find everything — without the error body becoming a data channel.

## What to do on Monday

Pick the endpoint in your codebase where a single failure produces the least useful response, and ask one question: *if this branch throws, what does the client see?* Trace it all the way to the HTTP response, not to the `catch` block. If the answer is "whatever the default happens to be," you have your next ticket.

Then do this: **grep your error middleware for the `_ =>` arm and for `when ex.Message.Contains`, and write down how many exception types in your domain layer are not named anywhere in that file.** In the codebase this came from, the answer was a non-zero number, and every one of them was a 500 waiting for the right input. That number is your real error-handling coverage, and nobody tracks it anywhere.

---

*The middleware excerpt is generalised from a real codebase — names and messages changed, structure and the bug preserved. The discussion of fail-safe versus fail-secure follows the standard distinction drawn from powered locking systems: fail-safe releases on power loss, fail-secure engages on power loss.*