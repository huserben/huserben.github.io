---
layout: post
title: Cookbook as Code
tags: DevOps LaTeX Cooking
comments: true
---

I like to cook (and especially to bake stuff). Since several years I keep ~~my~~ the recipes that turn out to be good in my personal cookbook. This has some advantages for me:

- I don't need to search through a tens or even hundreds of links to the original recipe
- The small (or big) adjustements I make to the original recipes only really need to be done once (how much millilitres was a cup again?)

In this post I share how I store the recipes *as code* so they can be versioned in a version control system like git. By using LaTeX with a nice template you'll get a nice looking cookbook in pdf format. Moreover I show how to use Github Actions to automatically create pdfs whenever you push a change to github and how you then can publish it on the internet so you have your latest cookbook available on your device of choice every time.

In the end we'll have some nice looking pdf:
![Final Recipe]({{ site.baseurl }}/images/posts/cookbook_as_code/final_recipe.png)

# Preparation
We're using [LaTEX](https://en.wikipedia.org/wiki/LaTeX) to create the cookbook. This allows us to use plain text to describe the document and build nicely looking documents out of it.

## TeX Distribution
In order to get started with LaTeX locally you need a TeX distribution and an editor of your choice.
I'm using [TeXLive](https://www.tug.org/texlive/) which you can download and install from the [official website](https://www.tug.org/texlive/acquire-netinstall.html). Be aware that this might take a while and takes a considerable amount of space - on my machine it's over 7 GB.  

### Add TeXLive folder to PATH
To make everything works smoothly, it's best to add the TeXLive bin folder to your PATH variable. On my windows system where I installed TeXLive to *D:/texlive* the bin path is *D:/texlive/2021/bin/win32*

## Tex Editor
Next you need an editor. You can use anything from regular Notepad over VS Code to specific TeX Editors. I'm using [TeXstudio](https://www.texstudio.org/).

## Install Emerald Font
The package we're using for creating the cookbook is using a font that doesn't seem to be supported out of the box. Thus we need to install it manually. Do the following to install the *emerald font*:

