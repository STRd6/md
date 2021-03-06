Like a Doctor
=============

Document all of your literate code: files that end in `.md`.

Inspired by Docco. Designed for the browser environment.

We use marked for generating the markdown.

    marked = require "./lib/marked"
    highlight = require "./lib/highlight"
    languages = require "./languages"

    marked.setOptions
      highlight: (code, lang) ->
        if highlight.LANGUAGES[lang]
          highlight.highlight(lang, code).value
        else
          console.warn "couldn't highlight code block with unknown language '#{lang}'"

          code

Export our public api.

    module.exports = doctor =
      parse: require('./parse')

Our docco style template.

      template: require('./template')

Document one file.

      compile: (content, language="coffeescript") ->
        doctor.parse(content).map ({text, code}) ->
          docsHtml: marked(text)
          codeHtml: marked "```#{language}\n#{code}\n```"

Generate the documentation for all files within the given package. Returns a
promise that will be fulfilled with an array of `fileData`.

      documentAll: (pkg) ->
        {entryPoint, source, repository} = pkg
        {branch, default_branch} = repository

        if branch is "blog" # HACK
          base = ""
        else if branch is default_branch
          base = "docs/"
        else
          base = "#{branch}/docs/"

        documentableFiles = Object.keys(source).filter (name) ->
          extension(name) is "md"

        results = documentableFiles.map (name) ->
          language = extension(withoutExtension(name))
          language = languages[language] || language

          doctor.compile source[name].content, language

        extras = [packageScript(base, pkg)]

        scripts = dependencyScripts unique([
          "https://code.jquery.com/jquery-1.10.1.min.js"
          "https://cdnjs.cloudflare.com/ajax/libs/coffee-script/1.6.3/coffee-script.min.js"
        ].concat(
          pkg.remoteDependencies or []
        ))

        # Add interactive loader to scripts string
        scripts += interactiveLoader

        results = results.map (result, i) ->
          # Assuming .*.md so we should strip the extension twice
          name = withoutExtension(withoutExtension(documentableFiles[i]))

          content = doctor.template
            title: name
            sections: result
            scripts:  "#{scripts}#{makeScript(relativeScriptPath(name))}"

          # Add an index.html if our file is the entry point
          if name is entryPoint
            extras.push
              content: doctor.template
                title: "index"
                sections: result
                scripts:  "#{scripts}#{makeScript(relativeScriptPath("index"))}"
              mode: "100644"
              path: "#{base}index.html"
              type: "blob"

          content: content
          mode: "100644"
          path: "#{base}#{name}.html"
          type: "blob"

        Promise.resolve(extras.concat(results))

Helpers
-------

`makeScript` returns a string representation of a script tag that has a src
attribute.

    makeScript = (src) ->
      script = document.createElement("script")
      script.src = src

      return script.outerHTML

`dependencyScripts` returns a string containing the script tags that are
the dependencies of this build.

    dependencyScripts = (remoteDependencies=[]) ->
      remoteDependencies.map(makeScript).join("\n")

`unique` returns a new duplicate free version of an array.

    unique = (array) ->
      array.reduce (results, item) ->
        results.push item if results.indexOf(item) is -1

        results
      , []

Include the interactive docs loader, this connection is a bit tenuous.

    interactiveLoader = """
      <script>
        #{PACKAGE.dependencies.interactive.distribution.interactive.content}
      <\/script>
    """

This returns a script file that exposes a global `require` that gives access to
the current package and is meant to be included in every docs page.

    packageScript = (base, pkg) ->
      content: """
        (function(pkg) {
          #{PACKAGE.dependencies.require.distribution.main.content}
          window.require = Require.generateFor(pkg);
        })(#{JSON.stringify(pkg, null, 2)});
      """
      mode: "100644"
      path: "#{base}package.js"
      type: "blob"

Package Script path

    relativeScriptPath = (path) ->
      upOne = "../"
      results = []

      levels = (path.split("/").length - 1)
      if levels > 0
        [0...levels].forEach ->
          results.push upOne

      results.concat("package.js").join("")

File extension for string

    extension = (str) ->
      if match = str.match(/\.([^\.]*)$/, '')
        match[match.length - 1]
      else
        ''

    withoutExtension = (str) ->
      str.replace(/\.[^\.]*$/,"")
