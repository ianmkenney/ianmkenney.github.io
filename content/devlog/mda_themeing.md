---
title: "MDAnalysis Sphinx documentation style for MDAKits"
date: "2023-07-28"
description: "Sphinx documentation style instructions for MDAnalysis projects"
---

With the planned expansion of the [MDAKits](https://mdakits.mdanalysis.org) framework, there is a greater need for instructions on deploying documentation with the [MDAnalysis](https://mdanalysis.org) style.
Currently, there is no installable theme and all configurations will be handled manually.
Here I will outline the current proceedure for applying an MDAnalysis theme to Sphinx documentation.

Until a proper Sphinx extension is published, we will be assuming that our basic configurations are generated with the default values provided by the [MDAKit cookiecutter template](https://github.com/MDAnalysis/cookiecutter-mdakit).

### Configuration file changes

MDAnalysis codes are likely to contain citations to literature and this should be automatically included with the theme. 
We start by adding bibtex support. by adding the bibtex extension to the `extensions` list in `docs/source/conf.py`.
Change the original list from 

```python
extensions = [
    'sphinx.ext.autosummary',
    'sphinx.ext.autodoc',
    'sphinx.ext.mathjax',
    'sphinx.ext.viewcode',
    'sphinx.ext.napoleon',
    'sphinx.ext.intersphinx',
    'sphinx.ext.extlinks',
]
```

to

```python
extensions = [
    'sphinx.ext.autosummary',
    'sphinx.ext.autodoc',
    'sphinx.ext.mathjax',
    'sphinx.ext.viewcode',
    'sphinx.ext.napoleon',
    'sphinx.ext.intersphinx',
    'sphinx.ext.extlinks',
    'sphinxcontrib.bibtex',  # <--- Add this line
]

bibtex_bibfiles = ['references.bib']
# bibtex_bibfiles = []  # <--- if you're not using citations
                        #      provide an empty list
```

This package is not installed by default and will need to be added to the `docs/requirements.yaml`.

```yaml
name: mdaencore-docs
channels:
dependencies:
    # Base depends
  - python
  - pip

  - sphinx<7.0

  - sphinx_rtd_theme
  - sphinxcontrib-bibtex  # <--- Add this line
  - pip:
    - msmb_theme
```

In `conf.py`, update the `html_theme` from `'XXX'` to `'msmb_theme'`.
Below this line, add the following block:

```python
html_theme_path = [
    msmb_theme.get_html_theme_path(),
    sphinx_rtd_theme.get_html_theme_path(),
]

# styles/fonts to match http://mdanalysis.org (see public/css)
#
# /* MDAnalysis orange: #FF9200 */
# /* MDAnalysis gray: #808080 */
# /* MDAnalysis white: #FFFFFF */
# /* MDAnalysis black: #000000 */

color = {'orange': '#FF9200',
         'gray': '#808080',
         'white': '#FFFFFF',
         'black': '#000000', }

extra_nav_links = OrderedDict()
extra_nav_links['MDAnalysis'] = 'http://mdanalysis.org'
extra_nav_links['docs'] = 'http://docs.mdanalysis.org'
extra_nav_links['wiki'] = 'http://wiki.mdanalysis.org'
extra_nav_links['user discussion group'] = 'http://users.mdanalysis.org'
extra_nav_links['GitHub'] = 'https://github.com/mdanalysis'
extra_nav_links['@mdanalysis'] = 'https://twitter.com/mdanalysis'

html_theme_options = {
    'canonical_url': '',
    'logo_only': True,
    'display_version': True,
    'prev_next_buttons_location': 'bottom',
    'style_external_links': False,
    'style_nav_header_background': 'white',  # '#e76900', # dark orange
    # Toc options
    'collapse_navigation': True,
    'sticky_navigation': True,
    'navigation_depth': 4,
    'includehidden': True,
    'titles_only': False,
}
```

With the changes so far, add 

```python
from collections import OrderedDict
import msmb_theme
import sphinx_rtd_theme
```

to the imports near the top of `conf.py`.

Below the line with `html_static_path = ['_static']`, add the following block

```python
html_favicon = '_static/logos/mdanalysis-logo.ico'
html_logo = '_static/logos/mdanalysis-logo-thin.png'
html_css_files = ['custom.css']
html_sidebars = {
    '**': [
        'about.html',
        'navigation.html',
        'relations.html',
        'searchbox.html',
    ]
}
```

Download and install the following files to their respective locations (relative to `docs/source`):

| Source                      | Target                                   |
|-----------------------------|------------------------------------------|
| [Custom CSS][customcss]     | `_static/custom.css`                     |
| [MDA icon][mdaicon]         | `_static/logos/mdanalysis-logo.ico`      |
| [MDA main logo][mdalogo]    | `_static/logos/mdanalysis-logo-thin.png` |

[customcss]: https://raw.githubusercontent.com/MDAnalysis/hole2-mdakit/main/docs/source/_static/custom.css
[mdaicon]: https://raw.githubusercontent.com/MDAnalysis/mdanalysis/develop/package/doc/sphinx/source/_static/logos/mdanalysis-logo.ico
[mdalogo]: https://raw.githubusercontent.com/MDAnalysis/mdanalysis/develop/package/doc/sphinx/source/_static/logos/mdanalysis-logo-thin.png

After creating and updating your environment, you should be able to build your newly formatted documentation.
