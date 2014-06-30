---
layout: post
title: "Pre-commit Hooks: The Good, The Bad &amp; The Ugly"
description: "My thoughts of using and creating a good pre-commit hook"
modified: 2014-06-29
category: articles
tags: [post]
comments: true
share: true
---

# Introduction

I know it has been a while since I put the page up but I've been busy. I've been preparing for a big interview, working full time at Wave and taking a class at night twice a week. So I decided to take an hour and write up a quick post about something I've been working on recently (hopefully it isn't too badly written). Since I've been working on my pre-commit hook at work, I'll write a bit about my thoughts on them. I hope by taking the time to write this, I can get in the groove of writing more often.

# The Good

The good thing about pre-commit hooks is that you can "stop" yourself, and team members, from pushing code that doesn't pass through various checks. You can't really stop this because they can force push or not use the hook, but you can discourage it. A lot of these checks tend to be running the changed files through a linter and a handful of custom scripts. I've worked quite a bit with Django so I'll be using some examples from my experiences within the post but these could apply to any language.

A simple example of a custom script I plan to write, for working with Django, is to search for any url in any git added html files. Any local links or assets that use direct linking should be caught by the hook. This is bad practice and can lead to a headache later on if the static directory changes name or the URL of a page changes. I've run into this in the past, where I wanted to make the URLs of a website more friendly and uniform. The old templates had direct links to pages throughout and as a result, I had to fix them by hand. I named each url and used Django's url tag instead. This way if we decide to change the url of one of our pages we could simply change the one line in the urls.py file. After doing that, each link to that page will be generated on the fly using the tag. I would also like to add another check that looks at any file name that does not end in 'thirdparty.html' and not allow any direct links at all. The rest could be stored in a context processor so that there is one place for all outside links that don't belong to third party code.

# The Bad

So far I've only touched on what I like about pre-commit hooks but there is also a bad side. It's easy to fall into a trap where developers ignore pre-commit hook results or treated them as 'good to merge' test. The former tends to happen when using pre-commit hooks with legacy code which does not meet the requirements of new hooks. In that situation the team has to make an active decision to fix anything found by the script in order for it to be effective. This means making legacy code match the new standards every time you alter an older file. This can be annoying when a one line change nets fixing hundreds of linting, indentation and whitespace errors. This can lead to skipping the pre-commit hook or ignoring its output if not agreed upon by the team. The opposing effect is that a developer may view pre-commit hook results as being much more valuable than it really is. It's possible for there to be holes or missing checks in the hook. This is especially true for code design standards that are not community standards. The take away in this case should be to treat it as one step in a checklist before merging code, not the checklist itself.

# The Ugly

Now to get down to an actual example. I'll be including a part of the pre-commit hook I use at work, with anything Wave specific removed:
<script src="https://gist.github.com/JBKahn/34362b12152f3c79555c.js"></script>

The above hook can be broken down into several sections:

1. Check for duplicate south migration files. This is used to avoid merging branches into master with the same migration number as an existing migration. This would make it harder to rollback code and can happen to long running branches.
2. Run python files through flake8 for style and simple linting checks (i,e. unnecessary imports) as well as print statement and debug statement checkers.
3. Run coffee script and javascript files through jshint and coffee-jshint linters.
4. Run JSON files through jsonlint, good for node, grunt and bower.

This is a work in progress as I hope to continuously improve this script as I find more things to lint and improvements for this script.

Things I'd like to do:

* Use the cached version of the files added that git diff can output. I would pass these to the linters instead so that I can lint only the copies staged for commit and not the local copy in the folder.
* Work on a 'jshintrc' file so that the js linter will not use the default settings and will only do the checks I want it to do.
* CSS/LESS linting; I'm uncertain it's of huge value but there are a few things I might add custom scripts to check for.
*  The URLs and the direct links in the html files of Django projects.
