---
layout: default
type: guide
shortname: Docs
title: Styling local DOM
subtitle: Developer guide
---

{% include toc.html %}


Polymer uses [Shadow DOM styling
rules](http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/) for
providing scoped styling of the element's local DOM.  Scoped styles should be
provided via `<style>` tags placed inside the element's local DOM `<template>`.


    <dom-module id="my-element">



      <template>

        <style>
          :host {
            display: block;
            border: 1px solid red;
          }
          #child-element {
            background: yellow;
          }
          /* styling elements distributed to content (via ::content) requires */
          /* selecting the parent of the <content> element for compatibility with */
          /* shady DOM . This can be :host or a wrapper element. */
          .content-wrapper > ::content .special {
            background: orange;
          }
        </style>

        <div id="child-element">In local DOM!</div>
        <div class="content-wrapper"><content></content></div>

      </template>

      <script>

          Polymer({
              is: 'my-element'
          });

      </script>

    </dom-module>

To place styles outside of the element, or share styles between elements, you can create
a [style module](#style-modules).

**Note:**  Prior to Polymer 1.1, the recommendation was to place `<style>` tags
inside the `<dom-module>` for an element (but _outside_ the `<template>`). This
 is still supported, but is no longer recommended.
{: .alert .alert-info }


### Styling distributed children (::content)

Under shady DOM, the `<content>` tag doesn't appear in the DOM tree. Styles are rewritten to remove the
`::content` pseudo-element, **and any combinator immediately to the left of `::content`.**

This implies:

*   You must have a selector to the left of the `::content` pseudo-element.

        :host ::content div

    Becomes:

        x-foo div

    (Where `x-foo` is the name of the custom element.)

*   To limit styles to elements inside the ::content tag, add a wrapper element around the
    `<content>` element. This is especially important when using a child combinator (`>`) to
    select top-level children.

        <dom-module id="my-element">

          <template>

            <style>
              .content-wrapper > ::content .special {
                background: orange;
              }
            </style>

            <div class="content-wrapper"><content></content></div>

          </template>

        </dom-module>

    In this case, the rule:

        .content-wrapper ::content > .special

    Becomes:

        .content-wrapper > .special



## Cross-scope styling {#xscope-styling}


### Background

Shadow DOM (and its approximation via Shady DOM) bring much needed benefits of
scoping and style encapsulation to web development, making it safer and easier
to reason about the effects of CSS on parts of your application.  Styles do not
leak into the local DOM from above, and styles do not leak from one local DOM
into the local DOM of other elements inside.

This is great for *protecting* scopes from unwanted style leakage.  But what
about when you intentionally want to *customize* the style of a custom element's
local DOM, as the user of an element?  This often comes up under the umbrella of
"theming".  For example a "custom-checkbox" element that may internally use a
`.checked` class can protect itself from being affected by CSS from other
components that may also happen to use a `.checked` class.  However, as the user
of the checkbox you may wish to intentionally change the color of the check to
match your product's branding, for example.  The same "protection" that Shadow
DOM provides at the same time introduces a practical barrier to "theming" use
cases.

One solution the Shadow DOM spec authors provided to address the theming problem
are the `/deep/` and `::shadow` combinators, which allow writing rules that
pierce through the Shadow DOM encapsulation boundary.  Although Polymer 0.5
promoted this mechanism for theming, it was ultimately unsatisfying for several
reasons:

*   Using `/deep/` and `::shadow` for theming leaks details of an otherwise
    encapsulated element to the user, leading to brittle selectors piercing into
    the internal details of an element's Shadow DOM that are prone to breakage
    when the internal implementation changes.  As a result, the structure of of
    an element's Shadow DOM inadvertently becomes API surface subject to
    breakage, diminishing the practical effectiveness of Shadow DOM as an
    encapsulation primitive.

*   Although Shadow DOM's style encapsulation *improves* the predictability of
    style recalc performance since the side effects of a style change are
    limited to a small subset of the document, using `/deep/` and `::shadow`
    re-opens the style invalidation area and reduces Shadow DOM's effectiveness as a
    performance primitive.

*   Using `/deep/` and `::shadow` leads to verbose and difficult to understand
    selectors.

For the reasons above, the Polymer team is currently exploring other options for
theming that address the shortcomings above and provide a possible path to
obsolescence of `/deep/` and `::shadow` altogether.

### Custom CSS properties {#xscope-styling-details}

Polymer includes a shim for custom CSS properties inspired by (and compatible with)
the future W3C [CSS Custom Properties for Cascading Variables](http://dev.w3.org/csswg/css-variables/)
specification (see
[Using CSS Variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_variables)
on the Mozilla Developer Network).

Rather than exposing the details of an element's internal implementation for
theming, instead an element author defines one or more custom CSS
properties as part of the element's API.

These custom properties can be defined similarly to other standard CSS properties
and will inherit from the point of definition down the composed DOM tree,
similar to the effect of `color` and `font-family`.

In the simple example below, the author of `my-toolbar` identified the need for
users of the toolbar to be able to change the color of the toolbar title.  The
author exposed a custom property called `--my-toolbar-title-color` which is
assigned to the `color` property of the selector for the title element.  Users
of the toolbar may define this variable in a CSS rule anywhere up the tree, and
the value of the property will inherit down to the toolbar where it is used if
defined, similar to other standard inheriting CSS properties.

Example:

    <dom-module id="my-toolbar">

      <template>

        <style>
          :host {
            padding: 4px;
            background-color: gray;
          }
          .title {
            color: var(--my-toolbar-title-color);
          }
        </style>

        <span class="title">{%raw%}{{title}}{%endraw%}</span>

      </template>

      <script>
        Polymer({
          is: 'my-toolbar',
          properties: {
            title: String
          }
        });
      </script>

    </dom-module>

Example usage of `my-toolbar`:

    <dom-module id="my-element">


      <template>

        <style>

          /* Make all toolbar titles in this host green by default */
          :host {
            --my-toolbar-title-color: green;
          }

          /* Make only toolbars with the .warning class red */
          .warning {
            --my-toolbar-title-color: red;
          }

        </style>

        <my-toolbar title="This one is green."></my-toolbar>
        <my-toolbar title="This one is green too."></my-toolbar>

        <my-toolbar class="warning" title="This one is red."></my-toolbar>

      </template>

      <script>
        Polymer({ is: 'my-element'});
      </script>

    </dom-module>

The `--my-toolbar-title-color` property only affects the color of the title
element encapsulated in `my-toolbar`'s internal implementation.  In the
future the `my-toolbar` author can rename the `title` class or
restructure the internal details of `my-toolbar` without changing the custom
property exposed to users.

You can also include a default value in the `var()` function, to use in case the user
doesn't set the custom property:

    color: var(--my-toolbar-title-color, blue);

Thus, custom CSS properties introduce a powerful way for element authors to
expose a theming API to their users in a way that naturally fits right alongside
normal CSS styling and avoids the problems with `/deep/` and `::shadow`. It is
already on a standards track with shipping implementation by Mozilla and planned
support by Chrome.

### Custom CSS mixins

It may be tedious (or impossible) for an element author to predict every
CSS property that may be important for theming, let alone expose every
property individually.

The custom properties shim includes an extension that enables an element
author to define a set of CSS properties as a single custom property and
then allow all properties in the set to be applied to a specific CSS rule
in an element's local DOM. The extension enables this with a mixin capability
that is analogous to `var`, but which allows an entire set of properties
to be mixed in. This extension adheres to the
[CSS @apply rule](http://tabatkins.github.io/specs/css-apply-rule/)
proposal.

Use `@apply` to apply a mixin:

<pre>@apply(--<var>mixin-name</var>);</pre>

Defining a mixin is just like defining a custom property, but the
value is an object that defines one or more rules:

<pre><var>selector</var> {
  --<var>mixin-name</var>: {
    /* rules */
  };
}</pre>

Example:

    <dom-module id="my-toolbar">

      <template>

        <style>
          :host {
            padding: 4px;
            background-color: gray;
            /* apply a mixin */
            @apply(--my-toolbar-theme);
          }
          .title {
            @apply(--my-toolbar-title-theme);
          }
        </style>

        <span class="title">{%raw%}{{title}}{%endraw%}</span>

      </template>

      ...

    </dom-module>

Example usage of `my-toolbar`:

    <dom-module id="my-element">

      <template>

        <style>
          /* Apply custom theme to toolbars */
          :host {
            --my-toolbar-theme: {
              background-color: green;
              border-radius: 4px;
              border: 1px solid gray;
            };
            --my-toolbar-title-theme: {
              color: green;
            };
          }

          /* Make only toolbars with the .warning class red and bold */
          .warning {
            --my-toolbar-title-theme: {
              color: red;
              font-weight: bold;
            };
          }
        </style>

        <my-toolbar title="This one is green."></my-toolbar>
        <my-toolbar title="This one is green too."></my-toolbar>

        <my-toolbar class="warning" title="This one is red."></my-toolbar>

      </template>

      <script>
        Polymer({ is: 'my-element'});
      </script>

    </dom-module>


### Custom property API for {{site.project_title}} elements {#style-api}

{{site.project_title}}'s custom property shim evaluates and applies custom property values once
at element creation time.  In order to have an element (and its subtree) re-
evaluate custom property values due to dynamic changes such as application of
CSS classes, etc., call `this.updateStyles()` on the element.
To update all elements on the page, you can also call `Polymer.updateStyles()`.

You can also directly modify a {{site.project_title}} element's custom property by setting
key-value pairs in `customStyle` on the element (analogous to setting `style`)
and then calling `updateStyles()`. To get the value of a custom
property on an element, use `getComputedStyleValue(propertyName)`.



Example:

    <dom-module id="x-custom">

      <template>

        <style>
          :host {
            --my-toolbar-color: red;
          }
        </style>

        <my-toolbar>My awesome app</my-toolbar>
        <button on-tap="changeTheme">Change theme</button>

      </template>

      <script>
        Polymer({
          is: 'x-custom',
          changeTheme: function() {
            this.customStyle['--my-toolbar-color'] = 'blue';
            this.updateStyles();
          }
        });
      </script>

    </dom-module>

### Custom properties shim—limitations and API details

Cross-platform support for custom properties is provided in Polymer by a
JavaScript library that approximates the capabilities of the CSS Variables
specification  *for the specific use case of theming custom elements*, while
also extending it to add the capability to mixin property sets to rules as
described above.  **It is important to note that this is not a full polyfill**,
as doing so would be prohibitively expensive; rather this is a shim that is
inspired by that specification and trades off aspects of the full dynamism
possible in CSS with practicality and performance.

Below are current limitations of this system. Improvements to performance and
dynamism will continue to be explored.

*   Only rules which match the element at *creation time* are applied. Any dynamic
    changes that update variable values are not applied automatically.

    HTML:

        <div class="container">
          <x-foo class="a"></x-foo>
        </div>

    CSS:

        /* applies */
        x-foo.a {
          --foo: brown;
        }
        /* does not apply */
        x-foo.b {
          --foo: orange;
        }
        /* does not apply to x-foo */
        .container {
          --nog: blue;
        }

*   Re-evaluation of custom property styles does not currently occur as a result
    of changes to the DOM.  Re-evaluation can be forced by calling
    `this.updateStyles()` on a Polymer element (or `Polymer.updateStyles()` to
    update all element styles).  For example, if class `b` was added to `x-foo`
    above, the scope must call `this.updateStyles()` to apply the styling. This
    re-calcs/applies styles down the tree from this point.

*   Dynamic effects are reflected at the point of a variable’s application, but not its definition.

    For the following example, adding/removing the `highlighted` class on the `#title` element will
    have the desired effect, since the dynamism is related to *application* of a custom property.

        #title {
          background-color: var(--title-background-normal);
        }

        #title.highlighted {
          background-color: var(--title-background-highlighted);
        }

    However, the shim does not currently support dynamism at the point of
    *definition* of a custom property.  In the following example,
    `this.updateStyles()` would be required to update the value of `--title-
    background` being applied to `#title` when the `highlighted` class was added
    or removed.

        #title {
          --title-background: gray;
        }

        #title.highlighted {
          --title-background: yellow;
        }