1. [Download](https://ctan.org/pkg/emerald?lang=en) and extract it locally  
2. Move both the *Font* and *Tex* directory to your local TexLive install  (this is a folder called *texmf-local* within your install directory)
  2.1 You can find the exact folder by running `kpsewhich --var-value TEXMFLOCAL`  
3. Install the font the following way:  
  3.1 Open a command line in the font map directory (e.g. *D:/texlive/texmf-local/fonts/maps/dvips*)  
  3.2 Run the following:  
    ```
    mktexlsr
    updmap-sys --force --enable Map=emerald.map
    mktexlsr
    ```

Ok now you should be good to go.

# Clone Repo
For a quick start, you can clone my repo from [github](https://github.com/huserben/cookbook):  
`git clone https://github.com/huserben/cookbook.git`

Then you can open the file *MyCookbook.tex* in TeXstudio.
Hitting *F5* (or the big double arrow on top) it will compile and display the resulting pdf:

![Compiled Recipe]({{ site.baseurl }}/images/posts/cookbook_as_code/compile.png)

If something failed, make sure you installed everything including the font as above - it works on my machine (and on github actions - see below :-)).

# Cookbook Structure
Let's check out the structure of the document quickly.

## xcookybooky
The cookbook is making use of the package [xcookybook](https://ctan.org/pkg/xcookybooky?lang=en). A detailed description can be found on [https://mirror.foobar.to/CTAN/macros/latex/contrib/xcookybooky/xcookybooky.pdf](https://mirror.foobar.to/CTAN/macros/latex/contrib/xcookybooky/xcookybooky.pdf)

Roughly it has the following elements:
- Preamble
- Titlepage
- Sections
- Recipes

## Preamble
In here we define which packages we're using for the document as well as some settings. For example can you find here that we're relying on xcookybooky:  

```
    \usepackage[
        handwritten,
        nowarnings,
        %myconfig
    ]
    {xcookybooky}
```

## Titlepage
After the `\\begin{document}` element we define the title and the author and create a title page, followed by a auto-generated table of contents (`\\tableofcontents`).

## Sections
As you might have spotted, there are different sections (e.g. for Snacks, Main Courses or Deserts). A section will be like a heading under which the recipes are grouped under. The sections also appear in the table of contents.

## Recipe
Finally we get to the meat of the cookbook (pun intended). The recipe itself has several different parts, like the header with some general information about preparation time, the ingredients, preparation steps and if needed a hint section.

Let's check this out in more detail with a real example.

### Adding a new Recipe
When we want to add a new recipe, we go the place in the document where we want to add it. Then we want to add a new line to start the recipe on a fresh page and add begin a new recipe section:  

```
\newpage
\begin{recipe}
    {My New Recipe}
\end{recipe}
```

*My New Recipe* will be the title of the recipe. *Build and View* and you should see the new page appear.  

#### Switching the Language
If you're using my repo as a base you might have noticed that some text is shown in the beautiful german language. If you want to change this to english go to the preamble, search for `\\usepacakge[german]{babel}` and replace german with english - build again and now you should be good to go.

### Add Overview
Ok let's add some highlevel overview to the recipe. For this you need to add the following **right after** the begin block:  

```
    [
        preparationtime = {\unit[10]{min}},
        bakingtime={\unit[30]{min}},
        bakingtemperature={\protect\bakingtemperature{fanoven=\unit[200]{°C}}},
        portion = {\portion{4}},
        calory,
        source = https://github.com/huserben/cookbook
    ]
```

After a new build you should see the information appear below the title. However something essential is still missing: a cool preview of how our food will look like. Let's change this by adding the following after title:  

```
    \graph
    {
        small=pictures/amaretto_ginger_ale/amaretto_ginger_ale.png
    }
```

After a build you should see the image appear. You can also change between *big* and *small* when including a picture.

### Add Ingredients
Next we need to specify our ingredients, you can do this by beginning an *ingredients* block after the *graph*:  

```
    \ingredients
    {
        4 cl & Amaretto \\
        2 dl & Ginger Ale \\
        & Ice Cubes
    }
```

End all ingredients with a "\\\\" to indicate a new line. In every line you can use a & to distinguish between the amount and the actual ingredient. This will then be formatted properly.

### Add Preparation Steps
As we've got the ingredients covered, let's state what we have to do with them. For this we begin a *preparations* block that will contain several *steps*:  

```
    \preparation
    {
        \step Put ice in Glass
        \step Pour Amaretto over ice cubes
        \step Fill up with Ginger Ale
        \step Stir
    }
```

### Add Hint
Lastly you can chose to add some hint, for example for some alternative ways of preparing. Add a *hint* block and write whatever you want to convey:  

```
    \hint
    {
        If available, serve with a slice of lemon.
    }
```

After a last build and view the document should now look like this:
![Added recipe]({{ site.baseurl }}/images/posts/cookbook_as_code/added_recipe.png)

And the following is the full example of the recipe:  

```
\newpage
\begin{recipe}
[
	preparationtime = {\unit[10]{min}},
	bakingtime={\unit[30]{min}},
	bakingtemperature={\protect\bakingtemperature{fanoven=\unit[200]{°C}}},
	portion = {\portion{4}},
	calory,
	source = https://github.com/huserben/cookbook
]
	{My New Recipe}
	\graph
    {
        small=pictures/amaretto_ginger_ale/amaretto_ginger_ale.png
    }

    \ingredients
    {
        4 cl & Amaretto \\
        2 dl & Ginger Ale \\
        & Ice Cubes
    }
    \preparation
	{
		\step Put ice in Glass
		\step Pour Amaretto over ice cubes
		\step Fill up with Ginger Ale
		\step Stir
	}

\hint
{
	If available, serve with a slice of lemon.
}

\end{recipe}
```

# Automating Deployment
By now you have a nice pdf that you can extend with simple text files. That makes it nice to version, so you can use git (or whatever is in fashion by the time you read this) and push it to your own repository or to a service like github.

However we can go a bit further and use [Github Actions](https://docs.github.com/en/actions) to generate a new version of the pdf everytime you push some changes. Then we can make use of [Github Pages](https://docs.github.com/en/pages) to host the document. After that you have always the latest version of the cookbook available on every device, just by browsing to your project page - isn't that cool?

## Github Actions Workflow
We can define Github Actions Workflows by creating a yaml file in a *workflows* folder that is within a *.github* folder in the root of the repo. This will then be picked up automatically by github and will run your actions as defined.  

You can find the finished workflow in the cookbook repo under [.github/workflows/create_cookbook.yml](https://github.com/huserben/cookbook/blob/main/.github/workflows/create_cookbook.yml).

### Workflow Basics
The first few lines of the workflow define that we want to run the workflow everytime we pushed something to the main branch.  

```
on:
  push:
    branches:
      - main
```

Then we define a job that uses a ubuntu machine to run the workflow on and as a first step it will checkout the repo:  

```
jobs:
  Publish-Cookbook:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
```

### Create PDF
The main point of the workflow is of course to build the pdf from the *.tex files in the repo. For this we use the [texlive-action](https://github.com/xu-cheng/texlive-action) that is offering a full TexLive environment in a docker container in which we will execute our commands to build the pdf.  

We must be able to execute custom commands as we need to install the emerald font like we did locally. So after downloading and extracting the font, we run the same commands to setup the environment. After this we build the cookbook by running *pdflatex* against *MyCookbook.tex*. We do this twice as the second time it will properly generate the table of contents.  

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

### Publish PDF
Now that we have our pdf, we want to make it available to the world. For this we first copy the pdf to the *deploy* folder and then use an [action to deploy a folder to a *gh-pages* branch](https://github.com/peaceiris/actions-gh-pages). We will use this *gh-pages* branch in the next step.  

```
      - name: Copy Cookbook
        uses: canastro/copy-file-action@master
        with:
          source: "MyCookbook.pdf"
          target: "deploy/MyCookbook.pdf"

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./deploy
```

## Setup Github Pages
To complete the deployment, we want to make the cookbook reachable from the internet. For this we move to the settings page of our repo and setup the pages to use the *gh-pages* branch:
![GH Pages Setup]({{ site.baseurl }}/images/posts/cookbook_as_code/gh_pages_setup.png)

However the pdf will not be available under *https://username.github.io/repo* just yet. You would need to browse to */MyCookbook.pdf*. We can make this available directly under the url by adding a little bit of html.  

In the git repository add a *deploy* folder and add a *index.html* with following content:  

```
<embed src="./MyCookbook.pdf" type="application/pdf" width=100% height=100% />
```

Now you can browse to your github pages url and it will display the pdf in your browser. Everytime you modify your cookbook and push, within a few minutes the cookbook will be updated magically. If that's not cool, what is?

# Conclusion
If you followed along you learned about how you could LaTeX instead of a Word document or a collection of cookbooks, printouts and post it notes for your recipes. We use plain text to describe our files which makes it perfect to be version controlled. Which in turn opens the door to more automation via github actions.  

Next time you're at a friends place and are looking for **that recipe** you will have it easily at hand, just browse to your github and check it out.