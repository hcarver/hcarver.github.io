---
layout: post
title: Git is not great
date: 2013-11-14 19:15:04.000000000 +00:00
categories:
- development
tags:
- distributed version control
- git
- software development
- subversion
- version control
status: publish
type: post
published: true
author: Hywel Carver
---
Git is better than what we had before but it's not great.
[Download my pdf cheatsheet](/assets/cheatsheet.pdf) for free if you want a clear guide to git that thinks the way you do.

Git is a modern, distributed version control system, replacing older, centralized systems like Subversion. It has emerged as the de facto answer to storing historical information about changes to files, at least among software developers.

Git is powerful and it's a lot better than Subversion. But it's not great. I think that statement is objectively true for three important usability reasons.

## The help text is terrible

**Good software helps the user to feel in control**. Git doesn't. Let's look at a relatively simple command:

`git add <files>`

This is the command you use to tell git that you want to include certain files in the next batch of work you commit. Let's look at the help-text description:

> This command updates the index using the current content found in the working tree, to prepare the content staged for the next commit.

Do you fully understand what this is saying? I do, but then I've spent the last day writing a git cheatsheet. Using the term 'commit' is forgivable, as it's totally standard in version control systems. But index? working tree? staged? These are all git-specific and go unexplained. I've been using git for two years and I finally learned what the index was last week, yet that understanding is required by the fifth word of the help-text for one of the most basic functions of git.

## Git does not forgive

**Good software lets you explore, make mistakes and undo them**. Git doesn't. Often, there is a way of undoing things, but it is so hidden and so arcane that you feel like you're in the Da Vinci Code.

Let's say you do an accidental git add to a file. How do you undo it? `git add-undo <file>`? `git undo add <file>`? Or maybe `git undo` undoes your last action? One nerd point to anyone who guessed at `git reset HEAD <file>`. Notice that this command doesn't look like it has anything to do with git add or with undoing.

OK, now you've accidentally done `git rm` on a file, and it's disappeared. How do you undo it? Pat yourself on the back if you think that `git checkout <file>` is an obvious solution. (Side-groan: this only counts as checking out the file if you think about how git works, not the mental model of the user).

Whoops, you've accidentally committed when you didn't mean to. Git provides the very nice `git commit --amend` to add some other changes to the commit. But if you didn't mean to commit at all, what do you do? 10 points from Gryffindor unless you said `git reset –soft “HEAD^”`. Again, notice that this command doesn't obviously relate to the concepts of committing or undoing.

My absolute favourite example of unforgiving user interface is this: go type `git checkout "HEAD^"` in any git repository. This turns git into a headless chicken:

> You are in 'detached HEAD' state. You can look around, make experimental changes and commit them, and you can discard any commits you make in this state without impacting any branches by performing another checkout.
>
> If you want to create a new branch to retain commits you create, you may do so (now or later) by using -b with the checkout command again. Example: git checkout -b new_branch_name

Sorry, what? My HEAD is detached? And how do I undo that? Why have you detached my HEAD? And what does that mean? And how do I undo it? This output doesn't help with any of those questions. If you see this, you might well want to do `git checkout master` to reattach your HEAD, but git doesn't tell you that.

## Git is organised how it's made

**Good software hides the way it works so that the user can think about it in the most natural way**. Git doesn't. Let's take `git reset` as our example.

If you know how git works, you'll know that `git reset` is for changing where HEAD points to, and for optionally updating the index and working copy. Now forget that you know that, and think about the user's model of what each form of `git reset` is for.

- `git reset <commit> <paths>` sets the index for the paths to what they were at a certain commit. So the user wants to cherry-pick a file to commit as it was in a different commit.
- `git reset --soft <commit>` resets HEAD to a certain commit. So the user can change where in the revision graph their already-prepared commit will go.
- `git reset <commit>` resets HEAD and the index to a certain commit. So the user wants their current work to be compared and committed against a different point in the revision history (maybe you've fetched some new code).
- `git reset --hard <commit>` resets HEAD, the index and the working copy. The user's doing some work and wants to scrub their changes and start again.

The last form of these is the only one that represents what you might think of as a reset, and all four are used in different situations: change what will be committed, change where your commit appears in the revision history, undo all your current changes.

To use these requires thinking in the terms of how git works internally rather than in terms of what you, the user, are doing and what you want to achieve.

An even worse example of this bad organisation is deleting a branch on a remote. Remember, to delete a local branch you do `git branch -d <branch name>`. So how do you delete a branch from a remote? I think the obvious way would be `git branch -d <remote>/<branch name>`. The actual command is `git push <remote> :<branch name>`. From the user's point of view, push is a totally different operation to deleting a branch, but push is a better reflection of the internals of git, and that's the point of view that wins out. Also, note how small a typo it would take to delete a branch (`git push origin :master`) instead of pushing a branch (`git push origin master`).

## The Antidote

Try [my git cheatsheet](/assets/cheatsheet.pdf). This is a single printable side of A4 that has all the key commands for using git day-to-day. This cheatsheet is unique because it is organised thematically, grouping the commands by the situation in which you'd need them. This makes it very easy to find the commands you need.

Thanks to Tom Carver for pointing out the `git branch` example, and to Tom Carver and Catharine Robertson for proofreading the Git Cheatsheet.