*   Unlike normal CSS inheritance which flows from parent to child, custom
    properties in Polymer's shim can only change when inherited by a custom element
    from rules that set properties in scope(s) above it, or in a `:host` rule for
    that scope.  Within a given element's local DOM scope, a custom property can
    only have a single value.  Calculating property changes within a scope would be
    prohibitively expensive for the shim and are not required to achieve cross-scope
    styling for custom elements, which is the primary goal of the shim.

        <dom-module id="my-element">

          <template>

            <style>
             :host {
               --custom-color: red;
             }
             .container {
               /* Setting the custom property here will not change */
               /* the value of the property for other elements in  */
               /* this scope.                                      */
               --custom-color: blue;
             }
             .child {
               /* This will be always be red. */
               color: var(--custom-color);
             }
            </style>

            <div class="container">
              <div class="child">I will be red</div>
            </div>

          </template>

          <script>
            Polymer({ is: 'my-element'});
          </script>

        </dom-module>


## Custom element for document styling (custom-style) {#custom-style}


An experimental `<style is="custom-style">` custom element is provided for defining
styles in the main document that can take advantage of several special features
of Polymer's styling system:

*   Document styles defined in a `custom-style` will be shimmed to ensure they do
    not leak into local DOM when running on browsers without native Shadow
    DOM.

*   Shadow DOM-specific `/deep/` and `::shadow` combinators will be shimmed on
    browsers without native Shadow DOM.

*   Custom properties used by Polymer's experimental
    [shim for cross-scope styling](#xscope-styling-details) may be defined in an
    `custom-style`. Use the `:root` selector to define custom properties that apply
    to all custom elements.

Example:

    <!doctype html>
    <html>
    <head>
      <script src="components/webcomponentsjs/webcomponents-lite.js"></script>
      <link rel="import" href="components/polymer/polymer.html">

      <style is="custom-style">

        /* Will be prevented from affecting local DOM of Polymer elements */
        * {
          box-sizing: border-box;
        }

        /* Can use /deep/ and ::shadow combinators */
        body /deep/ .my-special-view::shadow #thing-inside {
          background: yellow;
        }

        /* Custom properties that inherit down the document tree may be defined */
        :root {
          --my-toolbar-title-color: green;
        }

      </style>

    </head>
    <body>

        ...

    </body>
    </html>

All features of `custom-style` are available when defining styles as part of
Polymer elements (for example, in `<style>` elements within a custom element's
`<dom-module>`). The exception is the `:root` selector, which is only useful at
the document level. **The `custom-style` extension should only be used for
defining document styles, outside of a custom element's local DOM.**

