---
layout: post
title: Creating a minimalist blog with Jekyll Now.
---

In this post, I share why I chose [Jekyll Now](https://github.com/barryclark/jekyll-now) and [Github Pages](https://pages.github.com/) to build this blog.

As I grow older, I have accumulated dozens of side projects that I have done or tried to do. Most of them fail or don't get beyond a quick prototype. Nevertheless, in almost all of them I have learned some new technology, and in the end these learnings are the most valuable result of these side projects.

I wanted to create a blog to share this knowledge, with these requirements:
- **Full control the web page**. Instead of using a hosted service like Facebook or Medium, as I strongly believe in a decentralized web.
- **"Just work"**. I want a system that let me focus on writing instead of running the website.
- **Minimalist**. I want it to be minimalist, both visually and in functionality.
- **Good ergonomics**. I want to be able to write markdown and have a fairly well formatted website, instead of writing HTML directly.

I was aware about [GitHub pages](https://pages.github.com/) that lets you host a static website directly from a GitHub repository. This fitted my needs really well, as it would abstract away most of the work of running a website, and I can easily migrate it to my own servers if I desire in the future.

However, GitHub pages requires the content to be in HTML, and that was not ergonomic enough for me to write a blog with. My first inclination was to write some simple code to convert the markdown files into HTML, and automatically build it with [GitHub actions](https://github.com/features/actions). This started to get more complicated as I started thinking about how to code basic features like the page listing the posts, and the re-usage of the layout on all the pages. I wanted to focus on writing to humans in this project, not writing more code. 

Then, I found [Jekyll Now](https://github.com/barryclark/jekyll-now) after doing some research about these requirements. It did sound exactly like what I wanted, but I was initially put off by the example on its repository that seemed overly complex. It had a huge configurations file and a lot of features like linking Facebook, LinkedIn, Google+ (!!!), Twitter, Google Analytics, etc, and I was worried about getting into a technical rabbit hole. 

Nevertheless, I decided to learn a bit about Jekyll as the alternative of creating something from scratch wasn't so simple either. I am very happy that I did. [Jekyll](https://jekyllrb.com/) is elegant and has very simple rules and features that allowed me to meet all my needs:
- **URLs**: A document named `foo.html` or `foo.md` on the root of the repository will generate the corresponding `/foo` page on your website.
- **Layouts**: You can re-use layouts by just creating a file named `foo.html` on the `_layouts` folder and adding
```
---
layout: foo
---
```
to the header of the pages or other layouts you want to use it. The page will replace `{% raw %}{{content}}{% endraw %}` on the layout file.
- **Posts Page**: Markdown files created on the `_posts` folder with the format `YYYY-MM-DD-Title.md` will create a blog post. The posts URL can be set with the `permalink` configuration. In this website, I have the permalink configured to be `/blog/:title/`, so a file named `_posts/2021-01-01-Hello-World.md` will have the link `/blog/Hello-World`.
- **Posts Summary**: You can loop through each post using the `site.posts` variable in any page, and each post will have a `title`, and also an automatically generated `excerpt` to use on the page listing all posts.
- **Automatic push**: [Jekyll Now](https://github.com/barryclark/jekyll-now) comes with a pre-built Github Actions to automatically build the website, which in turn reflects on your Github Page in seconds after changing any file. No Terminal required :).
- **Development environment**: For quickly making changes on your local machine and seeing the results, you only need to install ruby 2.7, this gem: `gem install github-pages` and run `jekyll serve`

All in all, using [Jekyll Now](https://github.com/barryclark/jekyll-now) and removing its bloat has been a great solution that gave me the flexibility that I wanted with little to no overhead.  You can see the [final code for this blog here](https://github.com/vdfdev/vdf.dev).
