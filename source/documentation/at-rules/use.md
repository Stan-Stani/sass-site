---
title: "@use"
table_of_contents: true
eleventyComputed:
  before_introduction: >
    {% render 'doc_snippets/module-system-status' %}
introduction: >
  The `@use` rule loads [mixins](/documentation/at-rules/mixin),
  [functions](/documentation/at-rules/function), and
  [variables](/documentation/variables) (also referred to as members) from other Sass stylesheets, and
  always includes selectors from the referenced stylesheets (SCSS or CSS). Stylesheets loaded by `@use`
  are called "modules". Sass also provides [built-in
  modules](/documentation/modules) full of useful functions.
---

The simplest `@use` rule is written `@use "<url>"`, which loads the module at
the given URL. Any styles loaded this way will be included exactly once in the
compiled CSS output, no matter how many times those styles are loaded.

{% headsUp %}
  A stylesheet's `@use` rules must come before any rules other than `@forward`,
  including [style rules][]. However, you can declare variables before `@use`
  rules to use when [configuring modules][].

  [style rules]: /documentation/style-rules
  [configuring modules]: #configuration
{% endheadsUp %}

{% codeExample 'use' %}
  // foundation/_code.scss
  code {
    padding: .25em;
    line-height: 0;
  }
  ---
  // foundation/_lists.scss
  ul, ol {
    text-align: left;

    & & {
      padding: {
        bottom: 0;
        left: 0;
      }
    }
  }
  ---
  // style.scss
  @use 'foundation/code';
  @use 'foundation/lists';
  ===
  // foundation/_code.sass
  code
    padding: .25em
    line-height: 0
  ---
  // foundation/_lists.sass
  ul, ol
    text-align: left

    & &
      padding:
        bottom: 0
        left: 0
  ---
  // style.sass
  @use 'foundation/code'
  @use 'foundation/lists'
  ===
  code {
    padding: .25em;
    line-height: 0;
  }

  ul, ol {
    text-align: left;
  }
  ul ul, ol ol {
    padding-bottom: 0;
    padding-left: 0;
  }
{% endcodeExample %}

## Loading Members

You can access variables, functions, and mixins from another module by writing
`<namespace>.<variable>`, `<namespace>.<function>()`, or `@include
<namespace>.<mixin>()`. By default, the namespace is just the last component of
the module's URL.

{% headsUp %}
  Top-level selectors are included automatically and cannot be explicitly accessed
  like members.
{% endheadsUp %}

Members (variables, functions, and mixins) loaded with `@use` are only visible
in the stylesheet that loads them. Other stylesheets will need to write their
own `@use` rules if they also want to access them. This helps make it easy to
figure out exactly where each member is coming from. If you want to load members
from many files at once, you can use the [`@forward` rule][] to forward them all
from one shared file.

[`@forward` rule]: /documentation/at-rules/forward

{% funFact %}
  Because `@use` adds namespaces to member names, it's safe to choose very
  simple names like `$radius` or `$width` when writing a stylesheet. This is
  different from the old [`@import` rule][], which encouraged that users write
  long names like `$mat-corner-radius` to avoid conflicts with other libraries,
  and it helps keep your stylesheets clear and easy to read!

  [`@import` rule]: /documentation/at-rules/import
{% endfunFact %}

{% codeExample 'loading-members' %}
  // src/_corners.scss
  $radius: 3px;

  @mixin rounded {
    border-radius: $radius;
  }
  ---
  // style.scss
  @use "src/corners";

  .button {
    @include corners.rounded;
    padding: 5px + corners.$radius;
  }
  ===
  // src/_corners.sass
  $radius: 3px

  @mixin rounded
    border-radius: $radius
  ---
  // style.sass
  @use "src/corners"

  .button
    @include corners.rounded
    padding: 5px + corners.$radius
  ===
  .button {
    border-radius: 3px;
    padding: 8px;
  }
{% endcodeExample %}

### Choosing a Namespace

By default, a module's namespace is just the last component of its URL without a
file extension. However, sometimes you might want to choose a different
namespace—you might want to use a shorter name for a module you refer to a lot,
or you might be loading multiple modules with the same filename. You can do this
by writing `@use "<url>" as <namespace>`.

{% codeExample 'namespace' %}
  // src/_corners.scss
  $radius: 3px;

  @mixin rounded {
    border-radius: $radius;
  }
  ---
  // style.scss
  @use "src/corners" as c;

  .button {
    @include c.rounded;
    padding: 5px + c.$radius;
  }
  ===
  // src/_corners.sass
  $radius: 3px

  @mixin rounded
    border-radius: $radius
  ---
  // style.sass
  @use "src/corners" as c

  .button
    @include c.rounded
    padding: 5px + c.$radius
  ===
  .button {
    border-radius: 3px;
    padding: 8px;
  }
{% endcodeExample %}

You can even load a module *without* a namespace by writing `@use "<url>" as *`.
We recommend you only do this for stylesheets written by you, though; otherwise,
they may introduce new members that cause name conflicts!

