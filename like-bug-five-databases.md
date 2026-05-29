# The same `LIKE` bug in five local-first databases

Local-first and sync engines all share one hard problem: the **client** has to
re-evaluate queries against its local copy of the data (for optimistic updates,
offline reads, and reactive subscriptions), and that client-side evaluation has
to agree with whatever the **server/SQLite/Postgres** would have returned. The
moment the two disagree, you get *silent wrong results* — a row shows up locally
that the server would never return, or vice-versa. For a database, that's the
worst class of bug: no error, no crash, just quietly incorrect data.

Over the last few days I audited how a handful of popular local-first / query
engines implement one tiny, ubiquitous operator — SQL `LIKE` — in their
client-side matcher. **Five of them got it wrong, in the same family of ways.**
All five now have fixes proposed:

| Project | Root cause | Symptom | PR |
| --- | --- | --- | --- |
| **ElectricSQL** | regex without dotall, `^…$` anchoring, escaped `\%` mishandled | `%`/`_` miss newlines; `'ab\n' LIKE 'ab'` true; `'x' LIKE '\%'` wrong | [electric#4437](https://github.com/electric-sql/electric/pull/4437) |
| **Triplit** | regex without the dotall flag | `'a\nb' LIKE 'a%b'` → false (should be true) | [triplit#407](https://github.com/aspen-cloud/triplit/pull/407) |
| **Zero (Rocicorp)** | `m` (multiline) flag instead of `s` (dotall) | false negatives **and** false positives vs SQLite | [mono#6083](https://github.com/rocicorp/mono/pull/6083) |
| **WatermelonDB** | regex metacharacters never escaped | `Q.like('a.b')` matches `'axb'`; `Q.like('a(b')` throws | [WatermelonDB#1968](https://github.com/Nozbe/WatermelonDB/pull/1968) |
| **InstantDB** | regex without the dotall flag | `$like` misses newlines; diverges from the Postgres server | [instant#2714](https://github.com/instantdb/instant/pull/2714) |

## The pattern

SQL `LIKE` is deceptively simple: `%` matches any run of characters, `_` matches
exactly one, **everything else is a literal**, and — crucially — the wildcards
match *any* character, **including newlines**. The whole value must match.

The usual implementation compiles the pattern to a regex:

```js
// the tempting version
const re = new RegExp('^' + pattern.replace(/%/g, '.*').replace(/_/g, '.') + '$')
```

This has three independent bugs, and the five projects above each hit one or
more of them:

1. **No dotall.** In JavaScript regexes, `.` does **not** match `\n` unless you
   pass the `s` flag. So `%`→`.*` and `_`→`.` silently stop at a newline.
   `'a\nb' LIKE 'a%b'` returns `false`, but every SQL database returns `true`.
   Any text column with a newline (addresses, markdown, notes, logs) is affected.

2. **No metacharacter escaping.** If you don't escape the pattern, characters
   like `.`, `+`, `(`, `[` are treated as *regex*, not literals. `LIKE 'a.b'`
   then matches `'axb'`, and `LIKE 'a(b'` throws `Invalid regular expression`.

3. **Wrong anchor flag.** Using the `m` (multiline) flag makes `^`/`$` match at
   interior line boundaries, so `'fooa\nbar' LIKE 'foo_'` returns `true` even
   though the value is 8 characters long. (SQLite: `false`.)

## A portable test

You can check your own engine in ten seconds. The correct mapping escapes
metacharacters, keeps `%`/`_` as wildcards, anchors to the whole string, and
uses dotall:

```js
function likeToRegExp(pattern, caseInsensitive = false) {
  const escaped = pattern.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
  const body = escaped.replace(/%/g, '.*').replace(/_/g, '.')
  return new RegExp('^' + body + '$', caseInsensitive ? 'is' : 's')
}
```

And the cases that catch the bugs (all should match a real SQL `LIKE`):

```
'a\nb'   LIKE 'a%b'   => true     // % spans newlines
'a\nb'   LIKE 'a_b'   => true     // _ matches a newline
'ab\n'   LIKE 'ab'    => false    // whole value must match
'fooa\nb' LIKE 'foo_' => false    // no interior-line matching
'a.b'    LIKE 'a.b'   => true     // dot is a literal
'axb'    LIKE 'a.b'   => false    // …not "any char"
'a(b'    LIKE 'a(b'   => true     // and must not throw
```

If your client-side matcher disagrees with your server on any of these, you have
silently-wrong query results in production.

## Why I went looking

I do focused **database-compatibility hardening** — I diff a client/embedded
query evaluator against the real database semantics it's supposed to mirror,
then ship reduced repros, fixes, and regression tests. The five PRs above are
the output of a few days of that.

If you maintain a local-first DB, sync engine, or anything that re-implements
query semantics in another language and you want this done systematically
(LIKE is just the easiest one to demo — the same divergences show up in
null/three-valued logic, type coercion, collation/ordering, and numeric edge
cases), I run it as a fixed-scope **48-hour compatibility-hardening sprint**:
one bounded area, fixed price, reduced repros + fixes + tests + handoff notes.

→ **[Book a sprint](https://buy.polar.sh/polar_cl_z0eLsPUJeMwrcNs4MQPAQbKIM3Rbdb8fLDgVj2RZcmr)** · or reach me at sravan272001@gmail.com

*(All five PRs were opened constructively, with tests, against the projects'
own contribution guidelines. Thanks to those teams for building in the open.)*