## Shared styles and external stylesheets {#style-modules}

To share style declarations between elements, you can package a set
of style declarations inside a `<dom-module>` element. In this section,
a `<dom-module>` holding styles is called a _style module_ for convenience.

A style module declares a named set of style rules that can be imported into
an element definition, or into a `custom-style` element.

**Note:** Style modules were introduced in {{site.project_title}} 1.1;
they replace the experimental support for [external stylesheets](#external-stylesheets).
{: .alert .alert-info }

Define a style module inside an HTML import using the `<dom-module>`
element.

    <!-- shared-styles.html -->
    <dom-module id="shared-styles">
      <template>
        <style>
          .red { color: red; }
        </style>
      </template>
    </dom-module>

The `id` attribute specifies the name you'll use to reference
your shared styles. Style module names use the same namespace as elements,
so your style modules must have unique names.

Using the shared styles is a two-step process: you need to use a `<link>` tag
to _import_ the module, and a `<style>` tag to _include_ the styles in the correct place.

To use a style module in an element:

    <!-- import the module  -->
    <link rel="import" href="../shared-styles/shared-styles.html">
    <dom-module id="x-foo">
      <template>
        <!-- include the style module by name -->
        <style include="shared-styles"></style>
        <style>:host { display: block; }</style>
        Hi
      </template>
      <script>Polymer({is: 'x-foo'});</script>
    </dom-module>

You can also use a shared style module in a `custom-style` element.

    <!-- import the shared styles  -->
    <link rel="import" href="../shared-styles/shared-styles.html">
    <!-- include the shared styles -->
    <style is="custom-style" include="shared-styles"></style>

A single style tag can both `include` shared styles
and define local rules:

    <style include="shared-styles">
      :host { display: block; }
    </style>

(This works for both `custom-style` elements and `<style>` tags inside
custom elements.) The shared styles are applied _before_ the styles defined
inside the body of the `<style>` tag, so the shared styles can be overridden
by the styles defined in the body.

### External stylesheets (deprecated) {#external-stylesheets}

**Note:** This experimental feature is now deprecated in favor of
[style modules](#style-modules). It is still supported, but support will
be removed in the future.
{: .alert .alert-info }

Polymer includes an experimental feature to support loading external stylesheets
that will be applied to the local DOM of an element.  This is typically
convenient for developers who like to separate styles, share common styles
between elements, or use style pre-processing tools.  The syntax is slightly
different from how stylesheets are typically loaded, as the feature leverages
HTML Imports (or the HTML Imports polyfill, where appropriate) to load the
stylesheet text such that it may be properly shimmed and/or injected as an
inline style.

To include a remote stylesheet that applies to your Polymer element's local DOM,
place a special HTML import `<link>` tag with `type="css"` in your `<dom-
module>` that refers to the external stylesheet to load.

Example:

    <dom-module id="my-awesome-button">

      <!-- special import with type=css used to load remote CSS -->
      <link rel="import" type="css" href="my-awesome-button.css">

      <template>
        ...
      </template>

      <script>
        Polymer({
          is: 'my-awesome-button',
          ...
        });
      </script>

    </dom-module>

To include an external stylesheet at the document level that includes
local DOM aware rules, use a _standard_ HTML import to import a document
that includes a `custom-style` element.

`index.html`:

    <html>
    <head>
      <script src="bower_components/webcomponentsjs/webcomponents-lite.min.js"></script>
      <link rel="import" href="my-custom-styles.html">
    </head>
      ...

`my-custom-styles.html`:

    <style is="custom-style">
      html /deep/ iron-icon {
        color: red;
      }
    </style>

## Third-party libraries that modify local DOM {#scope-subtree}

If you are using a third-party library that adds local DOM nodes to your
Polymer element, you may notice that styles on the element do not update
properly.

The correct way to add DOM nodes to a Polymer element's local DOM is via
the Polymer DOM API. This API lets you manipulate nodes in a way that respects
the local DOM and ensures that styles are updated properly.

When using third-party libraries that do not know about the Polymer DOM
API, use `scopeSubtree` to apply proper CSS scoping to a node and all of its
descendants.

    scopeSubtree(containerNode, false);

`containerNode` is the root node of the tree you wish to scope. Setting
the second argument to `false` scopes the specified node and descendants
once. Setting it to `true` enables a mutation observer that applies CSS
scoping whenever `containerNode` or any of its descendants are modified.

Example:

    ready: function() {
      this.scopeSubtree(this.$.container, true);
    }