{% codeExample 'use-url' %}
  // src/_corners.scss
  $radius: 3px;

  @mixin rounded {
    border-radius: $radius;
  }
  ---
  // style.scss
  @use "src/corners" as *;

  .button {
    @include rounded;
    padding: 5px + $radius;
  }
  ===
  // src/_corners.sass
  $radius: 3px

  @mixin rounded
    border-radius: $radius
  ---
  // style.sass
  @use "src/corners" as *

  .button
    @include rounded
    padding: 5px + $radius
  ===
  .button {
    border-radius: 3px;
    padding: 8px;
  }
{% endcodeExample %}

### Private Members

As a stylesheet author, you may not want all the members you define to be
available outside your stylesheet. Sass makes it easy to define a private member
by starting its name with either `-` or `_`. These members will work just like
normal within the stylesheet that defines them, but they won't be part of a
module's public API. That means stylesheets that load your module can't see
them!

{% funFact %}
  If you want to make a member private to an entire *package* rather than just a
  single module, just don't [forward its module][] from any of your package's
  entrypoints (the stylesheets you tell your users to load to use your package).
  You can even [hide that member][] while forwarding the rest of its module!

  [forward its module]: /documentation/at-rules/forward
  [hide that member]: /documentation/at-rules/forward#controlling-visibility
{% endfunFact %}

{% codeExample 'private-members', false %}
  // src/_corners.scss
  $-radius: 3px;

  @mixin rounded {
    border-radius: $-radius;
  }
  ---
  // style.scss
  @use "src/corners";

  .button {
    @include corners.rounded;

    // This is an error! $-radius isn't visible outside of `_corners.scss`.
    padding: 5px + corners.$-radius;
  }
  ===
  // src/_corners.sass
  $-radius: 3px

  @mixin rounded
    border-radius: $-radius
  ---
  // style.sass
  @use "src/corners"

  .button
    @include corners.rounded

    // This is an error! $-radius isn't visible outside of `_corners.scss`.
    padding: 5px + corners.$-radius
{% endcodeExample %}

## Configuration

A stylesheet can define variables with the [`!default` flag][] to make them
configurable. To load a module with configuration, write `@use <url> with
(<variable>: <value>, <variable>: <value>)`. The configured values will override
the variables' default values.

[`!default` flag]: /documentation/variables#default-values

{% render 'code_snippets/example-use-with' %}

### With Mixins

Configuring modules with `@use ... with` can be very handy, especially when
using libraries that were originally written to work with the [`@import`
rule][]. But it's not particularly flexible, and we don't recommend it for more
advanced use-cases. If you find yourself wanting to configure many variables at
once, pass [maps][] as configuration, or update the configuration after the
module is loaded, consider writing a mixin to set your variables instead and
another mixin to inject your styles.

[`@import` rule]: /documentation/at-rules/import
[maps]: /documentation/values/maps

