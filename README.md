# 110 of the 115 times I was sure the site was blocking me, it was my own bug

I have a table in my project notes with two columns. The left one says what I
was confident about. The right one says what it turned out to be. A row gets
added every time those two disagree.

It is at 115 rows. In 110 of them the cause was my own bug or my own misreading.
Five were the platform genuinely behaving differently than I expected.

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

## Who else is failing on that connection

The block page above has a third meaning I only found in September. Two events
went behind a waiting room one afternoon and refused every read for the next
nine hours: 2,696 refusals, each one the same block page an address gets when
the edge has had enough of it. So each one was treated as the address being
rejected, the connection was thrown away, and a fresh token was bought for the
next one. Roughly three quarters of that day's bill was those two events.

Nine other events on the same connection read normally the whole time. That is
the fact that separates the two cases, and nothing was looking at it. A
rejected address refuses everything that goes through it. A gated event refuses
itself and leaves its neighbours alone. So the question is not how many
refusals a connection has seen, it is what share of its events are being
refused, and two of twelve is an event while twelve of twelve is an address.

The fix went in, and the run meant to confirm it found the hole within half an
hour. It proved an address healthy by a neighbour's successful read, and at
startup no neighbour had read yet, so three refusals inside two seconds tripped
the address breaker and took twelve events off the air for five minutes.
Reading the fix would not have found that. Running it did, thirty minutes after
I had committed it.

## The cache was two objects

Reads of the same resource thirty seconds apart came back with two different
bodies, alternating. The `age` header gave it away: 53, 54, 113, 114, 173, 174.
Two objects, born 29 seconds apart, each climbing exactly sixty a minute, each
served unrefreshed for three minutes against a declared maximum age of sixty.

So my view of an event flipped between two moments up to half a minute apart,
and a block present in one generation and absent in the other looked like
inventory appearing and disappearing on every read. Then the harder
measurement: 152 pushes from the site saying inventory had moved, a read right
after each one, and every single read returned an object built before the
push. Median lag 70 seconds, worst 176.

A cache-busting query parameter did nothing. Same generation, same weak etag,
same 29,940 bytes, `age` climbing straight through it. A unique URL does not
reach the origin when a shield sits in front of it.

None of this was a bug I could fix. What it changed is what I compare my own
latency against. My chain from seeing a change to acting on it takes under two
seconds. The copy I see the change in is typically a minute and a half old.
Optimising the two seconds was the wrong end.

## A verdict the tool could not reach

The tool that measured that cache had two verdicts, `FRESH` and `STALE`.
`FRESH` was decided from the `last-modified` header. That header came back
empty on every read, and the tool's own notes said so, two paragraphs below the
code that needed it.

So a tool whose entire output was a verdict had one verdict it could never
print, and nothing said so. It reconstructs the object's age from whichever
header is present now, and says which one it used. When a tool exists to say
one of two things, make it prove it can say both.

## Closing a socket with unread data is a reset

Not from the ticketing project. A test I wrote started a small server, recorded
what a client sent, wrote a response, and closed. Two runs in six the client
reported the connection forcibly closed and never saw the response.

The client had kept talking after its request, a settings acknowledgement in
this case, and the server closed with those bytes unread. A close with unread
data in the receive buffer is a reset, not a shutdown, and a reset can take the
response the client was about to read with it. The server drains until the test
is done with the client now. Any test server that answers and hangs up needs
the same.

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
