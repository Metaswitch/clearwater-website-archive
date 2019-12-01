Good documentation, ReadTheDocs and Markdown
--------------------------------------------
Project Clearwater prides itself on being user-friendly and easy to get started with - people often download and try out Project Clearwater as their first introduction to NFV and IMS - and having high-quality, accurate documentation is an important part of that. There are two main tools we use to help make sure our documentation is usable.

*   We use Markdown as the markup language for our documentation. This is a simple way of expressing formatting (headings, bullet points, links, etc) that can be written easily from any text editor, which both lets us focus on the content rather than worrying about HTML or DocBook tags, and makes it as easy as possible to contribute documentation. This means that we're never put off writing documentation because it's too difficult, and more importantly, that it's very easy for a member of the open-source community to fix or improve our documentation.
*   We host our documentation on ReadTheDocs.org - you can find this at http://clearwater.readthedocs.org. We originally used the Github wiki for documentation, but ReadTheDocs makes our documentation searchable, and allows us to structure it with a table of contents - both of which make it easier for users to find exactly what they're looking for.

ReadTheDocs uses reStructuredText by default, so unfortunately ReadTheDocs and Markdown don't always go well together. We didn't really want to switch to reStructuredText – it has more features, including tables and footnotes, but that makes it more complicated and less intuitive to use (and ease-of-use, for us and the community, is the best thing about Markdown). Instead, we've looked at and tried various approaches to writing Markdown docs on ReadTheDocs, and now found one that works for us, which we thought we'd blog about for the benefit of fellow RTD/Markdown fans.

## MkDocs

When we first moved to ReadTheDocs, the recommended Markdown solution was to use MkDocs. reStructuredText/Sphinx was much more commonly used on ReadTheDocs, though, and MkDocs support felt less well-tested and less solid. We hit a couple of issues, particularly around search and broken links, so we started looking at what else we could try. It's now been [removed](https://github.com/rtfd/readthedocs.org/commit/d7cfe1a1d56bb0578ce08240fb92f715ded37d8a#diff-e3e2a9bfd88566b05001b02a3f51d286) from the ReadTheDocs instructions, in favour of using CommonMark with Sphinx.

## Sphinx and CommonMark

There's been a recent [announcement](http://blog.readthedocs.com/adding-markdown-support/) that ReadTheDocs and Sphinx now support Markdown using CommonMark. We looked at this, but that blog post lists some restrictions that concerned us - in particular, there's no support yet for tables of contents. We have a lot of documentation which benefits from structure (i.e. grouping our instructions for manual install, automated install and deployment of all-in-one images all under one heading of "Installation") - since CommonMark doesn't provide this, we're not using it. (This is likely to be a good solution once it matures, though.)

## Our solution

Since reStructuredText files seemed to be much better supported and more mainline on ReadTheDocs than Markdown files were, we decided to take the approach of auto-converting our files into reStructured text. There are two ways we do this.

*   The open-source tool pandoc, for converting between document formats, can convert .md files to .rst files - we run this over all our Markdown files to convert them.
*   The MkDocs table of contents file is YAML, and can easily be parsed with PyYAML - so we've written a tool (open-source under the GPLv3) to turn this into a reStructuredText table of contents.

Our continuous integration server (Jenkins CI) monitors our documentation repository, and when it sees a change to the Markdown files, it generates rST versions of the same docs and an rST table of contents, and checks them in. This is working well for us so far - search in particular has become much more stable. If you've got some Markdown/MkDocs content which you'd like to put on ReadTheDocs, give our tools a try!