{% codeExample 'with-mixins' %}
  // _library.scss
  $-black: #000;
  $-border-radius: 0.25rem;
  $-box-shadow: null;

  /// If the user has configured `$-box-shadow`, returns their configured value.
  /// Otherwise returns a value derived from `$-black`.
  @function -box-shadow() {
    @return $-box-shadow or (0 0.5rem 1rem rgba($-black, 0.15));
  }

  @mixin configure($black: null, $border-radius: null, $box-shadow: null) {
    @if $black {
      $-black: $black !global;
    }
    @if $border-radius {
      $-border-radius: $border-radius !global;
    }
    @if $box-shadow {
      $-box-shadow: $box-shadow !global;
    }
  }

  @mixin styles {
    code {
      border-radius: $-border-radius;
      box-shadow: -box-shadow();
    }
  }
  ---
  // style.scss
  @use 'library';

  @include library.configure(
    $black: #222,
    $border-radius: 0.1rem
  );

  @include library.styles;
  ===
  // _library.sass
  $-black: #000
  $-border-radius: 0.25rem
  $-box-shadow: null

  /// If the user has configured `$-box-shadow`, returns their configured value.
  /// Otherwise returns a value derived from `$-black`.
  @function -box-shadow()
    @return $-box-shadow or (0 0.5rem 1rem rgba($-black, 0.15))


  @mixin configure($black: null, $border-radius: null, $box-shadow: null)
    @if $black
      $-black: $black !global
    @if $border-radius
      $-border-radius: $border-radius !global
    @if $box-shadow
      $-box-shadow: $box-shadow !global


  @mixin styles
    code
      border-radius: $-border-radius
      box-shadow: -box-shadow()
  ---
  // style.sass
  @use 'library'
  @include library.configure($black: #222, $border-radius: 0.1rem)
  @include library.styles
  ===
  code {
    border-radius: 0.1rem;
    box-shadow: 0 0.5rem 1rem rgba(34, 34, 34, 0.15);
  }
{% endcodeExample %}

### Reassigning Variables

After loading a module, you can reassign its variables.

{% codeExample 'reassigning-variables', false %}
  // _library.scss
  $color: red;
  ---
  // _override.scss
  @use 'library';
  library.$color: blue;
  ---
  // style.scss
  @use 'library';
  @use 'override';
  @debug library.$color;  //=> blue
  ===
  // _library.sass
  $color: red
  ---
  // _override.sass
  @use 'library'
  library.$color: blue
  ---
  // style.sass
  @use 'library'
  @use 'override'
  @debug library.$color  //=> blue
{% endcodeExample %}

This even works if you import a module without a namespace using `as *`.
Assigning to a variable name defined in that module will overwrite its value in
that module.

{% headsUp %}
  Built-in module variables (such as [`math.$pi`]) cannot be reassigned.

  [`math.$pi`]: /documentation/modules/math#$pi
{% endheadsUp %}

## Finding the Module

It wouldn't be any fun to write out absolute URLs for every stylesheet you load,
so Sass's algorithm for finding a module makes it a little easier. For starters,
you don't have to explicitly write out the extension of the file you want to
load; `@use "variables"` will automatically load `variables.scss`,
`variables.sass`, or `variables.css`.

{% headsUp %}
  To ensure that stylesheets work on every operating system, Sass loads files by
  *URL*, not by *file path*. This means you need to use forward slashes, not
  backslashes, even on Windows.
{% endheadsUp %}

### Load Paths

All Sass implementations allow users to provide *load paths*: paths on the
filesystem that Sass will look in when locating modules. For example, if you
pass `node_modules/susy/sass` as a load path, you can use `@use "susy"` to load
`node_modules/susy/sass/susy.scss`.

Modules will always be loaded relative to the current file first, though. Load
paths will only be used if no relative file exists that matches the module's
URL. This ensures that you can't accidentally mess up your relative imports when
you add a new library.

{% funFact %}
  Unlike some other languages, Sass doesn't require that you use `./` for
  relative imports. Relative imports are always available.
{% endfunFact %}

### Partials

As a convention, Sass files that are only meant to be loaded as modules, not
compiled on their own, begin with `_` (as in `_code.scss`). These are called
*partials*, and they tell Sass tools not to try to compile those files on their
own. You can leave off the `_` when importing a partial.

### Index Files

If you write an `_index.scss` or `_index.sass` in a folder, the index file will
be loaded automatically when you load the URL for the folder itself.

{% codeExample 'index-files' %}
  // foundation/_code.scss
  code {
    padding: .25em;
    line-height: 0;
  }
  ---
  // foundation/_lists.scss
  ul, ol {
    text-align: left;

    & & {
      padding: {
        bottom: 0;
        left: 0;
      }
    }
  }
  ---
  // foundation/_index.scss
  @use 'code';
  @use 'lists';
  ---
  // style.scss
  @use 'foundation';
  ===
  // foundation/_code.sass
  code
    padding: .25em
    line-height: 0
  ---
  // foundation/_lists.sass
  ul, ol
    text-align: left

    & &
      padding:
        bottom: 0
        left: 0
  ---
  // foundation/_index.sass
  @use 'code'
  @use 'lists'
  ---
  // style.sass
  @use 'foundation'
  ===
  code {
    padding: .25em;
    line-height: 0;
  }

  ul, ol {
    text-align: left;
  }
  ul ul, ol ol {
    padding-bottom: 0;
    padding-left: 0;
  }
{% endcodeExample %}

## Loading CSS

In addition to loading `.sass` and `.scss` files, Sass can load plain old `.css`
files.

{% codeExample 'loading-css' %}
  // code.css
  code {
    padding: .25em;
    line-height: 0;
  }
  ---
  // style.scss
  @use 'code';
  ===
  // code.css
  code {
    padding: .25em;
    line-height: 0;
  }
  ---
  // style.sass
  @use 'code'
  ===
  code {
    padding: .25em;
    line-height: 0;
  }
{% endcodeExample %}

CSS files loaded as modules don't allow any special Sass features and so can't
expose any Sass variables, functions, or mixins. In order to make sure authors
don't accidentally write Sass in their CSS, all Sass features that aren't also
valid CSS will produce errors. Otherwise, the CSS will be rendered as-is. It can
even be [extended][]!

[extended]: /documentation/at-rules/extend

## Differences From `@import`

The `@use` rule is intended to replace the old [`@import` rule], but it's
intentionally designed to work differently. Here are some major differences
between the two:

* `@use` only makes variables, functions, and mixins available within the scope
  of the current file. It never adds them to the global scope. This makes it
  easy to figure out where each name your Sass file references comes from, and
  means you can use shorter names without any risk of collision.

* `@use` only ever loads each file once. This ensures you don't end up
  accidentally duplicating your dependencies' CSS many times over.

* `@use` must appear at the beginning your file, and cannot be nested in style
  rules.

* Each `@use` rule can only have one URL.

* `@use` requires quotes around its URL, even when using the [indented syntax].

[`@import` rule]: /documentation/at-rules/import
[indented syntax]: /documentation/syntax#the-indented-syntax
