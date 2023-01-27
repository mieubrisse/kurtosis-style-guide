Flexible Minimalism
===================
A core tenet of the Kurtosis code style is flexible minimalism. This means that the codebase is:

- Minimal: it only solves the requirements that we know about now
- Flexible: it can be quickly adapted to solve new requirements

### Minimalism
The burden of a codebase increases exponentially with its size, due to the interconnections between components. The smaller the codebase, the easier it is to understand and work with. Therefore, we aim to keep our codebase as small as possible.

It's tempting to do "just-in-case engineering", adding things just in case they're used in the future. For example:

- "I'll create an interface for this struct just in case somebody wants to add a new implementation in the future"
- "I'll add this API call just in case someone needs it"
- "I'll add getters and setters, just in case somebody needs them in the future"
- "I'll extract this functionality out into a public function, just in case someone else needs it"
- "I'll create this functionality as a separate library, just in case someone wants to use it in the future"

All of these add complexity _now_ for a _possible_ gain in the future. Worse still, each of these is very easy to do in the future. So we'd be eating code complexity cost _now_ for a gain that may never show up, and which would be very easy to implement in the future anyways.

Instead, it's better to wait until the need arises and _then_ do the refactor because you'll have more information about what the problems space looks like.

### Flexibility
If our codebase is minimal, by definition it will only solve the problems we know about now. Every new requirement will necessarily require modification. Thus, our codebase will be changing a lot so we need a codebase that's easy to change.

In practice, this means:

- Coding defensively, so we can make changes without fear of silently breaking something
- Testing, so we have guardrails that allow us to make changes quickly
- Writing simple and explicit code, so it's easy to make changes
- Naming things clearly, so we're not confused looking at our own code

Flexibility is particularly important due to the underlying Kurtosis philosophy around the future. We believe that it is exceedingly difficult, and often impossible, to predict the future due to black swans. If we can adapt quickly, we can survive negative black swans and take advantage of positive black swans.

The alternative would be a codebase built by people who think really, really hard about things in an ivory tower, trying to predict everything. This _might_ work for a short time, but we believe that reality will ultimately prevail: the codebase will become bloated with overengineered waste, it will be harder to adapt, and the company will lag behind and die.

Adaptation is the key to success, and a flexible codebase ensures survival.

### Example
We chose a key-value store for the Kurtosis database rather than a relational database. We did this because we'll eventually need to be fault-tolerant when we go to the cloud, relational databases have problematic replication, and switching our datamodel from relation to key-value in the future would be a huge pain. This prevented us from painting ourselves into a corner (flexibility).

However, we chose [BoltDB](https://github.com/etcd-io/bbolt), rather than [etcd](https://github.com/etcd-io/etcd), for our first implementation of the Kurtosis database. BoltDB has a similar API as etcd, but is much simpler to start working with because it stores the database as a local file. By contrast, etcd would require standing up a full 3-node etcd cluster. This builds against the requirements we have now, which don't include high availability (minimalism).
