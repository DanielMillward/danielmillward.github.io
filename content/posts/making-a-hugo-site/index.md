+++
date = '2025-04-02T13:27:12-06:00'
draft = false
title = 'Making a Hugo site'
categories = ["General"]
+++


After several different variants of making this blog[^1] I've settled on Hugo with a custom theme, deployed on GitHub pages. The last time I had touched Hugo I was very much new at developing. With a tiny bit of experience under my belt I found the process surprisingly easy! For future me (and anyone reading), here's the general process:

1. Install Hugo: https://gohugo.io/installation/ 
	1. Verify you've installed it with `hugo version`
2. Run:

```
hugo new site my-app-name
cd my-app-name
git init
hugo server
```

This should serve a site at `localhost:1313`, probably giving a 404.

3. Now let's make a custom theme (based on [this blog post](https://retrolog.io/blog/creating-a-hugo-theme-from-scratch/)):
```bash
hugo new theme exampleTheme
```
4. Change the root `hugo.toml` to look like:
```toml
baseURL = 'https://danielmillward.com/'
languageCode = 'en-us'
title = 'Daniel Millward'
theme = "exampleTheme"
```

We now have the framework we need to develop our custom site. 
## Hugo Crash Course

Hugo uses GoLang's style of templating, i.e. Having HTML interspersed with {{}} blocks containing variables/functions. The templates you write will live in `themes/exampleTheme/layouts/`. 

By default, Hugo generates two subfolders:
1. `_default`: These are the whole page and content templates.
2. `partials`: This is where common "building blocks" live, e.g. the header/footer and nav bar.

The main "master" page template is defined at `_default/baseof.html`. The default one Hugo generates has references to some partials (head and header) as well as the "main" block.

The home page (at the "/" path) main block is at `_default/index.html`. You'll have to make this one. It'll look something like this:

```html
{{ define "main" }}

<div class="mainContainer">
    <div class="heropicContainer">
        <img src="/images/Ghibli-Portrait-2.webp">
    </div>
    <div class="mainText">
        <p>Here's some text!</p>
      </div>
</div>

{{ end }}
```

The `{{ define "main }}` and `{{end}}` blocks tell Hugo to put this in the block called "main" in `baseof.html`.

Let's look at a partial component, `partials/header.html`:

```html
<div id="nav-border" class="navContainer">
    <h2 class="nameLinkH2">
        <a href="/" class="nameLink">Daniel Millward</a>
    </h2>
    {{/* Need a reference to the page object while looping through the menus */}}
    {{ $currentPage := . }}
    <nav id="nav">
        {{ range .Site.Menus.main.ByWeight }}
        <a href="{{ .URL }}" class="header-link" {{if eq $currentPage.Path .URL}}style="font-weight: bold"{{end}}>
            {{ $text := print .Name | safeHTML }}
            {{ $text }}
        </a>  
        {{ end }}

</div>
```

This shows off a lot more of Hugo's templating functionality. You can see:
 - setting variables: Make a variable name starting with `$` and set with `:=`
 - Loop over an array/slice with `range`
 - Piping with `|` into a function

A big thing here is the `.` operator (e.g. calling `.Name` and setting `$currentPage` to `.`). The dot is the current context: Think of it as calling `this.` as a shorthand. By default, the context is (I think?) the Page object. This has a ton of methods you can use to get information about the current page, e.g. the title. 

The context can change: In the `{{range .Site.Menus.main.ByWeight}}` block the context is each of the menu items.

The `.Site` object can be changed for our needs by changing the configuration files. For this example, the `Menus` object was changed by appending this to the root `hugo.toml` file:
```toml
[menu]
  [[menu.main]]
    name = "Posts"
    url = "/posts"
    weight = 1
  [[menu.main]]
    name = "Categories"
    url = "/categories"
    weight = 2
```

Notice we called the .ByWeight function in `header.html`: Hugo lets us sort them by the weight attribute we've set.

In addition to the `index.html` page, Hugo automatically generates pages based on the `content` folder:

 - If you make a subfolder in `content` (or a sub-sub folder) that has an `_index.md` file, it will make a "list" page at that directory URL. For example, if you make a `content/posts` page with an `_index.md` file a "list" page will be accessible at `yoursite.com/posts`.
 - If you make a subfolder in `content` that has a `index.md` file, it will make a regular content page At that directory URL.
 - If you make a regular markdown file anywhere, Hugo makes a regular content page there.
 - If you make a subfolder that itself has multiple regular folders/regular pages, it generates a "list" page regardless of whether a `_index.md` file is found.

The folders with a `_index.md` file are called Sections.

So what's a "list" page? It's an automatically generated page that can list the pages in its directory. How does it know how to generate it? It uses the first matching template it finds in a [set lookup order](https://gohugo.io/templates/lookup-order/#section-templates). For example, if you made a folder `content/posts` and that had some regular pages, you could put the listing template at `themes/exampleTheme/layouts/posts/list.html`. It could look something like:
```html
{{ define "main" }}
<div class="postList">
  {{ .Content}}
  {{ range (where .Site.RegularPages "Type" "in" (slice "posts")).GroupByDate "2006" }}
  <div class="categoryBlock">
    <h2 class="postYear">{{ .Key }}</h2>
  {{ range .Pages }}
    {{ partial "postBlock.html" .}}
    {{- end -}}
  {{ end }}
  </div>
</div>
{{ end }}
```

Some new things here:

 - `.Content` is the rendered markdown found in the `_index.md` file in `content/posts/`.
 - Blocks with a dash, e.g. `{{-`, remove whitespace.
 - Referencing `{{.Key}}` in a range block of a map lets you get the key.
 - `range (where .Site.RegularPages "Type" "in" (slice "posts")).GroupByDate "2006"` comes from [this blog post](https://digitalnotions.net/divide-post-list-by-year-in-hugo/) which just separates all posts by year. NOTE: By referencing `.Site.RegularPages` instead of just `.Pages` we get ALL the regular pages inside the `content` folder, regardless of section!

To set what an individual regular page looks like, change the   `themes/exampleTheme/layouts/_default/single.html` template. You can get data from the [frontmatter](https://dpericich.medium.com/what-is-front-matter-and-how-is-it-used-to-create-dynamic-webpages-9d8dc053b457) of whatever markdown file is being rendered. Common variables are `.Date` and `.Title`. You reference the content itself with `.Content.`

We're pretty close to finishing the blog! If you have some images/other static stuff you want to reference, put them in the root `static` folder. If you keep the default `partials/head/css.html` then your main css file will be at `themes/exampleTheme/assets/css/main.css`.

It's best to add a `.gitignore` that has the `public` directory, since that's where Hugo builds the static site.

Specifically for writing Latex: In the `partials/head.html` template, add this at the bottom:
```html
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        inlineMath: [ ['$','$'], ["\\(","\\)"] ],
        processEscapes: true
      }
    });
  </script>
      
  <script type="text/javascript"
          src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
  </script>
```

That's pretty much it! The deployment is very straightforward: just upload your repo to GitHub and add a yaml workflow. The process is documented thoroughly [on Hugo's website](https://gohugo.io/host-and-deploy/host-on-github-pages/).

If you want to add a custom URL, that's also straightforward: GitHub [has an article on this](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site). Note that for this to work you'll need your GitHub repo to be called YourUserName.github.io, otherwise Hugo gets confused on where to link to your static files.