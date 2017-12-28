---
layout: post
title: Why so many merge conflicts when I rebase?!
---

> Why am I getting so many merge conflicts when I rebase? This should be so simple!<br>
> &mdash; Colleague having a bad time

I frequently find myself fixing other people's Git woes but this one was particularly messy.

If we bring it back to basics, nothing seemed irregular. My buddy created his branch off of
master only a few days prior. He spent a couple days implementing a new feature and prepared
to tidy his history ready for code review.

Wham! Massive. Merge. Hell.

Now the size of the work was reasonably large. Not _I've changed everything from spaces to tabs_ large 
(although given the chance he would!) but a good 60+ files and a few hundred lines of code large.

As a part of my _please leave the place tider than you found it_ policy, I ask my team to squash
irrelevant Git history, leaving either a single commit or few relevant commits; whichever makes for
easier unpicking in case of a problem. This is why I'm called over.

So we start the diagnosis and sure enough, the merge is hell.

For reference we started with:

```
$ git rebase master -i
```

Problematic symptom number 1: not all of the code seems to be there. This means every squashed
commit being played is causing us a tonne of conflicts.

We quickly noticed that the interactive rebase hasn't included the base commit. Strange. Well
there isn't many commits to look at so picking the last 5 should do the trick...

```
$ git rebase HEAD~5
```

<error>
    <small><strong>ERROR</strong></small>
    <p>Commit abc123 is a merge but no -m option was given.</p>
</error>

Interesting. Looking through the branch history highlighted the base commit was a merge commit.

Attempting to interactively rebase on to a preserved merge commit is messing with Git dragons which
don't want to be messed with.

In fact, this is from the documentation:

> -p
>
> --preserve-merges
>
> Recreate merge commits instead of flattening the history by replaying commits a merge commit introduces. Merge conflict resolutions or manual amendments to merge commits are not preserved.
>
> This uses the --interactive machinery internally, but combining it with the --interactive option explicitly is generally not a good idea unless you know what you are doing.

There's a far less complex way achieving the same end result.

We start by finding the common ancestor of our two branches. (`my_feature` and `master`).

```
$ git merge-base my_feature master
1c5908a2392d758c805c1ebac6707b7490cc615e
```

Now we unstage all our changes between our `HEAD` commit and our common ancestor commit.

```
$ git checkout my_feature
$ git reset --soft <common ancestor commit hash>
```

Doing this will now show all your changes as unstaged on top of the common ancestor.

```
$ git commit -m "We did it!"
$ git rebase master
$ git push -f
```

Phew!

&mdash; Dan