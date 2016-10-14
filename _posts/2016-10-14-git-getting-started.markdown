---
layout: post
title:  "Git - Getting Started"
date:   2016-10-14 18:18:41 -0400
categories: ubuntu git source-control
---
This tutorial is quite basic/for beginners, but a good start to learn how to leverage the git source
control system. It also speaks briefly about the differences between git and svn. Commands contained
within are mostly for reference purposes.

### Git (And SVN)

Anyone writing software typically utilizes a source control system. The most commonly known system in
use today (at least within "Agile" environments and/or the DevOps universe) is git.

Many organizations still use a system known as svn. This system has fast gone by the wayside in favor
of the git source control system due to svn's inflexibility and bloat. One of the primary reasons
people today utilize git is due to the distributed nature of the source control system. In svn, there
exists a central repository which contains all source code/commits/branches/etc. If a developer wishes
to work with the code base for a particular product, they must essentially create an entire clone of
the central system, and commits are pushed upstream, causing slowness. In addition, in the Agile model,
commit early, commit frequently is very much a principal that is followed. Creating branches for feature
requests and bug fixes dovetails nicely into this concept, which is very slow to manage for source code
that is of any substantial size within svn due to the fact that creating a branch in svn creates an exact
replica/copy of the "master" branch (or whatever source branch you specify). This can quickly cause your
development cycle to slow significantly.

In contrast, git was developed to address several of the above deficiencies of svn. First, git is a
distributed source control system. This means that once you have checked out the code, you can develop
entirely local to your environment/laptop/vm/machine without needing to communicate with any 'centralized'
server or software application. This is incredibly advantageous in that development is quick. In addition,
everything within git follows a node graph. This means that diferences are tracked between commits, not
the entire state of the files at any given time (snapshot). This 'difference' approach allows for a very
fast and lightweight source code management system that allows re-creating the code base by applying
'diffs' until you arrive at the commit you wish to retrieve (think 'differences between commits', not
'Save As' for each commit you make).

All development and commits are made locally - now, how does one 'publish' their changes or feature
branch? This is accomplished via 'pushing' your commits or branch. Typically, once your feature is
complete (your feature branch contains all commits corresponding to the feature and any/all tests pass),
you perform what is known as a 'merge' locally on your development instance. The 'master' branch is
typically your golden copy of the code base that should represent your production environment at any
given time. Due to this, you would typically merge your feature branch into the master branch, resolve
any conflicts that arise (in case other developers have merged changes while you were developing), run
a full regression suite of tests (if you are so lucky to have them), and then 'push' your master branch
'upstream'. The 'upstream' notion in this sense is the master branch that exists in your central git
server (the one that should represent what your production environment is currently running). Note that
this 'central server' slightly contradicts my previous statement re: no central server/service for git.
However, the central server is not consulted/communicated with until and unless you either need new
commits/branches that others have worked on, or you yourself wish to publish changes that you have made.
Think of it as more of a messenger of sorts, along with being a safe for your production/gold copy code
base.

### Prerequisites

The following commands/tutorial assume that you have git installed/in your PATH variable, you are using
some kind of *nix system, are operating on the command line, and that you have a GitHub account or some
other privately-hosted repository available. Setting up these components is outside the scope of this
tutorial.

### Git Primer

Now that the background is out of the way - let's get to some simple commands to work with. First, you
will want to ensure that git is installed and available in your path (note that these instructions
assume that you are on a *nix system). Note that there is a glossary of terms and definitions towards
the bottom of this article if you are unfamiliar with any of the terms used:

{% highlight bash %}
$ which git
# should output something like '/usr/bin/git' - if not, ensure that git is installed and it is
# also available in your PATH variable
{% endhighlight %}

To initialize an empty repository (get started with git), create a new project directory, and run the
init command:

{% highlight bash %}
$ mkdir my-git-tutorial
$ cd my-git-tutorial/
$ git init
{% endhighlight %}

You may be prompted to initialize your configuration settings as well (name, email, etc). If this is the
case, ensure that you do so as this information will be the data contained in each commit that you
perform.

Now that you have an empty git repository, create a README.md file and add some content to it. This file
is typically the entry point for anyone interested in your project - it should contain instructions on
how to setup/install/configure the software you are developing, any notes for the developer or user,
information on how to contribute, etc. See the GitHub (official) tutorial for more information about
this file.

Now that you have created a starter file, add it to the staged changes for commit. This is a point that
requires some explanation. In git, when you are ready to make commits (commit your changes), you do so
by first 'staging' the changes you've made (making them available to the next commit performed). If you
type the command `git status` in the project directory you created your README.md file in, you should see
the README.md file listed under the 'Untracked files' section. This means that there has been a change
(file added, in this case), but that git has not been told to stage this change as part of the next commit.
To do so, use the `git add` command:

