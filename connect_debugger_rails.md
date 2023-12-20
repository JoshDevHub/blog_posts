# Connecting Debugger to Rails Applications

In a previous post about [debugging silent create action failures](), I touched upon the practice of connecting the Rails web process to a debugger when facing challenging issues. A debugger allows you to methodically step through your code and examine the runtime values for every stage of a process.

Setting up the debugger can be challenging though, especially if your app is being driven by a `Procfile`. I wanted to make a dedicated post about the different ways you can get the debugger up and running in a Rails app. So let's get to it!

**Note**
This post assumes that you're on a Rails 7 app, which comes out of the box with the ruby debug gem. With earlier versions of Rails, the ruby debug gem wasn't included by default, and it was common to use the `pry-byebug` gem for debugging. So if that's the situation you're in, not everything I say in this post will work.

## Without a Procfile

In the root of your Rails application, run `ls Procfile*` in your terminal. If nothing shows up, your application's processes aren't managed by a `Procfile`, and you likely just run `rails server` to start your app.

If this is the case, then connecting a debugger to your server is fairly straightforward. Let's take a `#create` action on a `Post` resource as an example for illustration:
```rb
class PostsController < ApplicationController
  def create
    # assuming `current_user` is an instance of the User model
    @post = current_user.posts.build(post_params)

    if @post.save
      redirect_to root_path
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```
Let's say this post won't save and you want step through this code to analyze why. You can add a breakpoint to the beginning of the `#create` action:
```rb
def create
  # aliased as `binding.b` and `debugger`, you can use those too
  binding.break

  # the actual code ...
end
```
Now in your browser with `localhost` open, when you try to submit a form to create a new post, you'll see that the browser just hangs as if caught in an infinite loop. This is good thing! Your application is effectively paused at the breakpoint you set.

If you look in the terminal window that's running the server process, you'll see the prompt for using the debugger. You can visit the [documentation](https://github.com/ruby/debug) to learn more about the different commands that control the debugger.

## With a Procfile

If you ran the step above and you have one or more `Procfile`s in the root of your Rails app, then your application is probably run through one. To start your server process, you likely use a tool like `foreman`, `heroku local`, or some `bin/dev` script.

A `Procfile` tends to be used when there are multiple processes that need to be running for your app to function. This can include things like building CSS/JS, background worker, search engine, etc. A `Procfile` will specify the commands for each process that your app requires. Instead of opening new terminal windows for each process, tools like `foreman` and `heroku local` can streamline running your app by using the `Procfile` to run these processes from a single terminal instance. This is great for keeping your keeping your workspace clean and ensuring you have all the necessary processes running.

But this introduces some problems if you try to connect a debugger using the flow above. Aforementioned tools like `foreman` and `heroku local` won't know that they should be listening to and forwarding stdin (your inputs on the terminal) to your web process. So although you can set breakpoints and pause execution in your app, you'll be unable to actually control the debugger.

### Solutions

There are a few solutions to this:

#### Split out the server process

You can isolate the Rails server process from the rest of the commands in the `Procfile`. Navigate to your `Procfile` (or potentially `Procfile.dev` if you have a different one set to run locally), and remove the line that runs the web process. Now when you run the `Procfile`, all your processes will start up except the server. Then you can open a new terminal and run the server in isolation with `bin/rails server`. With the web process running separately, you'll be able to properly operate the debugger in the terminal that's running the server.

This is probably the most straightforward method, but it can be a bit inconvenient if you regularly have to do this. If it's your own project or one you control,you can just remove the server process from your local `Procfile.dev` and always run the server separately. If you're working with others, the team may not be okay with committing this kind of change to the `Procfile`, so you'd have to manually set this up each time you want to debug and then remember to revert it back before committing any changes.

#### Use other tools

Another solution is to use a different tool to drive the `Procfile`. The one I'm most familiar with is a tool called [overmind](https://github.com/DarthSim/overmind). If you run your `Procfile` with `overmind`, you'll be able to open up a new terminal window and individually connect to any of the processes that are running. So if you want to connect to the web process to debug, you can open up a new window and run `overmind connect web`, and you'll have a window where you can properly work with the debugger's prompt.

The downside of `overmind` is that it requires [tmux](https://github.com/tmux/tmux), which is a terminal multiplexer tool. If you don't already use tmux, I'd say it's probably not worth learning it just for the purposes of using `overmind`. But if you're like me and already know/use tmux, this can be a great solution to pursue.

#### Connect to debugger remotely

In the [`Procfile.dev`](https://github.com/rails/cssbundling-rails/blob/main/lib/install/Procfile_for_node.dev) that's currently generated by the cssbundling-rails and jsbundling-rails gems, we can see that they write the web process like:
```sh
web: env RUBY_DEBUG_OPEN=true bin/rails server
```
This `RUBY_DEBUG_OPEN=true` bit can allow us to remotely connect a debugger to the server instance the `Procfile` is running. When you run the `Procfile`, you'll see something like this in the server logs:
```text
web.1  | DEBUGGER: Debugger can attach via UNIX domain socket (/run/user/1000/ruby-debug-josh-1396004)
```
Except probably with a different username than "josh" :laugh:

But like the message said, a debugger can attach remotely via UDS. Don't worry, it sounds more intimidating than it is. You just need to open a new terminal instance and run `rdbg -An`. The `-A` option means "attach", and it automatically connects the debugger to the open UNIX domain socket that was setup. If you have multiple sockets open for a debugger to connect to (ie. because you're running multiple Rails projects), you may be forced to specify which one it should use. The `-n` option means "nonstop." This prevents the program from stopping when you connect the debugger, which is usually what you're going to want when debugging a Rails app.

Now when you hit a breakpoint, you'll be able to use the debugger in the terminal window where you remotely connected with `rdbg -An`. This is straightforward to use and the method I recommend for most people. If it's a project where you can't change the `Procfile.dev`, you can prefix your command to run the `Procfile` with `RUBY_DEBUG_OPEN=true` to still enable remote debugging.

### Conclusion

Using the debugger to figure out tricky problems has saved me a ton of time when working on Rails apps. It's a great skill to learn and practice, but I also know that a lot of beginners avoid it because of how troublesome the setup can be. Hopefully this post has given you some ideas about the workflow you want to use for debugging. Have fun out there :bug:
