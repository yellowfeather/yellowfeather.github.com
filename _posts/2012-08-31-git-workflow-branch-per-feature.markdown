---
title: Git Workflow – Branch per feature
date: 2012-08-31 00:00:00 Z
categories:
- git
layout: post
comments: true
alias: "/2012/08/git-workflow-branch-per-feature/"
author: chrisr
---

Description of the git workflow and a branch per feature, extracted from: <a href="http://railsapps.github.com/tutorial-rails-prelaunch-signup.html">http://railsapps.github.com/tutorial-rails-prelaunch-signup.html</a>

When you are using git for version control, you can commit every time you save a file, even for the tiniest typo fixes. If only you will ever see your git commits, no one will care. But if you are working on a team, either commercially or as part of an open source project, you will drive your fellow programmers crazy if they try to follow your work and see such “granular” commits. Instead, get in the habit of creating a git branch each time you begin work to implement a feature. When your new feature is complete, merge the branch and “squash” the commits so your comrades see just one commit for the entire feature.

Create a new git branch for this feature:
{% highlight sh %}
git checkout -b new-feature
{% endhighlight %}

The command creates a new branch named “new-feature” and switches to it, analogous to copying all your files to a new directory and moving to work in the new directory (though that is not really what happens with git).

Commit changes to branch e.g.
{% highlight sh %}
git add .
git commit -am "Implements 'new-feature' feature"
{% endhighlight %}

Once the new feature is complete, merge the working branch to “master” and squash the commits so you have just one commit for the entire feature:
{% highlight sh %}
git checkout master
git merge --squash new-feature
git commit -am "Implements 'new-feature' feature"
{% endhighlight %}

You can delete the working branch when you’re done:
{% highlight sh %}
git branch -D new-feature</pre>
{% endhighlight %}
