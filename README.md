# Osysharp.Presence

Who else is on a record **right now**, and the sentence that names whoever moved it first.

- **`RecordPresence(recordId)`** — a presence topic addressed by **any** record's id: a ticket, a case, an approval, a
  lead. Being subscribed IS having the record open, so there is nothing to post and nothing to clean up: the platform
  adds and removes a seat with the connection, and stamps **who** sits in it.
- **`RecordSeat`** — `Who` (stamped by the platform; an app cannot write it) and `Activity` ("viewing", "replying",
  "assigning", or the app's own word). **Never content** — what somebody wrote reaches an open page through the row's
  own rules, not through this topic.
- **`RecordSeats(id)`** — the strip for a record's header: each colleague with a lamp, lit while they act and dark while
  they look. The viewer is left out. **`RecordSeatsLine(id)`** folds it into one line for a list row ("Raj is replying").
- **`MovedFirst(changes)`** — a `[ConcurrencyCheck]` refusal, turned into "Ada changed Amount 2 minutes ago" from the
  facts `<Entity>.Changes(id)` hands over (field names only, never values). **`TookFirst(who, at)`** says the same for an
  ownership collision ("Ines took this 2 minutes ago").

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Presence@0;
use Osysharp.Observability;   // `Changes` — who moved a row
```

```osy
using Osysharp.Presence;
using Osysharp.Observability;

// The kit's one question: who sees colleagues on a record (and is seen there). Staff — never a customer.
policy CollaboratesOnRecords => IsStaff;

[ConcurrencyCheck]
entity Case { [MaxLength(40)] string Reference; decimal Amount; security { allow read, update when IsStaff; } }

[Route("/case/{id}")]
component CaseEditor(Guid id) {
  var cases = Case.Where(c => c.Id == id).ToList();
  var message = "";
  action Save(Case c) {
    try { c.Amount = c.Amount + 1; UnitOfWork.Commit(); message = "Saved."; }
    catch (ConflictException e) { message = WhoMovedIt(c.Id) + " — your edit is still here."; }
  }
  render {
    Stack(gap: 3) {
      RecordSeats(id);                                   // who else is here
      if (message != "") { Alert(message, tone: Tone.Warning); }
      foreach (var c in cases) { Button(label: "Add one", onPress: () => Save(c)); }
    }
  }
}

// `Changes` reads the platform's trail, so the sentence is made on the server.
string WhoMovedIt(Guid id) { return MovedFirst(Case.Changes(id)); }
```

Say what you are doing when it changes — a composer opening, an assign menu:

```osy
SeatActivity(ticket.Id, "replying");
```

## What holds whatever the app grants

- **Only who the app says.** The topic's door is `CollaboratesOnRecords`; anyone else is refused a seat, and their
  announcing puts nobody in the set.
- **Who is never written by the app.** `new RecordSeat { Who = … }` is refused at compile; the platform stamps the
  connection's principal.
- **A seat carries no words.** What is broadcast is who is there and what they are doing, never the content.
- **Online is not available.** A seat says who is looking. It is not consent to receive work — routing kits read their
  own explicit availability, never this topic.

## Its tests

The kit's test app is a small case desk: `presence.test.osy` (staff only, who is stamped, an activity and never
content), `conflicts.test.osy` (the refusal through the screen, the sentence from the trail).

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using CollaborationKitTests;

[Test(Desk)]
void the_sign_in_admits_a_right_code_and_refuses_everything_else() {
  Assert.NotEqual("", SignInWithCode("ada@desk.test", "1111"));
  Assert.Equal("", SignInWithCode("ada@desk.test", "9999"));
  Assert.Equal("", SignInWithCode("nobody@desk.test", "1111"));
}
```

From [`tests/auth.test.osy`](tests/auth.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Presence;
using Osysharp.Observability;
using CollaborationKitTests;

[Test]
void the_sentence_names_who_which_fields_and_how_long_ago() {
  var now = new DateTime(2026, 10, 1, 12, 0, 0);
  Assert.Equal("just now", Ago(now.AddSeconds(-10), now));
  Assert.Equal("a minute ago", Ago(now.AddSeconds(-70), now));
  Assert.Equal("12 minutes ago", Ago(now.AddMinutes(-12), now));
  Assert.Equal("3 hours ago", Ago(now.AddHours(-3), now));
  Assert.Equal("Ines took this 2 minutes ago", TookFirst("Ines", now.AddMinutes(-2), now: now));
  Assert.Equal("Somebody took this just now", TookFirst("", now, now: now));
}
```

From [`tests/conflicts.test.osy`](tests/conflicts.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Presence;
using CollaborationKitTests;

[Test(Desk)]
[runas(Ada)]
void a_records_seats_are_staff_only_by_the_apps_rule() {
  var id = Case.Single(c => c.Reference == "C-1").Id;
  // A customer is refused a seat. (A signed-out caller is refused too: the topic is not `[AllowAnonymous]`.)
  runas (Cus) {
    Assert.Denied(() => RecordPresence.For(id).Listen());
    // Announcing decorates YOUR OWN seat, and a refused caller has none — so it puts nobody in the set.
    SeatActivity(id, "viewing");
  }
  // ⭐ THE COUNTERPART — staff are admitted, so the refusal above is the rule, not a topic that refuses everybody.
  runas (Ada) {
    RecordPresence.For(id).Listen();
    var here = RecordPresence.For(id).Here;
    Assert.Equal(1, here.Count);
    Assert.Equal("Ada", here[0].Who!.DisplayName);
  }
}
```

From [`tests/presence.test.osy`](tests/presence.test.osy).

## Learn more

- [The Osysharp.Presence reference](https://osysharp.com/reference/realtime/collaboration-kit/)
- [Osysharp.Presence in the package catalogue](https://osysharp.com/packages/#osysharp-presence)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
