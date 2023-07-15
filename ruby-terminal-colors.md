# Ruby Terminal Colors

I'm a web development student who's been learning through [The Odin Project](https://theodinproject.com) for the past year now (wow time flies). Students who join me in the [Ruby on Rails](https://www.theodinproject.com/paths/full-stack-ruby-on-rails/courses/ruby-programming) course will encounter a number of projects where we build terminal applications using plain ol' Ruby. This is great for developing general programming skills and familiarity with the Ruby language, but since you're in the terminal, it can be a challenge to make your apps aesthetically pleasing. In this post, I'll explore some ways you can add visual flare to your ruby terminal apps through using colors.

## Off-the-Shelf Solutions

So you want to add color to your terminal app? Since you're a programmer, the first thing you do is go straight to Google. This generally leads people to two ideas. One is the popular [Colorize](https://rubygems.org/gems/colorize) gem, which is an easy and super convenient way to get color functionality into your app. You just install the gem, `require` it into whatever file(s) you wish to use it in, and then you'll have several color utility functions added onto the `String` class. Other things people often run into are Stack Overflow posts like [this one](https://stackoverflow.com/questions/1489183/how-can-i-use-ruby-to-colorize-the-text-output-to-a-terminal), which essentially peel back the curtain on how the Colorize gem works. When we want to add colors to a string, we wrap the string in an [ANSI escape code](https://en.wikipedia.org/wiki/ANSI_escape_code) that signals how your terminal emulator should style the text.

Both of these methods of injecting color into your app are great and easy to use. I would say they're ideal for if you're just wanting to color the log output for some server or process you're running. But if you're using them to add color to a terminal app or game, I find these and most other solutions out there underwhelming for one big reason: they offer an extremely limited color palette. The Colorize gem defines methods for 16 colors, and most other sources only show how to work with 8 or 16 colors. But most modern terminal emulators supports 24-bit "true color", meaning we actually have access to the RGB spectrum of 16,777,216 colors. Using this full spectrum can lend a lot more flavor and personality to a terminal app. So let's explore how to do it!

## A Note about MacOS Terminal

The default MacOS Terminal app would be one of the modern terminals that does not support true color. If you're a Mac user and want to use the exact technique I show in this post, you'll have to use a different terminal emulator. iTerm2 is a popular choice on MacOS. It provides support for true color and a few other cool features not present in the default Terminal app.

Another option would be to use 256 color mode, which will work on the default Terminal app. For this mode, you can still reference the rest of the article except your ANSI color string will look like this:
```rb
# the 5 at the beginning puts this in 256 color mode
"\e[38;5;#{color_code}m#{string}\e[0m"
```
And you can find what colors map to what color code through this [table on wikipedia](https://en.wikipedia.org/wiki/ANSI_escape_code#8-bit)

## 24-bit ANSI color code

So above I mentioned that ANSI codes are used to style the text in a terminal. They can also be used for many other things from moving the cursor around to deleting lines of text, but right now I'll just be focusing on the colors. We can see the form of the 24-bit ANSI code [here](https://en.wikipedia.org/wiki/ANSI_escape_code#24-bit). Translated into a Ruby string, it would look this:

```rb
"\e[38;2;#{r};#{g};#{b}m#{string}\e[0m"
```

Let's break down what's happening here. `"\e["` is the "control sequence introducer" and is how we begin using one of these codes. This is followed by a list of semicolon-separated parameter values to communicate what exactly should happen. `38` signals that we're setting the foreground color (we'd use `48` for the background). The `2` signals that we're using the RGB format for the color. Then we list each RGB value for the color we want to use. The `'m'` is a "final byte" to signal the end of the parameter list. This is followed by the string we want to apply the color to. The final piece is the `"\e[0m"` reset code. This returns the colors to their default state.

So let's quickly test this is working using the color green (r: 0, g: 128, b: 0) -- ignore the weird syntax highlighting.
```rb
green_rgb = "0;128;0

puts "\e[38;2;#{green_rgb}mhello\e[0m world"
```

And you should see this text when you run the above code: 
![Demonstration of the above ANSI code terminal output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k9lp86gjaw7u2ydzo3rb.png)

This is good. We have our green "hello", and "world", which was placed after the reset code, is back to normal.

So what's a good way to go about building this into our app? You can probably already see a method that can arise from the above code. Perhaps something like:

```rb
def fg_color(string, rgb_values)
  "\e[38;2;#{rgb_values}m#{string}\e[0m"
end

puts fg_color("hello world", "0;128;0")
```

But I think it'd be nice for the caller to not have to know or remember these RGB values, and if we extend the `String` class like the other solutions above do, we can get really clean, chainable methods. So let's explore that.

## Building a Color Module

So I'm going to build a module that gives the `String` class some color utilities, but first I want to mention the pitfalls of "monkey patching" (or extending the base classes of the language). Ruby makes this very simple to do. We can just reopen any of the base classes anytime we like.

```rb
class String
  def shout
    "#{self}!!!"
  end
end

puts "hello world".shout
#-> "hello world!!!"
```

But despite the ease of doing this, it's often considered a code smell. One of the reasons is that this kind of code will be globally available -- because `String` itself is of course globally available. You may think that this kind of thing could work

```rb
module StringExtensions
  def shout
    "#{self}!!!"
  end
end

class Display
  String.include StringExtensions
  # these methods are only available in this class, right?

  def yell_at_user
    puts "Don't do that".shout
  end
end

"hello world".shout
#-> "hello world!!!"
# the #shout method is available anywhere
```

We'd rather these methods only be available where they can/should be used. Enter the `refine` and `using` methods, which are builtin methods on the `Module` class. With `refine`, we can define some extensions on a base class, and then with `using`, we can be selective with where these refinements are available in our code. Check it out:
```rb
module StringExtensions
  refine String do
    def shout
      "#{self}!!!"
    end
  end
end

class Display
  using StringExtensions

  def yell_at_user
    puts "Don't do that".shout
  end
end

Display.new.yell_at_user
#-> "Don't do that!!!"

puts "hello world".shout
#-> NoMethodError
```
Using these methods, we can better control where the extensions to a base class are available. With this in mind, let's build a small color module. I'll use a few colors from the scheme of my favorite editor theme: [dracula](https://draculatheme.com/contribute)

```rb
module ColorableString
  RGB_COLOR_MAP = {
    cyan: "139;233;253",
    green: "80;250;123",
    red: "255;85;85"
  }.freeze

  refine String do
    def fg_color(color_name)
      rgb_val = RGB_COLOR_MAP[color_name]
      "\e[38;2;#{rgb_val}m#{self}\e[0m"
    end
  end
end

class Display
  using ColorableString

  def welcome_user
    puts "Welcome to my App".fg_color(:green)
  end

  def user_choice
    puts "Please choose an option".fg_color(:cyan)
  end

  def yell_at_user
    puts "Don't do that".fg_color(:red)
  end
end

display = Display.new

display.welcome_user
display.user_choice
display.yell_at_user
```

![Demonstration of above terminal output from above code](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o4jt0g3h44ysqngr2i6c.png)

And now we have color! And it's in a format that's easy to reuse and extend. If you need background colors, just create a `bg_color` method with the `38` replaced by a `48`. You can also create the color methods through metaprogramming if you feel so inclined, but I felt it best to keep it simple. Happy colorful coding!
