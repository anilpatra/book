[![Stories in Ready](https://badge.waffle.io/atk4/book.png?label=ready&title=Ready)](https://waffle.io/atk4/book)
Agile Toolkit Documentation
===========================

This is the official documentation for the Agile Toolkit project. It is available online in HTML, PDF and EPUB formats at http://book.agiletoolkit.org.

Requirements
------------

You can read all of the documentation within as its just in plain text files, marked up with ReST text formatting.  To build the documentation you'll need the following:

* Make
* Python
* Sphinx
* PhpDomain for sphinx

You can install sphinx using:

	easy_install sphinx

You can install the phpdomain using:

	easy_install sphinxcontrib-phpdomain

*To run the easy_install command, the setuptools package must be previously installed.*

Building the Documentation
--------------------------

After installing the required packages, you can build the documentation using `make`.

	# Create all the HTML docs. Including all the languages.
	make html

	# Create just the english HTML docs.
	make html-en

	# Create all the EPUB (e-book) docs.
	make epub

	# Create just the engish EPUB docs.
	make epub-en

	# Populate the search index
	make populate-index

This will generate all the documentation in an HTML form.  Other output such as 'htmlhelp' are not fully complete at this time.

After making changes to the documentation, you can build the HTML version of the docs by using `make html` again.  This will build only the HTML files that have had changes made to them.

Building the PDF
----------------

Building the PDF is a non-trivial task.

1. Install LaTeX - This varies by distribution/OS so refer to your package manager. You should install the full LaTeX package. The basic one requires many additional packages to be installed with `tlmgr`
2. Run `make latex-en`.
3. Run `make pdf-en`.

At this point the completed PDF should be in build/latex/en/


Contributing
------------

Contributing to the documentation is pretty simple. Please read the documentation on contributing to the documentation over on [the cookbook](CONTRIB.md) for help.

There are currently a number of outstanding issues that need to be addressed.  We've tried to flag these with `.. todo::` where possible.  To see all the outstanding todo's add the following to your `config/all.py`

	todo_include_todos = True

After rebuilding the HTML content, you should see a list of existing todo items at the bottom of the table of contents.

You are also welcome to make and suggestions for new content as commits in a GitHub fork.  Please make any totally new sections in a separate branch.  This makes changes far easier to integrate later on.

Translations
------------

Contributing translations requires that you make a new directory using the two letter name for your language.  As content is translated, directories mirroring the english content should be created with localized content.


Build Scripts
-------------

I experss my thanks to CakePHP Documentation project, which inspired the
build scripts for Agile Toolkit.

