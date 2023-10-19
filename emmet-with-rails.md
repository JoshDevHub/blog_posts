# Emmet and Rails Views

Emmet is a tool that helps you rapidly build sections of HTML by typing abbreviated snippets of what you want. Here's a brief example:
```html
<!-- type this and press `tab` in a `.html` file -->
div>ul>li.list-item*3

<!-- and you'll see it expand to: -->
<div>
  <ul>
    <li class="list-item"></li>
    <li class="list-item"></li>
    <li class="list-item"></li>
  </ul>
</div>
```
Support for this is built in for the current most popular editor: [VSCode](https://code.visualstudio.com/docs/editor/emmet). But if you work with Rails a lot, you'll probably notice that it does not work in `erb` template files out of the box. Let's enable it!

To start, open your `settings.json` file in VSCode. You can take a shortcut there by opening the command palette (`ctrl/cmd + shift + p`) and typing "settings.json". Look for the item `Open User Settings (JSON)` and click it to open the file. Inside the curly braces of the JSON, you'll want to add this:
```json
"emmet.includeLanguages": {
    "html.erb": "html"
},
```
This will tell Emmet to include its HTML snippets for files with the `.html.erb` extension. Save and close your `settings.json`, open up a view template in a Rails project, and you should now be able to use these snippets. [Here](https://docs.emmet.io/cheat-sheet/) is an awesome cheatsheet that shows some of the cool stuff you can do to generate HTML quickly.

## Wrap with Abbreviation

The snippet stuff can be super useful, but I think one of the best Emmet tricks is something that isn't enabled by default in VSCode. Have you ever needed to wrap a section of HTML in a `div` for styling purposes? I have to do this often, and I find doing it manually is cumbersome. You have to type the opening tag, the closing tag, and then you have to indent the section that you're wrapping. Enter `wrap with abbreviation`:

<gif>

So you can highlight the section you want to wrap, hit a keybind, and then type the emmet snippet you want to wrap that section in. Emmet will handle inserting the tags and indenting the content in between automatically.

VSCode doesn't have this keybind set by default. To set it, you'll need to navigate to your keyboard shortcuts. Click the gear icon in the bottom left and you should see `Keyboard Shortcuts` in the submenu. Click on it and then search 'wrap with abbreviation'. You should see the Emmet command pop up, and you'll be able to set the keybind to something that's comfortable to use for you.

## Other Considerations

What if you don't use VSCode? That's understandable, my preferred editor is Neovim. Emmet is still probably supported in some fashion by the editor you use. You'll just have to google to fill in some gaps. I wrote this for VSCode because it's what 95% of developers use these days.

You also might sometimes want these snippets in other places, like in `.jsx` files if you work with React. In that case, you can add `.jsx` to `emmet.includeLanguages` in your `settings.json` file like I showed with `html.erb`.
