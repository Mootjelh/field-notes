# Field notes: reading a bot-protected site from the outside

Notes from building a monitoring and automation tool against a large ticketing
site that sits behind a CDN edge, a WAF and a waiting room. Every section is
something that cost real time, what was actually going on, and the check that
tells the cases apart. All the numbers are measured.

## The rules, short

1. A structured error means you reached the origin. A block page means you did
   not. Separate them where the response is read, never by matching an error
   message.
2. When a connection starts failing, ask what share of its work is failing.
   One target out of twelve is that target. Twelve out of twelve is the
   address.
3. A cache can serve more than one stale copy of the same URL at once. Read the
   `age` header before trusting a diff between two reads.
4. An empty result is not an answer until the same request has returned data
   for something you know exists.
5. Before accepting a failure count as a background rate, group it by every
   dimension you have.
6. When your code reimplements a decision a reference already makes, compare
   against the reference over the whole input space.
7. A defect you can read is not always one you can reach. Run the end to end
   path before telling anyone their code crashes.
8. A tool with two verdicts has to prove it can print both.
9. A test server that closes with unread data sends a reset, which can take the
   response with it.

## Contents

- [A block page and an error are opposite problems](#a-block-page-and-an-error-are-opposite-problems)
- [Which share of the connection is failing](#which-share-of-the-connection-is-failing)
- [The cache was two objects](#the-cache-was-two-objects)
- [An empty result is not an answer](#an-empty-result-is-not-an-answer)
- [A count is not a rate until it is grouped](#a-count-is-not-a-rate-until-it-is-grouped)
- [Compare against the reference](#compare-against-the-reference)
- [A defect you can read and a defect you can reach](#a-defect-you-can-read-and-a-defect-you-can-reach)
- [A verdict the tool could not reach](#a-verdict-the-tool-could-not-reach)
- [Closing a socket with unread data is a reset](#closing-a-socket-with-unread-data-is-a-reset)
- [How these notes are kept](#how-these-notes-are-kept)
- [Elsewhere](#elsewhere)

## A block page and an error are opposite problems

Two failures come back looking alike. Both are a failed request and both are
JSON:

```
{"code":"Error.BadRequest","detail":{...}}    reached the origin: fix the payload
{"response":"block"}                          stopped at the edge: fix the address
```

The first is a field in your request body. The second is your IP, your TLS
fingerprint or your request rate. Work on the wrong one and nothing changes.

The less obvious part is where the distinction is made. For a while the tool
decided whether a read had been blocked by looking for the words "blocked by
the edge" in an error message. The edge has two block bodies, a long one and a
short one, and the short one produced a different message. Over 33 hours, 108
blocked reads arrived as ordinary failures. Three separate recovery mechanisms
missed every one of them, because all three were keyed on that sentence, and
the connection stayed blind.

The fix is a sentinel error returned at the point the body is parsed, covering
both signatures, so everything further up asks `errors.Is` and nothing reads
prose. A test pins the other half too: a structured body from the origin must
not count as a block.

## Which share of the connection is failing

A block page has a third meaning. Two events went behind a waiting room one
afternoon and refused every read for the next nine hours, 2,696 refusals, each
one the same body an address gets when the edge has had enough of it. So each
one was handled as a refused address: the connection was dropped and a fresh
session token was bought for the next read. Roughly three quarters of that
day's spend went on those two events.

Nine other events on the same connection read normally the whole time, and
that is what separates the two cases. A refused address refuses everything sent
through it. A gated event refuses itself and leaves its neighbours alone. The
useful number is the share of a connection's events being refused: two of
twelve is an event, twelve of twelve is the address.

The first version of that fix had a hole that only running it found, half an
hour after it was committed. It proved an address healthy by a neighbour's
successful read, and at startup no neighbour had read yet. Three refusals in
two seconds tripped the address breaker and took twelve events offline for
five minutes. The share rule has no such gap, since it needs no earlier
success to compare against.

## The cache was two objects

Reads of the same URL thirty seconds apart returned two different bodies,
alternating. The `age` header explained it:

```
read   1    2    3    4    5    6
age   53   54  113  114  173  174
```

Two objects, born 29 seconds apart, each ageing sixty seconds a minute, each
served for three minutes against a declared `max-age` of sixty. A block present
in one generation and missing from the other looked like inventory appearing
and disappearing on every read, and any diff between consecutive reads was
partly a diff between two caches.

Then the measurement that mattered. The site also pushes a notification when
inventory changes. Across 152 of those pushes, a read straight after each one
returned an object built before the push every time. Median lag 70 seconds,
worst 176.

A cache-busting query parameter changed nothing: same generation, same weak
etag, same 29,940 bytes, `age` climbing straight through it. A shield in front
of the origin serves one copy per window whatever the URL says.

What that changed was where the latency work goes. The chain from noticing a
change to acting on it takes under two seconds. The copy the change is noticed
in is typically a minute and a half old. Shaving the two seconds was work on the
wrong end.

## An empty result is not an answer

A well formed response with an empty list has at least three causes, and they
look identical from outside:

- the thing really is empty
- the request was wrong, and the server answered politely about nothing
- the data arrived and the parser dropped it

Two real cases. An event was on sale while the monitor read it every thirty
seconds and reported nothing available. The endpoint being called does not
serve that region, and answers 200 with an empty list for events it has never
heard of. A different endpoint had the data all along.

In the other, general admission tickets arrived with an empty `places` array
and the section name on the offer, because standing areas have no seat map. The
parser skipped entries with no places, so 37% of one venue was invisible, and
the log line for that venue was identical to the log line for a sold out one.

The check: before concluding anything from an empty read, run the same request
shape against data you know exists. If that is empty too, the problem is the
request.

## A count is not a rate until it is grouped

One night's log had 524 failed reads across 56 events in about eleven hours.
Against the total that is roughly one percent, which reads as background noise.

Grouped by connection instead of by event, 461 of the 524 were seven events on
one connection, every read timing out, for forty-five minutes straight, while
five other events on that same connection read fine.

Same numbers. Grouped one way it is a rate you accept. Grouped the other way it
is an outage you fix. A rate that is really an incident collapses into one
bucket as soon as you group it by the right dimension, so group by all of them
before deciding which one you have.

## Compare against the reference

This one is from [a fix to an open-source HTTP
client](https://github.com/bogdanfinn/fhttp/pull/27): deciding whether the
first two bytes of a compressed body are a zlib header.

The check had three conditions and a test that generated real headers with the
standard library's encoder and asserted they were all recognised. Reverting the
fix turned the test red, which looked like enough. Deleting one of the three
conditions, so the check accepted more than it should, left it green. The only
negative case failed a different condition, so nothing held that one.

Adding examples is the obvious repair, and it is how the previous bug of this
kind happened: two headers worked out by hand for a table were not valid at all.

The standard library already answers this exact question, and there are only
65,536 two-byte headers. So the test walks every one of them, asks the
reference, and asserts the check agrees. They agree exactly, 66 accepted by
both and no disagreement either way, and deleting any of the three conditions
now fails.

The general form: when your code reimplements a decision something else makes
correctly and the input space is small enough, enumerate it and compare. And
check a test in both directions. A test that only ever goes red when the fix is
reverted has only been shown to catch one kind of mistake.

## A defect you can read and a defect you can reach

Also from [someone else's HTTP client](https://github.com/imroc/req/pull/538),
chasing a crash report open for a year.

The cause was easy to see. One function is reached from two places and only one
of them runs the setup that fills in three fields. Come in through the other
and the fields are nil, and the code that uses them runs in its own goroutine,
so the nil dereference takes down the whole process with no chance to recover.
Two lines reproduce it.

The end to end test, a real server and an ordinary client, did not crash. Every
ordinary request passes through a third method that runs the setup, so the
fields are filled in before the second path is ever used. The defect is real,
but it is only reachable before anything has run that setup, which is why the
reporter saw it intermittently under load.

"This code is wrong" came from reading. "You can get here with it wrong" only
came from running it, and that half was the one that explained what the
reporter actually saw. Writing up the first version would have told a
maintainer their library crashes on a path where it does not.

## A verdict the tool could not reach

The tool that measured the cache above prints one of two verdicts per read,
`FRESH` or `STALE`. `FRESH` was decided from the `last-modified` header, which
came back empty on every read. So a tool whose whole output is a verdict had one
verdict it could never print, and nothing said so.

It now reconstructs the object's age from whichever header is present and says
which one it used. When a tool exists to say one of two things, make it show it
can say both.

## Closing a socket with unread data is a reset

A small test server recorded what a client sent, wrote a response and closed.
In two runs out of six the client reported the connection forcibly closed and
never saw the response.

The client kept talking after its request, an HTTP/2 settings acknowledgement
in this case, and the server closed with those bytes unread. A close with
unread data in the receive buffer is a reset, and a reset can discard the
response the client was about to read. The server now drains until the test is
done with the client. Any test server that answers and hangs up needs the same.

## How these notes are kept

Every conclusion that turns out wrong goes into a two-column table: what I was
confident about, and what it turned out to be. It is at 174 rows. In 169 of
them the cause was in my own code or my own reading of a response, and in five
the platform genuinely behaved differently than expected.

The sections above are the ones that generalise. The table is what produces
them, because it puts the correction next to the original claim, so a sentence
written two weeks ago cannot quietly become the thing everything else is
reasoned from.

If you keep one, count the rows before you quote the total. Mine was wrong
twice because the number had been incremented instead of counted.

## Elsewhere

Small Go packages that came out of the same work:

- [hardiff](https://github.com/Mootjelh/hardiff) compares two HTTP Archive
  captures and shows what changed.
- [identlint](https://github.com/Mootjelh/identlint) checks that a request's
  headers agree with the browser they claim to be.
- [proxypool](https://github.com/Mootjelh/proxypool) is a rotating proxy pool
  with cooldowns.
- [flatread](https://github.com/Mootjelh/flatread) reads FlatBuffers buffers
  when you do not have the schema, and
  [flatschema](https://github.com/Mootjelh/flatschema) infers one from samples.
