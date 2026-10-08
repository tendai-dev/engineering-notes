# Keeping an e-learning platform fast on constrained networks

A write-up of the engineering trade-offs behind the postgraduate e-learning platform
I built and operated at Midlands State University (Laravel, MySQL, JavaScript),
used for graded coursework by thousands of students.

## The constraint that drove everything

The platform's users weren't on fast, stable connections. Most were on intermittent
mobile data and modest Android devices, often sharing one connection across a household,
and the hardest moments were exam windows — thousands of students hitting the same
endpoints in the same hour, with zero tolerance for downtime because real grades depended
on it.

That single fact — *the network and the device are the bottleneck, not the server* —
decided most of the architecture. The goal wasn't peak elegance; it was that the core
flows (log in, open a course, submit an assessment) stayed usable when the connection was
bad and came back cleanly when it dropped.

## Decisions

**Server-rendered pages over a heavy SPA.**
The fashionable choice would have been a large client-side framework. I deliberately kept
pages server-rendered and light. Shipping a multi-megabyte JS bundle to a phone on 2G is a
cost paid by the user on every visit; a server-rendered page that's useful the moment it
arrives is not. I used JavaScript for progressive enhancement, not as the thing the page
depends on to render at all.

**Aggressive, layered caching.**
Read-heavy content (course material, schedules, static lookups) was cached so that the
common request never touched the database. During exam periods the read:write ratio is
extreme, so caching the reads is what kept the database alive under the spike. I cached at
the query level for hot data and at the page/fragment level for content that rarely
changed between requests.

**Minimise payload, not just request count.**
Images were resized and compressed to what the screen actually needed; responses were kept
lean; anything that could be deferred wasn't sent on the critical path. On a slow link,
every kilobyte is latency the student feels.

**Graceful degradation over hard failure.**
Core flows were designed to survive a dropped connection — a half-submitted action
shouldn't lose a student's work or leave the system in a bad state. Where a request could
be retried safely, it was made idempotent so a flaky network didn't cause duplicates.

**Move slow work off the request path.**
Anything heavy — notifications, report generation, bulk processing — went to background
jobs so the request the student was waiting on stayed fast. The user-facing path only did
the work needed to answer the user.

**Database that holds up under the exam spike.**
The load wasn't uniform; it was bursty and predictable (it followed the exam timetable).
I indexed for the queries that actually ran under load rather than guessing, and watched
the slow query log around exam windows to fix the real hotspots instead of imagined ones.

## The trade-off I'd defend

The main tension was resisting the heavier, more "modern" client architecture. It would
have looked more impressive in a code review and been more fun to build. But for *these*
users it would have been slower, heavier and more fragile. I'd rather earn the right to add
complexity by hitting the limits of the simpler thing first — and on this platform the
simpler, server-first approach was comfortably within its limits while being far more
reliable for the people who depended on it.

## What it taught me

Operating this myself — provisioning the Linux servers, deploying, and being on call when
something broke mid-exam — is where I learned that "it works on my machine" and "it works
for a student on a bad connection during finals" are completely different bars, and that
the second one is the only one that matters. I now reach for the simplest design that
clears the real-world bar, and add complexity only when the data says I have to.
