Title: A group nested in a chain inside another group takes the outer group's id

<!--
Please fill this template entirely and do not erase parts of it.
We reserve the right to close without a response
bug reports which are incomplete.
-->
# Checklist
- [x] I have verified that the issue exists against the `main` branch of Celery.
- [ ] This has already been asked to the [discussions forum](https://github.com/celery/celery/discussions) first.
- [x] I have read the relevant section in the
  [contribution guide](https://docs.celeryq.dev/en/main/contributing.html#other-bugs)
  on reporting bugs.
- [x] I have checked the [issues list](https://github.com/celery/celery/issues?q=is%3Aissue+label%3A%22Issue+Type%3A+Bug+Report%22+-label%3A%22Category%3A+Documentation%22)
  for similar or identical bug reports.
- [x] I have checked the [pull requests list](https://github.com/celery/celery/pulls?q=is%3Apr+label%3A%22PR+Type%3A+Bugfix%22+-label%3A%22Category%3A+Documentation%22)
  for existing proposed fixes.
- [x] I have checked the [commit log](https://github.com/celery/celery/commits/main)
  to find out if the bug was already fixed in the main branch.
- [x] I have included all related issues and possible duplicate issues
  in this issue (If there are none, check this box anyway).
- [ ] I have tried to reproduce the issue with [pytest-celery](https://docs.celeryq.dev/projects/pytest-celery/en/latest/userguide/celery-bug-report.html) and added the reproduction script below.

## Mandatory Debugging Information

- [x] I have included the output of ``celery -A proj report`` in the issue.
    (if you are not able to do this, then at least specify the Celery
     version affected).
- [x] I have verified that the issue exists against the `main` branch of Celery.
- [x] I have included the contents of ``pip freeze`` in the issue.
- [x] I have included all the versions of all the external dependencies required
  to reproduce this bug.

## Optional Debugging Information
- [x] I have tried reproducing the issue on more than one Python version
  and/or implementation.
- [ ] I have tried reproducing the issue on more than one message broker and/or
  result backend.
- [ ] I have tried reproducing the issue on more than one version of the message
  broker and/or result backend.
- [ ] I have tried reproducing the issue on more than one operating system.
- [ ] I have tried reproducing the issue on more than one workers pool.
- [x] I have tried reproducing the issue with autoscaling, retries,
  ETA/Countdown & rate limits disabled.
- [x] I have tried reproducing the issue after downgrading
  and/or upgrading Celery and its dependencies.

## Related Issues and Possible Duplicates

#### Related Issues

- #10829 (reported there in review; that PR fixes a different as_tuple() duplicate and does not change this one)

#### Possible Duplicates

- None

## Environment & Settings
**Celery version**: main (b69c634), also 5.6.3 and 5.3.6

<details>
<summary><b><code>celery report</code> Output:</b></summary>
<p>

```
software -> celery:5.7.0b1 (collider) kombu:5.7.0b1 py:3.12.3
            billiard:4.3.1 py-amqp:5.4.1
platform -> system:Linux arch:64bit, ELF
            kernel version:6.18.44-fc-v114 imp:CPython
loader   -> celery.loaders.default.Loader
settings -> transport:amqp results:disabled

deprecated_settings: None
```

</p>
</details>

# Steps to Reproduce

## Required Dependencies
- **Minimal Python Version**: N/A or Unknown
- **Minimal Celery Version**: 4.4.0rc5 (the chord-body case below: 5.6.0)
- **Minimal Kombu Version**: N/A or Unknown
- **Minimal Broker Version**: N/A (reproduces with `memory://`, no worker needed)
- **Minimal Result Backend Version**: N/A (reproduces with `cache+memory://`)
- **Minimal OS and/or Kernel Version**: N/A or Unknown
- **Minimal Broker Client Version**: N/A or Unknown
- **Minimal Result Backend Client Version**: N/A or Unknown

### Python Packages
<details>
<summary><b><code>pip freeze</code> Output:</b></summary>
<p>

```
amqp==5.4.1
billiard==4.3.1
celery @ main (b69c634)
click==8.5.0
kombu==5.7.0b1
python-dateutil==2.9.0.post0
tzdata==2026.4
vine==5.1.0
```

</p>
</details>

### Other Dependencies
<details>
<p>
N/A
</p>
</details>

## Minimally Reproducible Test Case

<details>
<p>

```python
from celery import Celery, chain, chord, group

app = Celery('repro', broker='memory://', backend='cache+memory://')


@app.task
def t(x):
    return x


def ids(node):
    (task_id, parent), children = node
    yield task_id
    if parent:
        yield from ids(parent)
    for child in children or ():
        yield from ids(child)


res = group(t.si(1), chain(t.si(2), group(t.si(3), t.si(4)))).apply_async()
all_ids = list(ids(res.as_tuple()))
print('group  :', len(all_ids), 'nodes,', len(set(all_ids)), 'unique ids')
print('         inner group id == outer group id:', res.results[1].id == res.id)

res = chord(group(t.si(1), t.si(2)), group(t.si(3), t.si(4))).apply_async()
print('chord  : body group id == header group id:', res.id == res.parent.id)
```

</p>
</details>

# Expected Behavior

Every result in the tree has its own id: 6 nodes, 6 unique ids, and the inner
GroupResult's id differs from the outer one. Likewise, a chord's body group has
an id of its own, not the header's.

# Actual Behavior

On main and 5.6.3, with task protocol 1 and 2:

```
group  : 6 nodes, 5 unique ids
         inner group id == outer group id: True
chord  : body group id == header group id: True
```

The group case is the same on 5.3.6, and with or without a `task_id` passed to
`apply_async()`. `freeze()` of the same group is fine (6 nodes, 6 unique ids);
only `apply_async()` is affected. 4.4.0rc4 gives 6/6, 4.4.0rc5 gives 6/5. The
chord case prints `False` up to 5.5.3 (5.3.6 included) and `True` from 5.6.0
on.

Root cause: `group._freeze_group_tasks()` uses the `group_id` it is given as the
group's own id (`gid = opts['task_id'] = group_id or uuid()`, from a537c2d in
4.4.0rc5). Callers pass `group_id` to mean "the group this signature belongs
to": the outer group passes its id to the chain, and the chain passes it on to
its last task, so the inner group adopts the outer id. Since 2a03096 (5.6.0),
`chord.run()` also passes the header's `group_id` when freezing the body, which
gives a group body the header's id. The tasks inside the inner group carrying
the outer `group_id` is intended (that is how a chord over the outer group
counts them); only the inner group's own id is wrong.

Reported by @GangEunzzang in #10829, thank you.

Written with help from Claude (AI).
