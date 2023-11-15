---
title: "Migrating from Wordpress"
date: "2023-02-26T16:01:00Z"
categories: 
  - "blog"
  - "wordpress"
  - "personal"
  - "hugo"
  - "migration"
  - "markdown"
---

So, after many years of happy blogging -- some years more often than others -- I've finally run out of hosting space with my cheap and cheerful hosting provider. I thought about just paying for more storage, but with retirement and the closing of my -- and by "my" I mean, "mine and MrsD's" IT Company -- I decided that a full [Wordpress](https://wordpress.org/ "WordPress website") website wasn't really necessary, what with its need for a database such as [MySQL](https://www.mysql.com/ "MySQL website") and [PHP](https://www.php.net/ "PHP website"). In the end I found a static site generator which is named [Hugo](https://gohugo.io/ "Hugo website").

## Hugo

Hugo is, as briefly mentioned, a static website generator, apparently, the fastest in the world. I write a topic in [Markdown](https://www.markdownguide.org/) and Hugo builds my website from that. I have chosen as a theme -- by default, Hugo doesn't install a theme -- the [ZZO](https://github.com/zzossig/hugo-theme-zzo) theme. It took a lot of experimenting and playing around, but I finally found out how to make it do pretty mush all I wanted.

The ZZO theme has [excellent documentation](https://zzo-docs.vercel.app/zzo "ZZO theme's excellent documentation").

I have to say that one of Hugo's drawbacks is the fact that there are numerous themes available, some free, some paid for. A strange thing to say? (or type?) but, the problem is pretty much that no two themes do exactly the same things for the same outcome! Theme *Sortcodes* are excellent tools, but start using them in your blog source, and try changing themes -- in the majority of cases, it will require some amendment.

However, I'm not planning on changing my theme, I hope!

## Migration

Exporting from Wordpress was easy enough, I logged in to the admin panel, and exported my entire site as XML and downloaded the generated file.

Next, I installed a set of tools from [Lone Korean on GitHub](https://github.com/lonekorean/wordpress-export-to-markdown) and after extracting the Zip file, I followed the instructions in the readme to install them:

```bash
npx wordpress-export-to-markdown
```

And after running the code, I had my old blog site converted to a pile of individual Markdown files. I chose to configure the conversion where each blog post was put into a directory for the year and a sub-directory for the month. Under this, each post for that particular month gets it's own sub-directory and a file named `index.md`. Everything was contained in a top level directory named `posts`. It looks something like this:

```text
content
|
+----posts
|    |
|    +----2023
|    |    |
|    |    +----migrating-from-wordpress
|    |    |    |
|    |    |    +----index.md
|    +----2022
|    +----2021
|    +----2020
|    +----2019

etc
```

Once the utility had completed, I simply lifted the `posts` folder and pasted it into the Hugo's `content` folder. Running up a development HTML server with:

```bash
hugo server -D
```

I connected my browser to [http://localhost:1313](http://localhost:1313) and I had a blog!

I'll be looking to get it online ASAP, I may stick with my current host, but I'm researching [Netlify](netlify.com) as a potential host as:

* It's free!
* It comes highly recommended by many.
* I can update my blog simply by pushing changes to a GitHub repository.
* I can use my current URL for the blog -- I think!
* Comments, if I choose to enable them, are handled by [Disqus](https://disqus.com/) through the HUgo ZZO theme.


There are obviously limits, but I don''t see myself making too many blog postings over the years, and certainly I won't be initiating builds every 5 minutes or so!'

I wonder how it will all go?

## Editing Posts

After the conversion, I had a look at the generated Markdown. It was mostly ok, strangely enough. However, a few things needed editing:

* Some special characters had been escaped, and this is not needed in Hugo:
  * `\_` changed to `_`.
  * `\*` changed to `*`.
  * `\-` changed to `-`.
  * `\=` changed to `=`.
  * `\[` changed to `[`.
  * `\]` changed to `]`.
  * `\#` changed to `#`.
* Some bash root prompts, the `#` had to be changed to something that didn't force a heading! The `#` is Markdown's top level heading marker.
* A lot of code sections, from the original Wordpress blog posts were not exported as code by Wordpress. These had to be found and "codified" using Markdown techniques. I also took the liberty of adding proper code highlighting depending upon the code language in question.
* Some postings had to be reverted to previous versions and re-converted. It seems that my blog got somewhat hacked over the years and lots of my stuff was deleted and replace by spam postings offering dubious services and fad diets, oh yes, and a plumbing service in Oklahoma, USA.  
* Images. The default on Wordpress is that when an image is clicked, it opens a full sized version of the thumbnail used in the post. The conversion converted the thumbnail image's details, but left the link to my Wordpress blog. This had to be edited, but was simple. I just replaced the `http://qdosmsq.dunbar-it.co.uk/blog/` part of the link, with `/posts/`, and it "just" worked -- because I chose to give my post a year and month directory during the conversion"

Cheers,
Norm.



