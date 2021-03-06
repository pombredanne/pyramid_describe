===================================
Self-Documentation for Pyramid Apps
===================================

.. warning::

  2013/09/13: though functional, this package is pretty new... Come
  back in a couple of weeks if you don't like living on the
  beta-edge!

A pyramid plugin that describes a pyramid application URL hierarchy,
either by responding to an HTTP request or on the command line, via
application inspection and reflection. It has built-in support for
plain-text hierachies, reStructuredText, HTML, PDF, JSON, YAML, WADL,
and XML, however other custom formats can be added easily.

Exposing an application's structure via HTTP is useful to dynamically
generate an API description (via WADL, JSON, or YAML) or to create
documentation directly from source code.

On the command-line it is useful to get visibility into an
application's URL structure and hierarchy so that it can be understood
and maintained.

.. note::

  Although pyramid-describe is intended to be able to describe any
  kind of pyramid application, currently it only supports
  pyramid-controllers_ based dispatch.

.. note::

  Currently, pyramid-describe can only inspect the first Controller
  it finds -- this will eventually be fixed to correctly implement
  the `inspect` option.


Project Info
============

* Project Page: https://github.com/cadithealth/pyramid_describe
* Bug Tracking: https://github.com/cadithealth/pyramid_describe/issues


TL;DR
=====

Install:

.. code-block:: bash

  $ pip install pyramid-describe

Command-line example:

.. code-block:: bash

  $ pdescribe example.ini --format txt
  /                       # The application root.
  ├── contact/            # Contact manager.
  │   ├── <POST>          # Creates a new 'contact' object.
  │   └── {CONTACTID}     # RESTful access to a specific contact.
  │       ├── <DELETE>    # Delete this contact.
  │       ├── <GET>       # Get this contact's details.
  │       └── <PUT>       # Update this contact's details.
  ├── login               # Authenticate against the server.
  └── logout              # Remove authentication tokens.

.. TODO - figure out how to serve these assets with the correct Content-Type...

Examples of the above application in all other formats with built-in
support are available at:
`text (pure-ASCII) <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.txt.asc>`_,
`reStructuredText <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.rst>`_,
`HTML <http://htmlpreview.github.io/?https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.html>`_,
`PDF <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.pdf>`_,
`JSON <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.json>`_,
`YAML <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.yaml>`_,
`WADL <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.wadl>`_,
and `XML <https://raw.github.com/cadithealth/pyramid_describe/master/doc/example.xml>`_.


Installation
============

Install with the usual python mechanism, e.g. here with ``pip``:

.. code-block:: bash

  $ pip install pyramid-describe


Usage
=====

There are three mechanisms to use pyramid-describe: via standard
pyramid inclusion which will add routes to the current application, by
explicitly embedding a ``pyramid_describe.DescribeController``
instance, or by directly calling the ``pyramid_describe.Describer``
object methods.


Pyramid Inclusion
=================

Pyramid-describe can be added via standard pyramid inclusion, either
in the INI file or directly in your `main` function. For example:

.. code-block:: python

  def main(global_config, **settings):
    # ...
    config.include('pyramid_describe')

When using pyramid inclusion, pyramid-describe expects to find
configuration options in the application settings. See the `Options`_
section for a listed of all supported options, with a short example
here:

.. code-block:: ini

  [app:main]

  describe.attach                        = /doc
  describe.formats                       = html json pdf
  describe.format.default.title          = My Application
  describe.format.html.default.cssPath   = myapp:static/doc.css
  describe.entries.filters               = myapp.describe.entry_docify

Note that multiple describers, each with different configurations, can
be added via pyramid inclusion by using the `describe.prefixes`
option.


DescribeController
==================

Pyramid-describe can also be added to your application by embedding a
DescribeController object. The DescribeController constructor takes
the following parameters:

`view`:

  An instance of ``pyramid.interfaces.IView``, which is the view that
  should be inspected and reflected.

`root`:

  The root path to the specified URL, so that host-relative URLs can
  be generated to the views found.

`settings`:

  A dictionary of all the options to apply to this describer. Note that
  in this case, the options should not have any prefix.

Example:

.. code-block:: python

  from pyramid_describe import DescribeController

  def main(global_config, **settings):
    # ...
    config.include('pyramid_controllers')

    settings = {
        'formats'                       : ['html', 'json', 'pdf'],
        'format.default.title'          : 'My Application',
        'format.html.default.cssPath'   : 'myapp:static/doc.css',
        'entries.filters'               : 'myapp.describe.entry_docify',
      }

    config.add_controller('MyAppDescriber', '/doc', DescribeController(settings))


Describer
=========

Pyramid-describe can also be added to your application by directly
calling the Describer's functionality. This is an even lower-level
approach than, but still quite similar to, embedding the
`DescribeController`_; the constructor takes the same `settings`
parameter as the DescribeController, and then a call to the `describe`
method actually generates the output. The `describe` method takes as
parameters a `context` and a `format`, and returns a dictionary with
the following attributes:

.. TODO - document `context` and `format`...

`content_type`:

  The MIME content-type associated with the rendered output.

`charset`:

  The character set that the output is encoded in.

`content`:

  The actual rendering output.

Example:

.. code-block:: python

  from pyramid_describe import Describer

  def my_describer(request):

    settings = {
        'formats'                       : ['html', 'json', 'pdf'],
        'format.default.title'          : 'My Application',
        'format.html.default.cssPath'   : 'myapp:static/doc.css',
        'entries.filters'               : 'myapp.describe.entry_docify',
      }

    describer = Describer(settings=settings)
    context   = dict(request=request)
    result    = describer.describe(context=context, format='pdf')

    request.response.content_type = result['content_type']
    request.response.charset      = result['charset']
    request.response.body         = result['content']

    return request.response


Options
=======

The configuration of pyramid-describe is done by setting any of the
following options. Note that if specified in the application settings
(i.e. the INI file), then they must be prefixed with (by default)
``describe.``. Otherwise, when passing a dictionary of settings to the
constructors, the prefix is left off. The following options exist:

* ``describe.prefixes`` : list(str), default: 'describe'

  Defines the prefix or the list of prefixes that pyramid-describe
  settings will be searched for in the configuration. For each prefix,
  a separate DescribeController will be created and attached to the
  application router. The following example attaches two controllers
  at ``/desc-one`` and ``/desc-two``:

  .. code-block:: ini

    [app:main]
    describe.prefixes = describe-one describe-two
    describe-one.attach  = /desc-one
    # other `describe-one` options...
    describe-two.attach  = /desc-two
    # other `describe-two` options...

* ``describe.class`` : resolve-spec, default: pyramid_describe.DescribeController

  Sets the global default Controller class that will be instantiated
  for each of the stanzas defined in `describe.prefixes`. Note that
  this option can be overriden on a per-stanza basis.

* ``{PREFIX}.class`` : resolve-spec, default: `describe.class`

  Sets the Controller class that will be instantiated for this PREFIX
  stanza, overriding `describe.class`.

* ``{PREFIX}.attach`` : str, default: /describe

  Specifies the path to attach the controller to the current
  application's router. Note that this uses the `add_controller`
  directive, and ensures that pyramid-controllers has already been
  added via an explicit call to ``config.include()``. This path will
  serve the default format: to request alternate formats, use
  "PATH/FILENAME.EXT" (where FILENAME is controlled by the
  ``{PREFIX}.filename`` configuration and EXT specifies the format)
  or use the "format=EXT" query-string. Examples using the default
  settings:

  .. code-block:: text

    http://localhost:8080/describe/application.txt
    http://localhost:8080/describe/application.json
    http://localhost:8080/describe?format=json

* ``{PREFIX}.fullname`` : str, default: 'application'

  Sets the filename (excluding the extension) that the output will be
  served at using the DescribeController. The extension provided by
  the request will determine which format to serve, and must be listed
  in the `formats` option. If the format is not listed, a 404 is
  returned. Typically, this is set to the application's name and
  might also include the application version.

* ``{PREFIX}.basename`` : str, default: null

  Similar to the `fullname` option, this option sets a filename base
  component that will either redirect to the current `fullname` or
  actually serve the content based on the `base-redirect` option. This
  allows there to be a persistent known location that can be used if
  the `filename` option is dynamic or changes with revisions.

