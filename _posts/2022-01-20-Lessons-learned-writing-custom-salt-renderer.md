---
layout: post
title: Lessons learned writing custom Salt renderer.
---

I had to write a custom [SaltStack](https://saltproject.io/) renderer, and while the [concept](https://docs.saltproject.io/en/latest/ref/renderers/index.html) sounded simple, it was much more painful than I expected. I ran into several "gotchas" as a beginner, so I list them here for you to learn from my mistakes.

For consistency purposes, I changed all references to the custom renderer to be called "foo", as if I were adding `salt://_renderes/foo.py`.

# Gotcha 1: Renderer is not available

The first issue I faced was with this error on the pillar trying to use the new renderer:

```
The renderer "yaml | foo" is not available
```

This error was happening even though I had followed the instructions to use the commands below:

```
salt '*' saltutil.sync_renderers
salt '*' saltutil.sync_all
salt '*' state.apply
```

These commands would even print out the `foo.py` being sent to the minions. 

The gotcha here is that `salt '*'` actually executes commands on the minions, while the pillar rendering happens on the master. So, instead, use `salt-call`:

```
salt-call saltutil.sync_renderers
```

The `salt-call` should copy the new renderer to `/var/cache/salt/master/extmods`, and you can check this folder on the master to make sure the content matches `salt://_renderes/foo.py`.

# Gotcha 2: Pillar needs to be refreshed

While in the beginning I tried to check that the pillar was being rendered correctly by creating a new state that consumed it, a better approach is to check directly the Pillar values sent from the master to the minions.

To that end, `salt '*' pillar.get <<PILLAR KEY>>` has a gotcha: it will come empty until you do `salt '*' saltutil.refresh_pillar`. Same for `salt '*' pillar.raw`. On the other hand, `salt '*' pillar.data` does not require refreshing it beforehand. However, I felt that the usability of `pillar.data` to be poor given that it outputs all pillars, requiring you to do extra processing to filter only the desired pillar key.

# Gotcha 3: Salt swallowing some errors

Most of the exceptions thrown during the execution of the renderer could be seen at `/var/logs/salt/master`, which made them easy to debug. However, when using the same renderer in another machine, the pillar would just not generate any values and also show no errors. It took me a long time to figure out that Salt was swallowing an error on the renderer, given the incosistent behavior. The issue was with a `import` statement because of a missing dependency.

The gotcha here to avoid salt swallowing the error is to use the `-ldebug` flag and specifically ask salt to only render the pillar data:

```
salt-call --local --ldebug slsutil.renderer /path/to/pillar.sls
```

# Conclusion

Salt seems to be a very interesting technology, and I hope this helps you avoid some of these issues :).
