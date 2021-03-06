---
template: overrides/main.html
---

# Setting up site search

Material for MkDocs provides a great, client-side search implementation,
omitting the need for the integration of third-party services, which might
violate data privacy regulations. Furthermore, with some effort, search can
be made available [offline][1].

  [1]: #offline-search

## Configuration

### Built-in search

[:octicons-file-code-24: Source][2] · 
[:octicons-cpu-24: Plugin][3]

The [built-in search plugin][3] integrates seamlessly with Material for MkDocs,
adding multilingual client-side search with [lunr][4] and [lunr-languages][5].
It's enabled by default, but must be re-added to `mkdocs.yml` when other plugins
are used:

``` yaml
plugins:
  - search
```

The following options are supported:

`lang`{: #lang }

:   :octicons-milestone-24: Default: _automatically set_ – This option allows
    to include the language-specific stemmers provided by [lunr-languages][5].
    Note that Material for MkDocs will set this automatically based on the
    [site language][6], but it may be overriden, e.g. to support multiple
    languages:

    === "A single language"

        ``` yaml
        plugins:
          - search:
              lang: ru
        ```

    === "Multiple languages"

        ``` yaml
        plugins:
          - search:
              lang:
                - en
                - ru
        ```

    The following languages are supported:

    <ul class="tx-columns">
      <li><code>ar</code> – Arabic</li>
      <li><code>da</code> – Danish</li>
      <li><code>du</code> – Dutch</li>
      <li><code>en</code> – English</li>
      <li><code>fi</code> – Finnish</li>
      <li><code>fr</code> – French</li>
      <li><code>de</code> – German</li>
      <li><code>hu</code> – Hungarian</li>
      <li><code>it</code> – Italian</li>
      <li><code>ja</code> – Japanese</li>
      <li><code>no</code> – Norwegian</li>
      <li><code>pt</code> – Portuguese</li>
      <li><code>ro</code> – Romanian</li>
      <li><code>ru</code> – Russian</li>
      <li><code>es</code> – Spanish</li>
      <li><code>sv</code> – Swedish</li>
      <li><code>th</code> – Thai</li>
      <li><code>tr</code> – Turkish</li>
      <li><code>vi</code> – Vietnamese</li>
    </ul>

    _Material for MkDocs also tries to support languages which are not part of
    this list, by automatically chosing the best-matching stemmer_.

    !!! warning "Only specify the languages you really need"

        Be aware that including support for other languages increases the general
        JavaScript payload by around 20kb (before `gzip`) and by another 15-30kb
        per language.

`separator`{: #separator }

:   :octicons-milestone-24: Default: _automatically set_ – The separator for
    indexing and query tokenization can be customized, which makes it possible
    to index parts of words that are separated by other characters than
    whitespace and `-`, e.g. by including `.`:

    ``` yaml
    plugins:
      - search:
          separator: '[\s\-\.]+'
    ```

`prebuild_index`{: #prebuild-index }

:   :octicons-milestone-24: Default: `false` · :octicons-beaker-24:
    Experimental – MkDocs can generate a [prebuilt index][7] of all pages during
    build time, which provides performance improvements at the cost of more
    bandwidth, as it reduces the build time of the search index:

    ``` yaml
    plugins:
      - search:
          prebuild_index: true
    ```
    
    This may be beneficial for large documentation projects served with
    appropriate headers, i.e. `Content-Encoding: gzip`, but benchmarking before
    deployment is recommended.

_Material for MkDocs doesn't provide official support for the other options of
this plugin, so they may be supported but can also yield weird results. Use
them at your own risk._

  [2]: https://github.com/squidfunk/mkdocs-material/tree/master/src/assets/javascripts/integrations/search
  [3]: https://www.mkdocs.org/user-guide/configuration/#search
  [4]: https://lunrjs.com
  [5]: https://github.com/MihaiValentin/lunr-languages
  [6]: changing-the-language.md#site-language
  [7]: https://www.mkdocs.org/user-guide/configuration/#prebuild_index

### Offline search

[:octicons-file-code-24: Source][8] · 
[:octicons-cpu-24: Plugin][9] · :octicons-beaker-24: Experimental

If you distribute your documentation as `*.html` files, the built-in search
will not work out-of-the-box due to the restrictions modern browsers impose for
security reasons. This can be mitigated with the [localsearch plugin][9] in
combination with @squidfunk's [iframe-worker][10] polyfill.

For setup instructions, refer to the [official documentation][11].

  [8]: https://github.com/squidfunk/mkdocs-material/blob/master/src/base.html#L378-L390
  [9]: https://github.com/wilhelmer/mkdocs-localsearch/
  [10]: https://github.com/squidfunk/iframe-worker
  [11]: https://github.com/wilhelmer/mkdocs-localsearch#installation-material-v5

## Customization

The search implementation of Material for MkDocs is probably its most
sophisticated feature, as it tries to balance a _great typeahead experience_,
_good performance_, _accessibility_ and a result list that is _easy to scan_.
This is where it deviates from other themes.

This section explains how search can be customized to tailor it to your needs.

### Query transformation

[:octicons-file-code-24: Source][12] · 
:octicons-mortar-board-24: Difficulty: _easy_

When a user enters a query into the search box, the query is pre-processed
before it is submitted to the search index. Material for MkDocs will apply the
following transformations, which can be customized by [extending the theme][13]:

``` ts
/**
 * Default transformation function
 *
 * 1. Search for terms in quotation marks and prepend a `+` modifier to denote
 *    that the resulting document must contain all terms, converting the query
 *    to an `AND` query (as opposed to the default `OR` behavior). While users
 *    may expect terms enclosed in quotation marks to map to span queries, i.e.
 *    for which order is important, `lunr` doesn't support them, so the best
 *    we can do is to convert the terms to an `AND` query.
 *
 * 2. Replace control characters which are not located at the beginning of the
 *    query or preceded by white space, or are not followed by a non-whitespace
 *    character or are at the end of the query string. Furthermore, filter
 *    unmatched quotation marks.
 *
 * 3. Trim excess whitespace from left and right.
 *
 * 4. Append a wildcard to the end of every word to make every word a prefix
 *    query in order to provide a good typeahead experience, by adding an
 *    asterisk (wildcard) in between terms, which can be denoted by whitespace,
 *    any non-control character, or a word boundary.
 *
 * @param value - Query value
 *
 * @return Transformed query value
 */
function defaultTransform(value: string): string {
  return value
    .split(/"([^"]+)"/g)                            /* => 1 */
      .map((terms, i) => i & 1
        ? terms.replace(/^\b|^(?![^\x00-\x7F]|$)|\s+/g, " +")
        : terms
      )
      .join("")
    .replace(/"|(?:^|\s+)[*+\-:^~]+(?=\s+|$)/g, "") /* => 2 */
    .trim()                                         /* => 3 */
    .replace(/\s+|(?![^\x00-\x7F]|^)$|\b$/g, "* ")  /* => 4 */
}
```

If you want to switch to the default behavior of the `mkdocs` or `readthedocs`
template, both of which don't transform the query prior to submission, or
customize the `transform` function, you can do this by [overriding the 
`config` block][14]:

``` jinja
{% block config %}
  <script>
    var search = {
      transform: function(query) {
        return query
      }
    }
  </script>
{% endblock %}
```

The `transform` function will receive the query string as entered by the user
and must return the processed query string to be submitted to the search index.

  [12]: https://github.com/squidfunk/mkdocs-material/blob/master/src/assets/javascripts/integrations/search/transform/index.ts
  [13]: ../customization.md#extending-the-theme
  [14]: ../customization.md#overriding-blocks

### Custom search

[:octicons-file-code-24: Source][15] · 
:octicons-mortar-board-24: Difficulty: _challenging_

Material for MkDocs implements search as part of a [web worker][16]. If you
want to switch the web worker with your own implementation, e.g. to submit
search to an external service, you can add a custom JavaScript file to the `docs`
directory and [override the `config` block][14]:

``` jinja
{% block config %}
  <script>
    var search = {
      worker: "<url>"
    }
  </script>
{% endblock %}
```

Communication with the search worker is implemented using a standardized
message format using _discriminated unions_, i.e. through the `type` property
of the message. See the following interface definitions to learn about the
message formats:

- [:octicons-file-code-24: `SearchMessage`][17]
- [:octicons-file-code-24: `SearchIndex` and `SearchResult`][18]

The sequence and direction of messages is rather intuitive:

- :octicons-arrow-right-24: `SearchSetupMessage` 
- :octicons-arrow-left-24: `SearchReadyMessage`
- :octicons-arrow-right-24: `SearchQueryMessage` 
- :octicons-arrow-left-24: `SearchResultMessage`

  [15]: https://github.com/squidfunk/mkdocs-material/blob/master/src/assets/javascripts/integrations/search/worker
  [16]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers
  [17]: https://github.com/squidfunk/mkdocs-material/blob/master/src/assets/javascripts/integrations/search/worker/message/index.ts
  [18]: https://github.com/squidfunk/mkdocs-material/blob/master/src/assets/javascripts/integrations/search/_/index.ts
