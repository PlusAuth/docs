# PlusAuth Documentation Content
This repository contains content of pages served at [docs.plusauth.com](https://docs.plusauth.com). We use markdown 
and mdx to provide best experience to both our consumers and content writers.

All documentation content are located in [/content](/content) folder and its first sub-level is the short ISO
code for the locale that the content is targeted to. By default, English (en) locale is used and other locales fallback 
to English when the content is missing in that locale.

# Writing Content

- PlusAuth Docs generates navigation sidebar from the directory/file structure. File and folder names bust 
be `kebab-cased` and can contain numbers for sorting purposes.
- You can write your content in `md` or `mdx` files. Both of the extensions are treated the same. 
- You can define page metadata in frontmatter section of markdown files.
- HTML in markdown files are supported, and you can use [TailwindCSS](https://tailwindcss.com/) classes.
