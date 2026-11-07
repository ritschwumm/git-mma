# git-mma

git mixed martial arts: some git-fu, some bash-fu, and very much inspired by jujutsu.

this repo contains a collection of powertools built on top and around git.

## installation

1. copy aliases from [.gitconfig](config/gitconfig) into `$HOME/.gitconfig`
1. copy all scripts in [bin/](bin/) into a directory in your `$PATH`, maybe `$HOME/bin/`
1. optionally, copy the hook [hooks/pre-push](hooks/pre-push) into some `$REPO/.git/hooks/pre-push`

## features

### git sequence

the git subcommands `git cherry-pick`, `git merge`, `git revert`, `git am`, `git rebase`, and `git restack`, can "get stuck" and require the user to resolve some issue. afterwards, the user has to call the same subcommand with a flag `--continue`, `--skip`, `--abort` and `--quit`.

this is difficult to commit to muscle memory -- even when using git aliases, there have to be a lot of them. there is a better solution, though:

[`git sequence`](bin/git-sequence) can figure out which git subcommand is currently stuck, and call *this* command with the `--continue`, `--skip`, `--abort` or `--quit` flag. with some additional git aliases, the situation improves a lot:

| stuck command		| you type			| [.gitconfig](config/gitconfig) alias	| executed by [`git sequence`](bin/git-sequence)
|-					|-					|-										|-
| `git rebase`		| `git continue`	| `git sequence --continue`				| `git rebase --continue`
| `git rebase`		| `git skip`		| `git sequence --skip`					| `git rebase --skip`
| `git rebase`		| `git abort`		| `git sequence --abort`				| `git rebase --abort`
| `git rebase`		| `git quit`		| `git sequence --quit`					| `git rebase --quit`

| stuck command		| you type			| [.gitconfig](config/gitconfig) alias	| executed by [`git sequence`](bin/git-sequence)
|-					|-					|-										|-
| `git cherry-pick`	| `git continue`	| `git sequence --continue`				| `git cherry-pick --continue`
| `git merge`		| `git continue`	| `git sequence --continue`				| `git merge --acontinue`
| `git revert`		| `git continue`	| `git sequence --continue`				| `git revert --continue`
| `git am`			| `git continue`	| `git sequence --continue`				| `git am --continue`
| `git rebase`		| `git continue`	| `git sequence --continue`				| `git rebase --continue`
| `git restack`		| `git continue`	| `git sequence --continue`				| `git restack --continue`

### stacked branch management

as [jujutsu](https://www.jj-vcs.dev/) and many other tools have shown, stacked branches and a rebase workflow are powerful.

of the two main features of jujutsu, namely first-class conflicts and automatic rebasing, at least the latter can be emulated in git quite well.

#### git restack

[`git restack`](bin/git-restack) rebases a whole tree of branches, while maintaining their connections. again, a few git aliases improve the experience even more.

| you type						| [.gitconfig](config/gitconfig) alias	| what it does
|-								|-										|-
| `git on master devel`			| `git restack master devel`			| restacks all descendants of "devel" onto of "master".
| `git onh feature`				| `git restack HEAD feature`			| restacks the "feature" branch onto HEAD.
| `git restack -h`				| n.a.									| show all options
| `git restack --completion`	| n.a.									| prints bash code which enables bash completion when eval'ed

#### git ox

built on this foundation, [`git ox`](bin/git-ox) provides additional bits and pieces of a jujutsu-like workflow. of course there are short aliases for some of it.

| you type						| [.gitconfig](config/gitconfig) alias	| what it does
|-								|-										|-
| `git rox`						| `git ox reword`						| changes the commit comment of HEAD and restacks all descendants onto the replacement commit.
| `git rox e29ad`				| `git ox reword e29ad`					| changes the commit comment of e29ad and restacks all descendants onto the replacement commit.
| `git fox`						| `git ox amemd`						| amends the current HEAD commit and restacks all descendants onto replacement commit.
| `git dox`						| `git ox delete`						| removes the HEAD commit and restacks all descendants onto the parent commit.
| `git dox e29ad`				| `git ox delete e29ad`					| removes the commit e29ad and restacks all descendants onto the parent commit.
| `git ox -h`					| n.a.									| show all options
| `git ox --completion`			| n.a.									| prints bash code which enables bash completion when eval'ed

### commit-based navigation

especially when using stacked branches, it's often very useful to check out naked commits (as opposed to branches). in this setting it's helpful to be able to navigate back and forth in the commit history, and later re-attach the HEAD to a named branch.

two git aliases and two new subcommands, [`git advance`](bin/git-advance) and [`git attach`](bin/git-attach), make this a breeze:

| you type		| [.gitconfig](config/gitconfig) alias	| what it does
|-				|-										|-
| `git fwd`		| `git advance`							| checks out the next child commit (if it is unambiguous).
| `git back`	| `git checkout HEAD~`					| checks out the parent commit.
| `git attach`	| n.a.									| checks out the branch on top of the current commit (if it is unambiguous).

a real-life example:
this tests the last 5 commits one by one. if tests fail, it stops on this commit.
if test passed, though, it checks out the child of the current commit and repeats running tests.
if there is no unique child commit, `git fwd` fails and the loop ends, too.

```bash
git checkout HEAD~5
while npm test && git fwd; do true; done
```

## minimal task management

sometimes when working on a larger feature it's necessary to take notes, e.g. to not forget some vital *other* change that has to be done later.
putting these notes in a separate file is tedious and makes them disconnected from any branches/commits.

a very simple and unexpectedly useful method to deal with this is to put such notes into the comment of an otherwise empty commit. as an empty commit would possibly not survive a rebase, but adding a uniquely named empty file in `$REPO/.todo/` works just fine.

this is exactly what [`git todo`](bin/git-todo) does:

```bash
$ git "todo there are still no tests"
```
will create a todo-commit with the comment `[todo] there are still no tests`.

[`git todo`](bin/git-todo) supports an additional shortcut: if called with more than one parameter, all parameters are joined with a space and used together for the commit comment. this makes quoting is mostly unnecessary: shell-metacharacters still have to be quoted, of course.

```bash
$ git todo there are still no tests
```
does not quote the comment, but will still create a todo-commit with the same comment `[todo] there are still no tests` as the line above does.

if the todo has been resolved, the todo-commit can be deleted easily with `git dox <commitish>`.

### temporary branches

in my experience, temporary branches are a highly underrated tool. most of the time, a single temporary branch is enough, though -- but *this* branch is created an deleted a whole lot of times.

a small script named  [`git tmp`](bin/git-tmp) make this a lot quicker:

| you type			| what it does
|-					|-
| `git tmp`			| creates a new branch "tmp" at or moves it to the current HEAD.
| `git tmp e29ad`	| creates a new branch "tmp" at or moves it to the commit e29ad.
| `git tmp -d`		| deletes the branch "tmp".

to prevent accidententally pushing this "tmp" branch to a remote, a git hook [hooks/pre-push](hooks/pre-push) is provided.