* ``{PREFIX}.index-redirect`` : { bool, int, str }, default: true

  Controls what happens when a request comes to the index location
  of the DescribeController, i.e. the value of the `attach` option.
  The following values are accepted:

  falsy

    Responds with the actual content using the default format.

  truthy

    Redirects with a 302 to the `basename` if set, otherwise to
    the `fullname`, using the default format's extension.

  int

    Same as if truthy, but uses the specified response code (e.g.
    301 instead of 302).

  str

    Responds with a redirect using the specified string as the
    ``Location`` header. By default, issues a 302 unless the string is
    prefixed with the code and a space, e.g. ``301
    /path/to/filename``. If the location is not absolute, it will be
    evaluated relative to the current URL.

* ``{PREFIX}.base-redirect`` : { bool, int, str }, default: true

  If `basename` is set, then this controls how the response is handled
  -- see the `index-redirect` option for accepted values, with the
  adjustment that the default redirect location is the `fullname`.

* ``{PREFIX}.inspect`` : str, default: /

  Specifies the top-level URL to start the application inspection at.

  TODO: this does not work.

  WARNING: this does not work.

  SERIOUSLY: this does not work, it only adds the specified path as a
  URL prefix... doh!

* ``{PREFIX}.include`` : list(regex-spec), default: null

  The `include` option lists encapsulated regular expressions that an
  endpoint must match at least one of in order to be included in the
  output. This option can be used with the `exclude` option, in which
  case endpoints are first matched for inclusion, then matched for
  exclusion (i.e. the order is "allow,deny" in apache terminology).

  Encapsulated regular expressions are expressed in the syntax
  "/EXPR/FLAGS", where the "/" can be replaced by any character
  otherwise not found in the rest of the expression. The flags can
  be any combination of the following characters:

  * ``i``: Case-insensitive matching.
  * ``l``: Use locale-dependent processing (for \w, \W, etc.).
  * ``m``: Multi-line mode, i.e. "^" and "$" match individual lines.
  * ``s``: The "." matches newlines as well.
  * ``u``: Use the unicode properties db (for \w, \W, etc.).
  * ``x``: Allow verbose regular expressions.

  Example:

  .. code-block:: ini

    describe.include = :^/api/:i :^/foo(/.*)?$:
    describe.exclude = :.*/private(/.*)?$:i

* ``{PREFIX}.exclude`` : list(regex-spec), default: null

  The inverse of the `include` option -- see `include` for details.

* ``{PREFIX}.entries.filters`` : list(resolve-spec), default: null

  This option specifies a callable (or string in python dot syntax) or
  list of thereof that filter and modify the entries before they are
  rendered to the requested format. Each entry that is selected for
  inclusion for rendering is first passed through each filter and
  replaced by the return value from the call. This is done for each
  filter consecutively. If any filter returns ``None``, the entry is
  removed from the selection list.

  These filters are intended to allow two primary features:

  * Access control: a filter can inspect the entry and the requesting
    user and determine if the entry should be made visible. If not, it
    should return ``None``.

  * Custom documentation parsing: a filter can parse the entry's `doc`
    attribute (which gets auto-populated with the entry's python
    documentation string), and extract other information such as
    expected parameters, return values, and exceptions thrown.
    Typically, this is done with something like numpydoc_.

  Filters are passed two parameters: an `entry` object (see
  pyramid_describe.entry.Entry for detailed attributes) and an
  `options` dictionary. The latter has many interesting attributes,
  including a reference to the current `request`.

  TODO: add documentation about `entry` and `options`.

  Note that there is a *separate* `filters` option that is used to
  filter the entire output document, which is format-specific. See
  the formatting options for details.

* ``{PREFIX}.formats`` : list(str), default: ['html', 'txt', 'pdf', 'rst', 'json', 'yaml', 'wadl', 'xml']

  Specifies the list of formats that can be generated. The default
  list includes all supported built-in formats, but this can be
  extended by adding a format to this list and then specifying a
  template to render the format. For example:

  .. code-block:: ini

    # declare support for HTML, JSON and SWF
    describe.formats = html json swf

    # HTML and JSON are built-in, but SWF needs a custom template
    describe.format.swf.renderer = mypackage:templates/describe-swf.mako

  Note that the "pdf" and "yaml" formats require that optional python
  package dependencies be installed (respectively `pdfkit` and
  `PyYAML`), and that pdfkit_ furthermore requires that the
  wkhtmltopdf_ program be available.

