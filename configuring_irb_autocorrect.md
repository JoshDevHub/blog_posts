# Configuring IRB autocomplete

Since the release of Rails 7, the Rails console comes with a super cool autocomplete feature that provides suggestions to you as you type. This is much like the intellisense and snippets you get while writing code in an editor. But let's take a look at what this autocomplete looks like out of the box:

![IRB autocompletion](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ebvtdtf0gxvzfrr8giuj.png)

Yuck.

This might look okay for you depending on the color setup of your terminal, but for me the contrast is almost poor enough to render it unusable. Fortunately this can be changed! Let's explore how.

### Time for a Facelift

Until recently there was no good way to configure IRB's autocomplete styling. But a few months ago, a [`Reline::Face`](https://github.com/ruby/reline/pull/552) class was added to IRB that offers a simple public API for configuring the colors of the autocomplete dialog.

Using this, I can get it looking *much* nicer:

![IRB better autocomplete](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/oe210sre8gk7awcyb31g.png)

To start with, you'll want to create a new file in your home directory called `.irbrc`. The contents of this file are executed before an IRB session starts, so you can use it to configure certain properties of your IRB sessions.

```sh
# to create the file
touch ~/.irbrc

# to edit the file
code ~/.irbrc # or `vim` or `subl` or whatever you use
```

In this file, I have the following code set to configure the autocompletion dialog:

```ruby
# configure autocomplete dialog colors
if defined? Reline::Face
  Reline::Face.config(:completion_dialog) do |conf|
    conf.define(:default, foreground: "#cad3f5", background: "#363a4f")
    conf.define(:enhanced, foreground: "#cad3f5", background: "#5b6078")
    conf.define(:scrollbar, foreground: "#c6a0f6", background: "#181926")
  end
else
  IRB.conf[:USE_AUTOCOMPLETE] = false
end
```

To walk through this some, I first check `defined? Reline::Face` because I sometimes work on older projects that don't have these newer features available. When I'm unable to customize through `Reline::Face`, I prefer to just disable autocomplete entirely with `IRB.conf[:USE_AUTOCOMPLETE] = false`.

If `Reline::Face` does exist, I can define the colors used by `:default`, `:enhanced`, and `:scrollbar`. The `enhanced` option defines what the currently highlighted item in the autocomplete dialog looks like. The `default` option defines what every other item in the list looks like. And the `scrollbar` of course defines what the scrollbar looks like.

If your terminal doesn't support truecolor (a common example is the default Terminal app for MacOS), then you won't be able to use full RGB for these. See [Reline's docs](https://github.com/ruby/reline/blob/master/doc/reline/face.md) for other options.

### More with IRB

There are several other cool things you can do to configure your experience with IRB. You can set aliases for commonly used methods, you can define methods for more complicated tasks that you find yourself frequently executing, you can [write a custom prompt that tells you if you're in the production or development console](https://twitter.com/_swanson/status/1346851840944730112), etc.

If you're interested in any of this, I definitely recommend reading the [IRB docs](https://docs.ruby-lang.org/en/3.2/IRB.html). I like using some of the features around `EVAL_HISTORY`, which are disabled by default. Let me know in the comments if you'd like any more IRB-centered posts :smile:
