---
layout: post
title: Testing your Cookbook
tags: LaTeX Cooking TeXtidote Unit-Tests
comments: true
---

Just this week I moved my LaTeX based cookbook to github and added *Continuous Deployment* with a Github Actions workflow (see [here](https://huserben.github.io/Cookbook-as-Code/) for the full story). Now that I have it properly versioned, I wanted to refactor it a bit so that it's not stored in one huge file, but each recipe in its own file.  
Normally when we refactor we're having a safety net in form of unit tests that make sure we're not breaking something. This got me thinking whether this would be possible for the cookbook as well.

In this post I'll explain how I managed to introduce some checks using the [TeXtidote](https://github.com/sylvainhalle/textidote) linter and include it in the github action workflow.

All code is merged into the main branch of the [github repo](https://github.com/huserben/cookbook).

# Making sure LaTeX still compiles
The first and arguable the most important check is that the LaTeX sources still compile and a proper PDF can be built from it. This is already part of the workflow by building the pdf as part of the workflow:  

```
      - name: Create PDFs
        uses: xu-cheng/texlive-action/full@v1
        with:
          run: |
            apk add curl
            curl -OL https://mirrors.ctan.org/fonts/emerald.zip
            unzip emerald.zip
            cp -a ./emerald/. /opt/texlive/texmf-local/
            mktexlsr
            updmap-sys --force --enable Map=emerald.map
            mktexlsr
            pdflatex -synctex=1 -interaction=nonstopmode "./MyCookbook".tex
            pdflatex -synctex=1 -interaction=nonstopmode "./MyCookbook".tex
```

So if I changed something that make it break and did not check it locally (shame on me), the workflow will fail after a push to the main branch.
Ok, but it's not ideal that we can push to main and only discover that it broke after the fact.

## Only allow changes via PR
In order to prevent direct pushes to the main branch, we can add a branch protection rule. This will make sure we don't allow direct pushes to main but instead we have to go via a PR. Moreover we can define that the status checks (the action workflow) must pass successfully before we can merge the PR:

![Branch Protection]({{ site.baseurl }}/images/posts/testing_cookbook/branch_protection_rule.png)

## Run Workflow on PR
With a small extension to the workflow file we can tell github to automatically run the workflow whenever we have a PR to the main branch:

``` yaml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```

## Testing it out
So let's try whether this works by randomly adjusting some file that should break the compilation:

![Breaking Change]({{ site.baseurl }}/images/posts/testing_cookbook/breaking_change.png)

We can see that the PR will automatically run the workflow:  

![Automatically triggered workflow]({{ site.baseurl }}/images/posts/testing_cookbook/auto_workflow.png)

And after waiting a bit we should have our failing build - No merge for you!
![Failed Build]({{ site.baseurl }}/images/posts/testing_cookbook/failed_build.png)

Ok that adds some peace of mind for the next refactor. But let's see if we can do better.

# Automated Spell Checking
I'm good at many things. But spelling and grammar are not on that list. So I really appreciate if I would have something tell me when I made some typos again. After a quick online search I came across a tool called [TeXtidote](https://github.com/sylvainhalle/textidote). It can analyze LaTeX files and give feedback to spelling, grammar and general styling rules. It even supports different languages, which is good because my cookbook is in german :-)  

## Installing TeXtidote
To install it I just followed the [instructions](https://github.com/sylvainhalle/textidote#getting-textidote) on the repo.  

As TeXtidote requires java, make sure you have that installed:  `sudo apt install default-jdk`.  
Then download and install the latest version:  

``` bash
          curl -OL https://github.com/sylvainhalle/textidote/releases/download/v0.8.2/textidote_0.8.2_all.deb
          sudo apt-get install ./textidote_0.8.2_all.deb
```

After successfull install we should be able to run `textidote --version`:  

``` bash
TeXtidote v0.8.2 - A linter for LaTeX documents and others
(C) 2018-2021 Sylvain Hallé - All rights reserved
```

## Running Linter
No time to check any configuration, let's just try to run it and see what happens:  
``` bash
admin@DESKTOP-AFV28V8:/mnt/d/git/cookbook$ textidote --output html MyCookbook.tex > report.html
TeXtidote v0.8.2 - A linter for LaTeX documents and others
(C) 2018-2021 Sylvain Hallé - All rights reserved

Found 669 warning(s)
Total analysis time: 0 second(s)
```

Well that's **a lot** of warnings...Let's check the generated report:

![Report]({{ site.baseurl }}/images/posts/testing_cookbook/report.png)

A report with syntax highlighting, awesome!
But it looks like it finds stuff I don't care too much about, so maybe check the config options anway.

## Configuring TeXtidote
There are a bunch of configuration options for TeXtidote. So let's see how we want to configure it for my use case.

### Add Spell Checking
Spell checking is actually off by default. We can add it by passing `--check de_CH` for checking the Swiss version of german (arguable the best version of german).  

For other supported languages check the docs.

#### Adding Custom Words
As dictionaries are incomplete you can also specify words to exclude from the spell check. Just create a text file and add each word you don't want to trigger a warning on a new line:  

`--dict TeXtidote/ignore_words.txt`

### Ignore certain Rules
Some things I don't mind so much (e.g. the short sections) so I just want to ignore that rule. This can be done with the ignore switch followed by all the rules that we want to ignore. For the section lenght this would look like this:  

`--ignore sh:seclen`

### Deal with False Positives
We might also end up with some rules that we don't want to exclude, but it triggers because it cannot properly identify some parts. We can solve this problem by *find and replace* rules defined in a file.

For example we define the path to an image for the recipe, but it often triggers errors, so we want to replace this with an empty block:

```
# Empty lines beginning with a pound sign are ignored
# Search and replace patterns are separated by a tab
find	replace
(small)(.*)(.png)	 
(big)(.*)(.png)	 
```

**Note:** We have a tab followed by a whitepsace (what to replace it with).

Then we can just tell TeXidote what file to use: `--replace TeXtidote/replacements.txt`.

### Output
Last but not least we want to specify the output. The html report is nice to initially fix the problems, but in the end we want to run it on the commandline and get an error back to fail the build if needed.

We can do this with `--output singleline`.

### Create config File
A neat way of configuring is to create a *.textidote* file that includes all the configuration. That way you can just run `textidote` in the same directory and the options will be picked up.

Our file will look like this:

```
--output singleline
--dict TeXtidote/ignore_words.txt
--ignore sh:nobreak,sh:d:002,sh:nonp,sh:seclen,lt:de:BISSTRICH,lt:de:COMMA_PARENTHESIS_WHITESPACE,lt:de:DE_CASE,lt:de:PUNCTUATION_PARAGRAPH_END
--replace TeXtidote/replacements.txt
--check de_CH
MyCookbook.tex
```

## Testing It
Ok now let's run that:  

``` bash
admin@DESKTOP-AFV28V8:/mnt/d/git/cookbook$ textidote
TeXtidote v0.8.2 - A linter for LaTeX documents and others
(C) 2018-2021 Sylvain Hallé - All rights reserved

Using replacement file TeXtidote/replacements.txt
Found 1 warning(s)
Total analysis time: 10 second(s)

Sections/Mains/baked_mac_and_cheese.tex(L10C22-L10C22): Möglicher Rechtschreibfehler gefunden. Suggestions: [Ches, These, Chose, Diese, Phase, Achse, Thesen, Chemie, Wiese, Chile, Riese, Chelsea, Schere, Chöre, Themse, Scheue, Echse, Chefs, Lese, Kiese] (154) "   {Baked Macaroni and Chese}"
```

Ok apparently I mispelled *Cheese* there. Let me fix it and try again:  

``` bash
admin@DESKTOP-AFV28V8:/mnt/d/git/cookbook$ textidote
TeXtidote v0.8.2 - A linter for LaTeX documents and others
(C) 2018-2021 Sylvain Hallé - All rights reserved

Using replacement file TeXtidote/replacements.txt
Found 0 warning(s)
Total analysis time: 10 second(s)
```

That works. Now we just have to run it as part of our push.

# Running TeXtidote within Github Action Workflow
We have our existing workflow that we can extend. What we must do is to download and install TeXtidote and then running it.

## Install TeXtidote
Installing is the same as locally, we just need to download it from the github release page and then installing it using `apt-get`. Java is probably already available on the agent that runs the workflow so we can skip that for now:  

``` yaml
      - name: Install TeXtidote
        run: |
          curl -OL https://github.com/sylvainhalle/textidote/releases/download/v0.8.2/textidote_0.8.2_all.deb
          sudo apt-get install ./textidote_0.8.2_all.deb
```

## Run TeXtidote
Once it's installed it could not be simpler to run it, as we have all our config already in the *.textidote* file we just need to run *textidote*:  

``` yaml
      - name: Run TeXtidote
        run: |
          textidote
```

And that should do the trick. Let's try it out!

## Testing the Workflow
Let's start with the failing test. I create a PR with the same spelling mistake as before and expect it to fail:  

![Spelling Mistake]({{ site.baseurl }}/images/posts/testing_cookbook/spelling_mistake.png)

**Success!**

So now let's fix it and make sure it's happy when there are no errors:

![No Spelling Mistake]({{ site.baseurl }}/images/posts/testing_cookbook/no_spelling_mistake.png)

# Conclusion
So with these checks in place I can happily change the structure of my cookbook. Even when I *just* add something new I know that the build will tell me about my typos (and there will be typos).  

Once we have something under version control we should start thinking about how we can make sure that it stays in a good shape. Whether this is an enterprise application or a cookbook, automation is your friend.