* ``{PREFIX}.format.default`` : str, default: first format listed in `{PREFIX}.formats`

  Set the default format if not specified in the request.

* ``{PREFIX}.format.{FORMAT}.renderer`` : asset-spec, default: 'pyramid_describe:template/{FORMAT}.mako'

  Override the default renderer for the specified format using a
  pyramid-style asset specification. The default is to use the
  pyramid-describe template with the exception of the structured
  data formats (JSON, YAML, XML, and WADL), which do not use a
  template.

  Specifying a renderer pre-empts all other rendering fallback
  mechanisms.

  See `Format Cascading`_ for details on how the `{FORMAT}` string is
  evaluated.

* ``{PREFIX}.format.request`` : { bool, list(str) }, default: false

  Specifies which options, if any, can be controlled by request
  parameters. The setting can either be a boolean ("true", "false",
  etc), or a list of options. If truthy, all options can be
  specified. If falsy, no options can be specified. Otherwise it
  is interpreted as a space-separated list of options that can be
  specified.

  Note that this setting can be overridden on a per-format basis
  by the `format.{FORMAT}.request` setting.

* ``{PREFIX}.format.{FORMAT}.request`` : { bool, list(str) }, default: none

  The per-format version of `format.request`. Note that this
  completely overrides the `format.request` setting for the
  given format, it does not extend it.

  See `Format Cascading`_ for details on how the `{FORMAT}` string is
  evaluated.

* ``{PREFIX}.format.default.{OPTION}``

  Set a default rendering option for all formats. Note that this can
  be overridden by request parameters (see the `format.request`
  option). See the `Format Options`_ section for a list of all
  supported options.

* ``{PREFIX}.format.override.{OPTION}``

  Set a rendering option for all formats that overrides any request
  parameters. See the `Format Options`_ section for a list of all
  supported options.

* ``{PREFIX}.format.{FORMAT}.default.{OPTION}``

  Set a default rendering option for the specified format, which
  overrides any default value set for all formats. Note that this can
  be overridden by request parameters (see the `format.request`
  option). See the `Format Options`_ section for a list of all
  supported options.

  See `Format Cascading`_ for details on how the `{FORMAT}` string is
  evaluated.

* ``{PREFIX}.format.{FORMAT}.override.{OPTION}``

  Set a rendering option for the specified format that overrides any
  request parameters and any generic format override options. See the
  `Format Options`_ section for a list of all supported options.

  See `Format Cascading`_ for details on how the `{FORMAT}` string is
  evaluated.


Format Cascading
================

Some formats are rendered based on the output of other renderers. For
example, PDF's are generated from HTML, and HTML is in turn generated
from reStructuredText. Because options may need to be different for
the the various formats based on the ultimate output, there is the
ability to specify "cascaded" formats by joining them with a "+" in
the settings. The cascaded options can either be explicitly overriden
or explicitly reverted to their system-wide default by setting them to
the special value ``pyramid_describe:DEFAULT``.

Therefore, options for format "rst" apply to the reStructuredText
rendering, regardless of ultimate output. Options for format
"rst+html" apply to reStructuredText rendering, but only if the next
renderer is "html". These can be chained to any depth, for example
options for format "rst+html+pdf" apply to reStructuredText rendering,
but only if the next renderer is "html" followed by "pdf". Note that
one cannot skip a renderer in a rendering pipeline, e.g. in the
previous case, you cannot short-hand the format as "rst+pdf".

For example, the following configuration will apply a different CSS to
the HTML rendering based on whether the output is going to be HTML,
PDF, or SWF:

.. code-block:: ini

   # the following sets the `cssPath` option for *any* HTML rendering:
   describe.format.html.default.cssPath = mypkg:style/rst2html.css

   # this now overrides the `cssPath` option during rendering of the
   # HTML, but only in the context of a PDF rendering:
   describe.format.html+pdf.default.cssPath = mypkg:style/rst2pdf.css

   # when generating SWFs, this tells the describer to revert to using
   # the system default value of `cssPath`:
   describe.format.html+swf.default.cssPath = pyramid_describe:DEFAULT


Format Options
==============

* ``title`` : str, default: 'Contents of "{PATH}"'
* ``endpoints.title`` : str, default: 'Endpoints'
* ``legend.title`` : str, default: 'Legend'