{% highlight bash %}
$ git status
# should show README.md under the 'Untracked files' section
$ git add README.md
$ git status
# should now show the README.md under the 'Changes to be committed' section
{% endhighlight %}

Your README.md file is now ready to be committed. When you perform a commit operation, this tells git
that you wish to officially track the change corresponding to the README file (adding the file to the
repository). When committing, it is best practice to include a short message related to the subject of
the commit being made. There are conventions for how this message should be structured, how many
characters should be used, etc. - a simple Google search should turn these results up quickly.

Performing a commit operation can be done in such a way that the commit message is given on the command
line (short message). More experienced developers utilize text editors with fancy formatting to help
them conform to whatever convention has been established for commit messages. To do this (for instance, if
you wish to utilize vim whenever you perform a commit), ensure that your `EDITOR` variable is set to the
application you wish to use (for instance, `export EDITOR=vim`). Doing this ensure that whenever you type
the command `git commit` and press enter, the editor will pop up allowing you to type your commit message.

{% highlight bash %}
$ git commit -m "Initial Commit"
# this is your first commit - very simple, message include on the command line
{% endhighlight %}

If all goes well, your changes should be committed. Performing a `git status` operation will show you that
there are no changes to be tracked. You can check the 'graph' of your commits via the log operation:

{% highlight bash %}
$ git log
# should show your last commit, including information about the author/committer
{% endhighlight %}

Now that you have performed a commit, if you wish to push your changes upstream, use the push command. Note
that again, this assumes that you have already established an 'upstream' and configured your project to
utilize it.

{% highlight bash %}
$ git push origin master
# push the current 'master' branch changes to the upstream repository
{% endhighlight %}

Over time, commits will be made both by you and (hopefully) other developers - to obtain changes that other
developers have made and pushed to the central master branch, perform a pull operation:

{% highlight bash %}
$ git pull
# retrieves any changes from the upstream repository for the current branch being worked on
{% endhighlight %}

Following the above steps is typical for development. There are many more advanced operations that can be
done (listed in the below sections), and over time, you should become familiar with them as they will
certainly make your life easier.

### Advanced

There are many slightly more complicated ways to interact with git that helps with development patterns.
Below are some of those - they are not explained in any detail but are provided more as a reference for
research direction.

Useful git commands:

{% highlight bash %}
$ git log --graph --format='%h %Cred %ar %Cgreen %s %Cblue %an'
# provides a nice colorful visual graph of your commits and merges/changes

$ git rebase <branch_name>
# re-establish the base commits of your current branch using the named branch - adding '--interactive'
# allows a per-commit traverse in case you are worried about the commits being added

$ git merge <branch_name>
# merge the branch specified into the current branch - adding '--no-ff' ensures that all commits made
# within the incoming branch are laid on top of the current branch commits (not interwoven based on
# the date/time the commits were made)

$ git reset --soft HEAD~
# undo the last commit made, but leave all the changes from said commit as 'staged' - this is useful if
# you want to undo a change in the latest commit

$ git reset --soft HEAD~2
# same as previous soft reset, but go back 2 commits instead of just 1

$ git reset --hard HEAD~
# destructively undo the latest commit - this is dangerous, and you should know what you are doing if you
# wish to use this command

$ git push origin master --force
# force-push any local changes on the master branch to the upstream master - this is EXTREMELY dangerous
# and in general, a no-no for collaborative development as it will require all other developers to
# re-establish their node graph if there were destructive changes made

$ git config --list
# list the known configurations for your git profile (username, email, etc)
{% endhighlight %}

One last thing to note - placing a file named `.gitconfig` in your home directory with configuration
directives (see the official git documentation) allows for a more customized experience. It allows you to
not only set up your profile information (author information), but also allows for shortcuts/aliases
through git.

### Definitions

* Repository: Directory/other containing source code being tracked by git.
* Distributed: All changes/diffs known locally, and modifications can be made entirely locally as well.
* Staged Change: Updates/changes that you wish git to record during the next commit operation.
* Commit: Record any/all staged changes recorded.
* Reset: Revert a commit/set of commits.
* Branch: current graph being worked on - typically used for features, bug fixes, etc.
* Master: 'gold copy' branch - this should most likely reflect what you have in production.
* Upstream: the 'central' repsository where changes are published to for other developers/endpoints to consume.
* HEAD: Term used to describe the latest commit made/known to git.
* Node: In this context, a commit made within git (as in, 'node graph theory').
* Merge: Joining of one branch into another.
* Rebase: Manipulating the graph of nodes (commits) to be in line with another branch.
