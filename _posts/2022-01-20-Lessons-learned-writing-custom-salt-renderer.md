---
layout: post
title: Lessons learned writing custom Salt renderer.
---

I had to write a custom [SaltStack](https://saltproject.io/) renderer, and while the [concept](https://docs.saltproject.io/en/latest/ref/renderers/index.html) sounded simple, it was much more painful than I expected. I ran into several "gotchas" as a beginner, so I list them here for someone else in similar situation, or even myself in the future.

For consistency purposes, I changed all references to the custom renderer to be called "foo", as if I were adding `salt://_renderes/foo.py`.

# Gotcha 1: Renderer is not available

The first issue I faced was with this error on the pillar trying to use the new renderer:

```
The renderer "yaml | foo" is not available
```

This error was happening even though I would follow the instructions to use any of the commands below:

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

# Gotcha 2: Renderer is not available
