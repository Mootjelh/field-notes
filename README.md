# 71 of the 74 times I was sure the site was blocking me, it was my own bug

I have a table in my project notes with two columns. The left one says what I
was confident about. The right one says what it turned out to be. A row gets
added every time those two disagree.

It is at 74 rows. In 71 of them the cause was my own bug or my own misreading.
Three were the platform genuinely behaving differently than I expected.

The project is a monitoring and automation tool for a large ticketing site,
which means most of my working days are spent looking at responses that are
technically fine and mean nothing I expected. That part is normal. The part
worth writing down is that nearly every time I decided the other side was
blocking me, throttling me, or behaving strangely, the other side was doing
nothing of the kind.

## Why the table exists

The problem is not being wrong. Everyone is wrong constantly and it costs
nothing, because the next test corrects you.

The problem is being wrong and writing it down.

A typical sequence went like this. I concluded something on a Tuesday, wrote it
into the project notes as a fact, and then reasoned from that fact for two
weeks. Every decision after it inherited the error. When it finally broke, the
bug was not in the code I wrote that week. It was in a sentence I had written a
fortnight earlier and never checked again.

So the table is not a list of mistakes for its own sake. It is a list of
sentences I trusted, kept so that the next person to open the notes, which is
almost always me, reads the correction in the same glance as the claim.

The total in it has been wrong twice, incidentally. Both times because I
incremented the number instead of counting the rows. A table about not trusting
unchecked numbers, with an unchecked number in it.

## An empty result is not an answer

This is the most expensive pattern in the list. It produced four separate rows.

I would call an endpoint, get a well formed response with an empty list in it,
and conclude there was nothing to find. Three completely different situations
produce exactly that response:

- the thing really is empty
- I asked the wrong path, and the server answered politely about nothing
- the data arrived and my parser dropped it

From the outside they are identical. All three are a 200 with `[]` in it.

The worst instance took an hour to find. An event was actively selling tickets,
my monitor read it every thirty seconds and reported zero available, and it was
correct that the response contained nothing. The endpoint I was calling does not
serve that region at all. It answers 200 with an empty list, forever, for events
it has never heard of. A different endpoint had the data the whole time.

Another was entirely mine. General admission tickets have no seat map, so they
arrive with an empty `places` array and the section name attached to the offer
instead. My parser skipped any entry with no places. Thirty seven percent of one
venue was invisible, and the log line for that venue was identical to the log
line for a sold out one.

The rule I use now: when a read comes back empty, do not conclude anything until
you have proved the shape of the request against data you know exists. An empty
result tells you about your request at least as often as it tells you about the
world.

## Two failures that look the same and need opposite fixes

A structured error means you reached the server and your payload is wrong. A
block page means you never got there.

```
{"code":"Error.BadRequest","detail":{...}}    you arrived, fix the payload
{"response":"block"}                          you did not, fix the address
```

These need completely different work. One is a field in your request body. The
other is your IP, your TLS fingerprint, or your rate. I confused them more than
once, because both come back as a failure and both are JSON.

They are separated at the point the response is read now, with a sentinel error,
rather than by matching a phrase further up the stack. That last part matters.
For a while the monitor decided whether a read had been blocked by looking for
the words "blocked by the edge" in an error message. The edge has two different
block responses, and the shorter one produced a different message. So 108 blocked
reads over 33 hours arrived as ordinary failures, three separate recovery
mechanisms all missed them, and the connection stayed blind. Every one of those
mechanisms was keyed on a sentence.

## A test that passes proves nothing until you have seen it fail

I wrote a test to pin down a safety property. It passed. I moved on.

Months later I deleted the check it was supposed to be guarding, to see what
would happen, and the test still passed. Its fixture was empty, so the code under
test returned early every time and never reached the comparison the test was
about. Every assertion in it was true for the wrong reason.

Now, when a test guards something that matters, I delete the guard and run it. If
it does not go red, the test is decorative. This takes about two minutes and it
has caught three tests that were checking nothing.

The same applies to a fix. Reverting the fix and watching the test fail is the
only evidence that the test and the fix are about the same thing.

## A number is not a rate until you have grouped it correctly

One night's log had 524 failed reads across 56 events over roughly eleven hours.
Against the total number of reads that is about one percent, which looks like
background noise, and I wrote it down as background noise.

Grouped by connection instead of by event, 461 of those 524 are seven events on
a single connection, every read timing out, for forty five minutes straight,
while five other events on that same connection worked fine throughout.

Same 524 numbers. Grouped one way it is a rate you accept. Grouped the other way
it is an outage you fix. Nothing about the data changed.

The general version: before accepting a number as a background rate, group it by
every dimension you have. A rate that is really an incident collapses into one
bucket.

## Check the cheapest thing first

At one point everything went silent. Every read on every event failed. The only
symptom was a generic token error, repeated once per event per scan, which looks
exactly like a network problem.

The captcha solving account had 0.009 dollars left on it.

Nine tenths of a cent, about nine more solves, on a system that spends 350 a day.
There was no degraded mode. No token means no read, on everything, at once.

It took a while because I went looking for a subtle problem first. The check that
would have found it immediately is one unauthenticated HTTP request that returns
a number, and nothing in the system was making it. It does now, and the startup
output prints the balance as days remaining rather than as an amount, because an
amount invites you to glance at it and a countdown does not.

## What I take from this

None of these are clever. They are all versions of the same thing: the story in
your head is cheaper to produce than the measurement, so it gets produced first,
and then it quietly becomes the thing you reason from.

The table works because it puts the correction physically next to the claim. I
cannot read the confident version without reading what it turned out to be. That
is the only trick in it.

If you keep one, count the rows before you quote the total.
