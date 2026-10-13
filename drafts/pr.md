Title: Keep a nested group from taking its parent group's id

*Note*: Before submitting this pull request, please review our [contributing
guidelines](https://docs.celeryq.dev/en/main/contributing.html).

## Description

Fixes #<issue>

`group._freeze_group_tasks()` used the `group_id` it was given as the group's
own id. But `group_id` names the group a signature belongs to, so a group that
ends a chain inside another group, e.g.
`group(a, chain(b, group(c, d)))`, got the outer group's id, and
`as_tuple()` listed that id twice. Since 5.6.0, a group used as a chord body
gets the header's id the same way.

What changed (`celery/canvas.py`, 2 lines changed, 1 added):

- `group._freeze_group_tasks()` gives the group a fresh id when it has none,
  instead of `group_id or uuid()`. This is the behaviour before 4.4.0rc5.
- `chord.run()` was the only caller relying on the old behaviour, to give the
  header the chord's group id. It now sets `header.options['task_id']`
  explicitly before freezing the header.

Chord counting is unchanged: the tasks inside the nested group still get the
outer group's id as their `group_id`. On a worker (in-memory broker, threads
pool, protocol 1 and 2), for `group(a, chain(b, group(c, d)))`,
`c` and `d` see `request.group == <outer id>` before and after this change,
and `chord(group(a, chain(b, group(c, d))), body.s())` runs its body with
`['a', ['c', 'd']]` both before and after.

Tests:

- `test_group::test_group_ending_a_chain_in_a_group_gets_its_own_id` (protocol
  1 and 2) and `test_chord::test_group_body_does_not_take_the_header_id`: all
  3 fail on main and pass with this change.
- `t/unit/tasks`: 1148 passed on main, 1151 passed with this change (the 3 new
  tests), no failures.
- flake8 and isort are clean.

Possible breakage: code outside Celery that calls
`group.freeze(group_id=X)` and expects the result's id to be `X` will now get
a fresh id. Inside Celery, only `chord.run()` relied on that. `group_id` now
means the same thing for a group as it does for `Signature.freeze()`: the
group the signature belongs to.

Written with help from Claude (AI).
