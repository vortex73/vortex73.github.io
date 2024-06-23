% title: Ludicrosity
% description: A Static Site Generator in Zig
% tags: zig,ssg,project
% type: post
% author: vortex
% date: 2024-05-10

Writing a blog can be quite the demanding task if you're not equipped with the right tools and strategies.
Enter *Ludicrosity* a static site generator written in Zig.

```text
   __           _ _                    _ _         
  / / _   _  __| (_) ___ _ __ ___  ___(_) |_ _   _ 
 / / | | | |/ _` | |/ __| '__/ _ \/ __| | __| | | |
/ /__| |_| | (_| | | (__| | | (_) \__ \ | |_| |_| |
\____/\__,_|\__,_|_|\___|_|  \___/|___/_|\__|\__, |
                                             |___/
A static site generator in Zig

```

## TODO
Its awesome but not perfect. Here's a list of things I need to get to at some point.
   ```text

   - [x] Fix tagging
   - [x] Sorting Posts
   - [ ] File render caching
   - [ ] User defined pages
   - [ ] Syntax Highlihgting
   - [ ] Searching through posts
   - [ ] Making code more modular

   ```

## Building Ludicrosity
I personally write notes in [markdown](https://en.wikipedia.org/wiki/Markdown) for everything that I learn. The necessity for a static site generator was birthed by the need to publish all of these into a mass accessible medium like a website.

> Why a static site?
> Static Sites are simple and quick to host with minimal(and in my case *no*) interaction with javascript.

v0.1 of Ludicrosity was built purely using Vanilla javascript(*yeah*). This was the foundation of the core of the engine and where the *unique* templating format was born!
> find the legacy version [here!](https://github.com/vortex73/ludicrosity/tree/archive).

However JS while *quite powerful* can be difficult to manage over time in larger projects. *enter:* [Zig](https://ziglang.org/), a relatively new systems programming language with a more controlled environment with an emphasis on memory safety. It also helped that I was learning Zig concurrently at the time.

Ludicrosity can feed on full-fledged markdown and churn out a static site.

All source code for ludicrosity is housed within one single `src/marked.zig` for ultimate simplicity.
Currently we depend on one dependency [md4c](https://github.com/mity/md4c) for markdown parsing.
To build the project we expect the following source tree:

```text

.
├── build.zig
├── md4c
└── src
    └── marked.zig

```

The project can be cloned and built using the following commands:

```text

git clone git@github.com:vortex73/ludicrosity.git --recurse-submodules # pulls even the md4c dependency
zig build -Doptimize=ReleaseFast

```

## Benchmarking
The project was built from the ground up with an emphasis on speed. Meticulous efforts have gone into reducing the number of system-calls that Ludicrosity makes per file render. While a *LOT* more can be done, here's the current measure:

![bench](/assets/bench.png)

> Checkout [Zero](https://github.com/procub3r/zero) another insane SSG written in Zig! :D

## Project Structure
Ludicrosity is highly opinionated. Which means you need to adhere to the following file structure strictly.

```text

[project root]
├── assets
├── content
│   └── post1.md
│   └── post2.md
│   └── ...
├── css
│   └── style.css
├── rendered
├── tags
└── templates
    └── post.html
    └── tags.html
    └── snippets
        └── header.html
        └── footer.html
        └── ...

```

Ludicrosity will read template files `post.html` and `tags.html`, and render the converted markdown into them and store them in the `rendered/` directory.
A file will be created for each tag listing all posts referencing that particular tag in the `tags/` directory.

## Metamatter<sup>TM</sup>
While YAML frontmatter is great, I wanted something distinctly unique and yet easy to parse to store the metadata. Inspired heavily from the [Pandoc](https://pandoc.org/chunkedhtml-demo/8.10-metadata-blocks.html) style of writing metadata, Ludicrosity uses its own frontmatter format called `Metamatter`. And its beautifully simple.

```text

% title: Post0
% date: 24/11/1951
% author: mukesh
% type: post
% tags: fast,post,blog,portfolio

```
Thats it!

## Templating Language? What's that?

Ludicrosity has limited templating capabilities at the moment.

```text

<div class="content">
    <h1><!--title--></h1>
    <div class="meta">
        <ul>
            date: <!--date--><br>
            type: <!--type--><br>
            <ul>tags: <!--tags--></ul>
        </ul>
    </div>
    <!--BODY-->
</div>

```
Mention the tagname as per what was mentioned in the `Metamatter` block inside an HTML comment element. This will be substituted for the posts' actual metadata.
`<!--BODY-->` is exclusively reserved for the body of the post. 
### Snippets
```text

<!--@header@-->
...
<!--@footer@-->

```
Ludicrosity supports recursive snippets functionality. Define the snippet in `templates/snippets/` and use inside templates using the syntax `<!--@snippet@-->`.