* ``showUnderscore`` : bool, default: false
* ``showUndoc`` : bool, default: true
* ``showLegend`` : bool, default: true
* ``showBranches`` : bool, default: false
* ``pruneIndex`` : bool, default: true
* ``showRest`` : bool, default: true
* ``showImpl`` : bool, default: false
* ``showInfo`` : bool, default: true
* ``showName`` : bool, default: true
* ``showDecorated`` : bool, default: true
* ``showExtra`` : bool, default: true
* ``showMethods`` : bool, default: true
* ``showIds`` : bool, default: true
* ``showDynamic`` : bool, default: true
* ``showGenerator`` : bool, default: true
* ``showGenVersion`` : bool, default: true
* ``showLocation`` : bool, default: true
* ``ascii`` : bool, default: false
* ``maxdepth`` : int, default: 1024
* ``width`` : int, default: 79
* ``maxDocColumn`` : int, default: null
* ``minDocLength`` : int, default: 20

* ``cssEmbed`` : bool, default: true
* ``cssPath`` : { asset-spec, resolve-spec, list({ asset-spec, resolve-spec }) }, default: 'pyramid_describe:template/rst2html.css'

* ``rstMax`` : bool, default: false
* ``rstPdfkit`` : bool, default: true

* ``stubFormat`` : str, default: '{{{}}}'
* ``dynamicFormat`` : str, default: '{}/?'
* ``restFormat`` : str, default: '<{}>'

* ``pdfkit.options`` : str

  This option is YAML-parsed, and then sets the options that are
  inserted into the HTML meta tags that are instructions to the pdfkit
  processor. The default values specified by pyramid-describe are:

  .. code-block:: text

    {
      margin-top: 10mm,
      margin-right: 10mm,
      margin-bottom: 10mm,
      margin-left: 10mm,
    }

  Options not specified revert to the defaults specified by pdfkit.
  For details, see `pdfkit <https://pypi.python.org/pypi/pdfkit>`_
  and `wkhtmltopdf <http://code.google.com/p/wkhtmltopdf/>`_. Options
  that may be of interest:

  * grayscale
  * page-size
  * orientation
  * no-outline
  * print-media-type
  * disable-plugins
  * zoom
  * javascript-delay
  * disable-javascript

* ``restVerbs`` : list(str), default: pyramid_controllers.restcontroller.HTTP_METHODS

  Sets the list of known HTTP methods. This is used during inspection
  to determine whether a given exposed method on a RestController can
  be accessed via an HTTP method.

* ``filters`` : list(resolve-spec), default: null

  Unlike the top-level `entries.filters` setting which filters
  individual entries as they get selected for rendering, the
  format-specific `filters` option is called on the entire data object
  before final rendering, and is very format-specific in what is made
  available.

  For the structured-data formats (JSON, YAML, XML, and WADL), the
  filters are provided the data object created by
  Describer.structure_render. Each filter is expected to return that
  object (enhanced in some way), or a new object to replace it.

  For RST and HTML, the filters are provided a
  `docutils.nodes.document` object (as is returned by
  `docutils.core.publish_doctree
  <http://docutils.sourceforge.net/docs/api/publisher.html#publish-doctree>`_).

  For PDF, rendering is accomplished from entries to RST to HTML to
  PDF. Therefore, the filtering occurs during the RST to HTML
  transformation -- there is no separate PDF-only filtering. If
  filtering is needed at one of the previous stages that is required
  only during PDF generation (but not, for example, to HTML), then the
  `formatstack` option can be inspected, which will include ``'pdf'``
  during the HTML filtering. For example:

  .. code-block:: python

    def html_filter(doc, stage):
      if 'pdf' in stage.options.formatstack:
        # do PDF-specific filtering...
      else:
        # do filtering for everything except PDF...
      return doc

  TODO: add documentation about `data` and `options`.

* ``encoding`` : str, default: 'UTF-8'

.. _pyramid-controllers: https://pypi.python.org/pypi/pyramid_controllers
.. _numpydoc: https://github.com/numpy/numpy/blob/master/doc/HOWTO_DOCUMENT.rst.txt
.. _pdfkit: https://pypi.python.org/pypi/pdfkit
.. _wkhtmltopdf: http://code.google.com/p/wkhtmltopdf/
