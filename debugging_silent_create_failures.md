# Debugging Silent Create Action Failures

I often work with beginner Rails developers through [The Odin Project](https://theodinproject.com) and [The Agency of Learning](https://agencyoflearning.com/). One common pain point people may run into while learning is the dreaded "silent create action" failure. You've written your model, controller, and routes for a new resource, you've built the form view for creating this resource, but when you fill out the form and click the submit button, nothing happens. And the logs may not offer obvious clues to pinpoint the problem.

In this post, I'll discuss some techniques I find useful for getting to the bottom of this type of issue.

## Isolating Layers

So let's say you have a typical `#create` action like this:

```rb
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)

    if @post.save
      redirect_to @post, notice: 'Post successfully saved'
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

You fill in the form for a new post, click submit, and then nothing happens. Where should you start? When you're filling in a form to create a resource, you're exercising every layer of the app to make that action happen: the view, the controller, and the model. It'd be a simpler problem if we could examine just one slice of the app. Narrowing down the focus makes it easier to spot bugs.

So I begin by ensuring the model layer works in isolation. Rails provides a super convenient way to check this: the Rails console. The Rails console is a powerful tool for interacting with your application's code and data. Open up a rails console by doing:
```sh
bin/rails console
```

And then try to get a `Post` (or whatever resource you're working with) to save. Often you can spot problems with this related to how your data is set up, particularly when factoring in associations. In the above example, perhaps I've said that a `Post` `belongs_to` a `User`. But because I haven't thought to provide a user id for the new post, the database will reject it.

Most of the time, I take this step before even writing any route, controller, or view code at all. I like to make sure there are no problems with how I've set up associations and data migrations, and I like to have it fresh in my head exactly what data this model needs to save. But in the event that I didn't take this step when starting out, I make it the first thing I check when I run into trouble.

## Checking on the Parameters

A quick next step I take is to look at what the parameters are on the request to the `create` action. You can find these by looking back through the server logs. They'll appear adjacent to where the logs start for the request like this:
```
| Started POST "/posts" for 127.0.0.1 at <timestamp>
| Processing by GoatsController#create
|   Parameters: {"authenticity_token"=>"[FILTERED]", "post"=>{"title"=>"Title", "body"=>"Post's body"}, "commit"=>"Create Post"}
```
If you have trouble finding it among the other stuff happening in the server log, well, so do I! I recommend learning how to programmatically search through your terminal output. Providing a universal method for this is challenging because various tools and terminal emulators implement this functionality differently. Another option would be to use tools like `grep` or `[the_silver_searcher](https://github.com/ggreer/the_silver_searcher)` (a favorite of mine) to search the file where your dev logs are written to. This file is located at `log/development.log` in a Rails project.

After you've located the parameters, you can examine them for clues regarding the failure to create. You can see the exact data that your resource will be receiving, and you can consider if that data is what you expect. The parameters will also have a special annotation if any of them are "unpermitted". This happens when you haven't properly configured [strong parameters](https://guides.rubyonrails.org/action_controller_overview.html#strong-parameters) for your resource.

## Save with a Bang

If I still don't know what the problem is, my next step is to change how I'm saving the object. The standard construction of a `save` tends to look like:
```rb
if @post.save
  redirect_to :post, notice: 'Post successfully saved'
else
  render :new, status: :unprocessable_entity
end
```
In this code, if the model fails to persist to the database, the `save` call will return `false`. This will fire off our `else` condition. This is nice for elegantly handling issues with user inputs, but it's less nice for understanding any bug that is causing the save's failure. We need to get at what's going wrong rather than just silently rendering the `new` template for this resource.

To get more information about what exactly is causing the failure, I like to temporarily change my `#save` to a `#save!`. When you add a `!`, a failure no longer returns `false`. Instead, it will raise an error. Errors and their messages tend to provide crucial insights into the underlying cause of a problem.

## The Big Guns

For a straightforward `#create` action, I usually find the steps above are sufficient for solving the problem. Sometimes there's more going on in the action though. If I still don't have a great understanding of the problem, I'll usually break out the ultimate tool: a debugger. 

Debuggers are powerful tools that allow you to step through your code line-by-line, inspecting variables and understanding the flow of execution. Using debuggers is a whole topic unto itself, and getting into the weeds with that would balloon the scope of this post. If you want more information on using them, I recommend reading the [README for rdbg](https://github.com/ruby/debug). This is *the* debugging solution for modern Ruby/Rails development. It's in Ruby's stdlib as of v3.1, and Rails 7+ apps include it in the Gemfile by default. I also recommend [this section of the Rails guides](https://guides.rubyonrails.org/debugging_rails_applications.html#debugging-with-the-debug-gem) for exploring how to use the debug gem with Rails applications.

## Developing a Process

Hopefully the steps I've outlined here can help demystify these silent `create` failures. I've found it helpful to develop a debugging  process for the different types of common errors I run into, and I encourage other developers do the same. Remember, debugging isn't just about fixing errors; it's a skill that evolves with each challenge. Embrace the journey, uncover the intricacies of your code, build a debugging process. You'll understand your code better, and you'll uncover errors faster.

Happy debugging :bug:
