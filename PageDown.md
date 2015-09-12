# Introduction #

PageDown is the JavaScript [Markdown](http://daringfireball.net/projects/markdown/) previewer used on [Stack Overflow](http://stackoverflow.com/) and the rest of the [Stack Exchange network](http://stackexchange.com/). It includes a Markdown-to-HTML converter and an in-page Markdown editor with live preview.

While on the Stack Exchange sites PageDown is exclusively used on the client site (on the server side, [MarkdownSharp](http://code.google.com/p/markdownsharp/) is our friend), PageDown's converter also works in a server environment. So if your Node.JS application lets users enter Markdown, you can also use PageDown to convert it to HTML on the server side.

The largest part is based on work by John Fraser, a.k.a. Attacklab. He created the converter under the name _Showdown_ and the editor under the name _WMD_. See [this post on the Stack Exchange blog](http://blog.stackoverflow.com/2008/12/reverse-engineering-the-wmd-editor/) for some historical information.

Over the years, we (the Stack Exchange team and others) have made quite a few changes, bug fixes etc., which is why we decided to publish the whole thing under a new name. This is not to mean we want to deprive John Fraser of any credits; he deserves lots. And you'll still be finding the letters "wmd" in a few places.

It should be noted that Markdown is **not safe as far as user-entered input goes**. Pretty much anything is valid in Markdown, in particular something like `<script>doEvil();</script>`. This PageDown repository includes the two plugins that Stack Exchange uses to sanitize the user's input; see the description of Markdown.Sanitizer.js below.

In the [demo/ folder](http://code.google.com/p/pagedown/source/browse/#hg%2Fdemo), it also include usage examples. To see the editor at work, open [demos/browsers/demo.html](http://pagedown.googlecode.com/hg/demo/browser/demo.html) in your browser. To see the server side version at work, go to the [demos/node folder](http://code.google.com/p/pagedown/source/browse/#hg%2Fdemo%2Fnode) and run `node demo.js`, then navigate your browser to http://localhost:8000.

# node.js and npm #

The easiest way to get the Markdown converter working in node.js is by getting it [through npm](https://npmjs.org/package/pagedown).

```
    $ npm install pagedown
    [...snip...]
    $ node
    > var pagedown = require("pagedown");
    > var converter = new pagedown.Converter();
    > var safeConverter = pagedown.getSanitizingConverter();
    > converter.makeHtml("*hello*")
    '<p><em>hello</em></p>'
    > safeConverter.makeHtml("Hello <script>doEvil();</script>")
    >'<p>Hello doEvil();</p>'
```

If you don't want to use npm, then follow the instructions below.


# The three main files #

## Markdown.Converter.js ##

### In the browser ###

Used in a regular browser environment, it provides a top-level object called `Markdown` with two properties:

**`Markdown.Converter`** is a constructor that creates a converter object. Call this object's `makeHtml` method to turn Markdown into HTML:

```
    var converter = new Markdown.Converter();

    document.write(converter.makeHtml("**I am bold!**"));
```

The constructor optionally takes an `OPTIONS` object as an argument.

The only currently recognized option is `OPTIONS.nonAsciiLetters`. If this is truthy, the converter will work around JavaScript's lack of unicode support in regular expressions when handling bold and italic. For example, since intra-word emphasis is prevented, the asterisks in `w**or**d` will not cause bolding. However without the workaround, the same is not true if the letters are outside the ASCII range: `с**ло́в**о` will cause the middle part of the word to be bolded.

If the workaround is enabled, latin and cyrillic will behave identically here.

**`Markdown.HookCollection`** is a constructor that creates a very simple plugin hook collection. You can usually ignore it; it's exported because the Markdown Editor (see below) uses it as well.

### On the server ###

If you're using it in a CommonJS environment (we'll just assume Node.JS from now on), those two properties are the module's exports.

```
    var Converter = require("./Markdown.Converter").Converter;
    var converter = new Converter();

    console.log(converter.makeHtml("**I am bold!**"));
```

## Markdown.Sanitizer.js ##

### In the browser ###

In a browser environment, this file has to be included after including Markdown.Converter.js. It adds a function named `getSanitizingConverter` to the `Markdown` object (created in the previous file).

This function returns a converter object that has two plugins registered on the `postConversion` plugin hook. One of them sanitizes the output HTML to only include a list of whitelisted HTML tags. The other one attempts to balance opening/closing tags to prevent leaking formating if the HTML is displayed in the browser.

```
    var converter = getSanitizingConverter();
    document.write(converter.makeHtml("<script>alert(42);</script><b>bold"); // creates "alert(42);bold"
```

### On the server ###

Same thing, except that `getSanitizingConverter` is exported, and you don't need to load Markdown.Converter.js.

```
    var converter = require("./Markdown.Sanitizer").getSanitizingConverter();
    console.log(converter.makeHtml("<script>alert(42);</script><b>bold"); // creates "alert(42);bold"
```

## Markdown.Editor.js ##

This file is only made for usage in the browser (obviously running it on the server neither makes sense nor works).

It has to be included after Markdown.Converter.js, as it requires the top-level `Markdown` object and its `HookCollection` property. If you're not using the provided Markdown converter, you'll have to change this file to include these two things.

The file adds the property `Markdown.Editor`, which is a constructor taking up to three arguments:

  * The first argument is required, and is the Markdown converter object (as created above) used for the editor's preview.

  * The second argument is optional, and is usually only necessary if you're using several editors within the same page. If given, this argument is a string, appended to the HTML element ids of the three elements used by the editor.
> By default, the editor looks for `#wmd-button-bar`, `#wmd-input`, and `#wmd-preview`. If you're using more than one editor, you of course can't give the second group of elements the same ids as the first, so you may create the second input box as `<textarea id="wmd-input-2">` and pass the string `"-2"` as the second argument to the constructor.

  * The third argument is optional as well, and if given, is an object containing information about the "Help" button offered to the user. The object needs a `handler` property, which will be the `onclick` handler of the help button. It can also have a `title` property, which is then set as the tooltip (i.e. `title` attribute) of the help button. If not given, the `title` defaults to "Markdown Editing Help".

> If the third argument isn't passed, no help button will be created.

The created editor object has (up to) three methods:

  * **`editor.getConverter()`** returns the converter object that was passed to the constructor.
  * **`editor.run()`** starts running the editor; you should call this _after_ you've added your plugins to the editor (if any).
  * **`editor.refreshPreview()`** forces the editor to re-render the preview according to the contents of the input, e.g. after you have programmatically changed the input box content. This method is only available after `editor.run()` has been called.

To start using it, you'll probably want to look at the example in the demos/browser folder; it also includes some boilerplate CSS for you to start working with.

# Plugin hooks #

Both the converter and the editor objects have a property called `hooks`, which is a collection of plugin hooks. The functionality is of the category "simplest thing that works", so don't expect too much magic. The two (important) methods on this object are:

  * **`hooks.set(hookname, func)`** registers the given function as a plugin on the given hook. Any previously registered plugins on the same hook are lost.
  * **`hooks.chain(hookname, func)`** registers the given function as the next plugin on the given hook; e.g. the second registered function  will be called with the output of the first registered function as its only argument. This does not make sense on all plugin hooks.

Following is a list of the available plugin hooks.

## Hooks on the converter object ##

### `preConversion` ###

Called with the Markdown source as given to the converter's `makeHtml` object, should return the actual to-be-converted source. Fine to chain.

```
    converter.hooks.chain("preConversion", function (text) {
        return "# Converted text follows\n\n" + text; // creates a level 1 heading
    });
```

### `postConversion` ###

Called with the HTML that was created from the Markdown source. The return value of this hook is the actual output that will then be returned from `makeHtml`. This is where `getSanitizingConverter` (see above) registers the tag sanitzer and balancer. Fine to chain.

```
    converter.hooks.chain("postConversion", function (text) {
        return text + "<br>\n**This is not bold, because it was added after the conversion**";
    });
```

### `plainLinkText` ###

When the original Markdown contains a bare URL, it will by default be converted to a link whose link text is the URL itself. If you want a different link text, you can register a function on the hook that receives the URL as its only argument, and returns the text to be displayed. This will not change the actual URL (i.e. target of the link).

Note that the returned string will be inserted verbatim, not HTML-encoded in any way.

Okay to chain, although this may or may not make sense (after all you're receiving a URL and returning just about anything).

```
    converter.hooks.chain("plainLinkText", function (url) {
        if (/^http:\/\/\w+\.stackexchange.com/i.test(url))
            return "<b>A link to an awesome site!</b>";
        else
            return "some page on the internet";
    });
```

### _Notice on the following hooks_ ###

The following converter plugin hooks should be considered advanced. In order to use them, it makes sense to have a certain understanding how PageDown (or similar Markdown implementations that are based on the original Perl version) works internally. They are much more powerful than the previous ones, but it's also easier to break things with them. Documentation here is kept to a minimum; you should inspect the code around the hooks.

### `postNormalization` ###

Called with the Markdown source after normalization. This includes things like converting all line endings to `\n`, replacing tabs with spaces, and turning whitespace-only lines into empty lines. But it also includes replacing certain characters with placeholders for internal reasons, so be sure to have a look at the actual code. This hook is fine to chain.

### `preBlockGamut` and `postBlockGamut` ###

The above warning about understandind the inner workings of PageDown goes doubly with these two hooks. They are the most powerful ones.

Called with the text before or after creating block elements like code blocks and lists. Fine to chain. Note that this is called recursively with inner content, e.g. it's called with the full text, and then only with the content of a blockquote. The inner call will receive outdented text

If you are creating a new kind of block-level structure that can include other Markdown blocks (say, you're reimplementing block quote), you will need to run the full block gamut again on the contents of your block. For this purpose, these two plugin hooks receive a second argument, which is a function that will do just that. Be aware that from within that function, your plugin will of course be called again, so make sure you're not creating ambiguity that leads to infinite recursion.

As a contrived and simplified expample, let's say you're inventing fenced blockquotes, where blocks that are surrounded by lines with three quote characters in them are turned into block quotes. Such a thing could look like this:

```
    converter.hooks.chain("preBlockGamut", function (text, runBlockGamut) {
        return text.replace(/^ {0,3}""" *\n((?:.*?\n)+?) {0,3}""" *$/gm, function (whole, inner) {
            return "<blockquote>" + runBlockGamut(inner) + "</blockquote>\n";
        });
    });
```

The first editor in the [demo page](http://pagedown.googlecode.com/hg/demo/browser/demo.html) implements this.

### `preSpanGamut` and `postSpanGamut` ###

Called with the text of a single block element before or after the span-level conversions (like bold, code spans, etc.) have been made on it. For example, you would use this if you wanted to have text `between !!double exclamation points!! appear` in red.

## Hooks on the editor object ##

Remember to add all your necessary plugins _before_ you call the editor's `run()` method.

### `onPreviewRefresh` ###

Called with no arguments after the editor's preview has been updated. No return value is expected. Fine to chain, if you want to be notified of a refresh in several places.

```
    editor.hooks.chain("onPreviewRefresh", function () {
        console.log("the preview has been updated");
    });
```

### `postBlockquoteCreation` ###

Called after the user has clicked the "Blockquote" button (or pressed Ctrl-Q) and the user's selection has been changed accordingly. This function is passed the content that would be inserted in place of the original selection, and should return the actual to-be-inserted content.

The name isn't 100% correct, as this hook is also called if a blockquote is _removed_ using this button. Fine to chain (but probably unlikely to be used anyway).

```
    editor.hooks.chain("postBlockquoteCreation", function (text) {
        if (!/^>/.test(text))
            return text; // the blockquote button was clicked to remove a blockquote -- no change

        return text + "\n\nThe above blockquote is brought to you by PunyonDew(tm)\n\n"
    });
```

### `insertImageDialog` ###

When the user clicks the "add image" button, they usually get a little dialog to enter the URL of an image. If you want a different behavior (like, in the case of Stack Exchange, a dialog to upload an image), register a plugin on this hook that returns `true` (this tells the editor not to create its own dialog and instead wait for you). The function is called with a single argument, which is a callback that you should call with the URL of the to-be-inserted image, or `null` if the user cancelled.

It does _not_ make sense to chain functions on this hook.

```
    editor.hooks.set("insertImageDialog", function (callback) {
        alert("Please click okay to start scanning your brain...");
        setTimeout(function () {
            var prompt = "We have detected that you like cats. Do you want to insert an image of a cat?";
            if (confirm(prompt))
                callback("http://icanhascheezburger.files.wordpress.com/2007/06/schrodingers-lolcat1.jpg")
            else
                callback(null);
        }, 2000);
        return true; // tell the editor that we'll take care of getting the image url
    });
```

Note that you cannot call the callback directly from the hook; you have to wait for the current scope to be exited. If for any reason you want to return a link immediately, you'll have to use a 0ms timeout:

```
    editor.hooks.set("insertImageDialog", function (callback) {
        setTimeout(function () { callback("http://example.com/image.jpg"); }, 0);
        return true;
    });
```