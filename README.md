# @allnulled/documentator

To document projects using documentator comments. Very simple tool.

## Index

- [@allnulled/documentator](#allnulleddocumentator)
  - [Index](#index)
  - [Installation](#installation)
  - [Usage](#usage)
  - [Best practices](#best-practices)
  - [Comment types](#comment-types)
    - [Multiline comments](#multiline-comments)
    - [Oneline comments](#oneline-comments)
  - [Special properties](#special-properties)
  - [Workflow](#workflow)
  - [Custom configurations](#custom-configurations)
  - [Overview](#overview)
  - [Conclusion](#conclusion)

## Installation

```sh
git clone ... .
npm install
```

- Use `git`, and not `npm`, because you'll need the whole structure for advanced features.
- Even so, to fully automate work, you need `libretranslate` running on port 5000.
  - And I am not gonna do a script to make this work.
  - I think I did in previous chapters, in some repo.

## Usage

```js
const comments = require("@allnulled/documentator").parse_directory(".", {
    fileformats: [
        ".js",
        ".css",
        ".html"
    ],
    ignored: [
        "/node_modules/",
        "/docs/",
    ],
    output_dir: "docs/reference",
    pipes: ["generate_html"]
});
```

- Only synchronous API, so avoid to use it in production.

## Best practices

- Delete `test` folder as you are not gonna use it, and the tool is (presumptly) enough tested.

## Comment types

### Multiline comments

Multiline comment requires `/**` to start and `*/` to end, minimum. It has no conditions inside, but:
  - it will clean indentation (if starts with spaces and then `*`) and not compute it.
  - it will accumulate as string all the comment properties that...
    - follow from the begining of the line `@{ whatever inside }:`)
  - the default text (without assigned property) goes to the property `$comment`
  - the properties that start with `$` are not inserted in the documentation

```js
/**
 * Text to $comment.
 * 
 * @property 1: value 1.
 * @property 2: value 2.
 * @property 3: value 3.
 * @property 4: value 4.
 * @property 5: value 5.
 * @property 5: this is following the previous property.
 * 
 */
```

### Oneline comments

Oneline comment requires `//` and `\n` or end of file. It can do the same as the multiline, but:
  - it uses `|` instead of new line to separate properties.

```js
// @prop1: val1 | @property 2: value 2
```

This type of comment requires to start by one property, so `// @something: something` is the minimum.

## Special properties

The are some special properties.

- `$output`: specifies the file to push this comment into. By default `index.html`.
- `$section`: specifies the section to push this comment into. It is a lower category of `$output` (file) in the book. By default `0. Prelude`.
  - Sections are sorted alphabetically: name and prefix-numerate them consequently.
- `$priority`: specifies the importance of this comment in the section. The higher, the sooner it will appear in the section. By default `0`.
- `$reference`: with this property you can write the documentation out of the code, and the `documentator` will take care to inject it again later.
- `$nokey.*`: When you start a property with `nokey.`, the `documentator` will not print it. This is useful to separate blocks inside 1 comment.

## Workflow

On 1 hand, you want to work seeing the results automatically. For this:

```sh
npm run dev
```

On the other hand, maybe, you want to get the translations automatically too. For this:

```sh
libretranslate # must be running on port 5000, and on other script:
npm run autotranslate
```

With this, you get all the translations automatically generated. And on the other hand, the `npm run dev` will separate the strings it catches, so you can write in your preferred language.

## Custom configurations

- You can play with the parameters of the `parse_directory` function.
- You can change the main language you want to use, at `config.json#main_language`.
  - By default is set to `es`.
  - This parameter is used in different places, so you have to provide it independently.


## Overview

Note that documentator is not only a parser.

Its power lives in the `parse_directory` function, where the parser is only a required feature to complet the whole process.

This means that you can change from the parser, that is true. But the pipe it gives is so ambitious, that it is rare that you need to do this: it maybe is that you want to, but not that you need to.

The tool provides a pipe for:

 - Recursive multiline and one-line comments finder
 - Allowed to be externalized (`$reference` or `reference`)
 - Allowed to be saved in different files (`$output` or `output`)
 - Allowed to be saved in different sections (`$section` or `section`)
 - Allowed to be saved with different priorities inside the same (file and) section (`$priority` or `priority`)
 - Internationalizable
   - Automatically internationalizable to more than 70 languages
     - Maybe that way, but what you expect from me, to hire translators or what?
 - Markable (coliving with internationalizability, thanks to Markdown)
   - So you can, still internationalizing, inject styles through Markdown.
     - Note that comments that contain HTML openers or closers (&gt; or &lt;), are not internationalized.
     - But you still can use Markdown syntax to have a bit of styling.
 - Injectable
   - Because nothing privates you from injecting your own HTML from the comments
 - Renderizable
   - I am not missing anything from this feature, but the piping allows to render EJS templates to.
   - So, note that your documentation is symbolically conditioned. 
     - But not limited, because you have special escaping characters in HTML yet.
     - And also note that your code will be correctly escaped when injected as HTML.

The pipe is a bit complex. The code is spread in the builder, the pipes, and the client needed to request the translations, available languages, and keymaps of the translations.

The pipe in the client is the one that will give you the best view:

From the client-side, all the translations pass through this method:

```js
  get_translation(key_brute, element) {
    const key = keymapper_json[key_brute] || key_brute;
    let translation = key;
    Layer_1_Get_translation: {
      try {
        translation = this.translations[this.current_language_iso][key];
        if (typeof translation !== "string") {
          throw new Error("Translation not found");
        }
      } catch (error) {
        console.info("[documentatori18n][key-not-found][key=" + key + "]");
        translation = key;
      }
    }
    // @CAUTION: HACKY POINT!
    let translation_rendered = translation;
    Layer_2_Apply_ejs_rendering: {
      try {
        translation_rendered = window.ejs.render(translation_rendered, { element });
      } catch (error) {
        console.error("[documentatori18n][key-rendering-error][key=" + key + "]");
        console.log(error);
      }
    }
    let translation_corrected = translation_rendered;
    Layer_4_Corret_typos: {
      try {
        // translation_corrected = translation_corrected.replace(/\.( |\n|\r|\t)*\.( |\n|\r|\t)*$/g, ".");
      } catch (error) {
        console.error("[documentatori18n][key-correcting-error][key=" + key + "]");
        console.log(error);
      }
    }
    let translation_marked = translation_corrected;
    Layer_3_Transform_markdown: {
      try {
        translation_marked = window.marked.parse(translation_marked);
      } catch (error) {
        console.error("[documentatori18n][key-marking-error][key=" + key + "]");
        console.log(error);
      }
    }
    console.log("Corrected: " + translation_marked);
    return translation_marked;
  },
```

So the order is: 

  - Get the translation
  - Render as EJS
  - Correct typos (none right now)
  - Transform markdown to HTML

This way, all your strings are automatically **translated** and **marked**.

The order is important, because we could not do automatic translation if we marked the text before.

## Conclusion

So, in the end, Markdown seems to have a non-practical, but technical, reason to exist. And it is a half-syntactical half-programmatical reason. And this is: to allow auto-translation colive with marked text in one same flux of piping.

Nice, I never heard of that. But I can see it now.

Okay.

This tool, finally, aims to quit the job of *documentation + translation + markup*, as a big framework to automate all the process, this way you can forget about this bunch of work.

Basically.# documentator
