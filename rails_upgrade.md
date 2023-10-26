# Legacy Rails Upgrade & Adventures in Middleware

At work we've been tasked with upgrading an old Rails project (version 4.2) up to Rails 6 or possibly Rails 7. I've never handled an upgrade of this magnitude, and it's been a big learning experience for me. It's fascinating to learn more about the history of Ruby, Rails, and the more popular gems in the ecosystem. It's also involved learning more about the parts of Rails that are typically out of view in day-to-day application development. In this post, I'll focus in on one particular problem that was quite the adventure to solve.

## The Problem

Because 4.2 is the latest major patch version of Rails 4, we started out by bumping up to Rails 5.0. After we got the gems bundled with Rails 5 and the server running, we noticed an unusual error when trying to visit any page of the application:
```
Started GET "/" for ::1 at 2023-09-29 10:01:37 -0500
Processing by ErrorsController#server_error as HTML
Completed 500 Internal Server Error in 0ms (ActiveRecord: 0.0ms)


Error during failsafe response: Devise could not find the `Warden::Proxy` instance on your request environment.
Make sure that your application is loading Devise and Warden as expected and that the `Warden::Manager` middleware is present in your middleware stack.
If you are seeing this on one of your tests, ensure that your tests are either executing the Rails middleware stack or that your tests are using the `Devise::Test::ControllerHelpers` module to inject the `request.env['warden']` object for you.
```
The first natural step was to google that error, but all of the results from googling turned up issues related to testing, websockets, turbo, or jobs. Here we were just trying to load the root page for the app, and from looking through the code, we knew none of those things were relevant to this failure.

Because the upgrade to Rails 5.0 involved bumping the devise gem from version 3 to version 4, I thought a good next step would be to look through the changelogs for older versions of devise. I thought they may point out some breaking changes that occurred in the jump from 3 to 4, and these changes to the gem's code may provide an answer to the issue we were encountering here. But after browsing their documentation, I couldn't find anything that hinted at the above error being introduced by changes to the gem, so I knew this was an unlikely culprit.

## Rails Middleware

At this point my coworker and I were pretty stumped, so we promptly reached out for help (an essential developer skill by the way!). One person suggested we check to see if the `Warden::Manager` middleware is present in the middleware stack. Now if you're newer to Rails, you may think "What the heck is middleware?" We can think of middleware like a stack of small components that sits between the web server and our Rails application. When a request comes in, it passes through the middleware stack before reaching the application. The response also passes through the middleware stack when leaving our application to the client. Each of these small components in the middleware stack can examine the request/response as it's passing through, and depending on what it finds, it may decorate the request or perform some side-effect. You can see a list of the default Rails middlewares [here](https://guides.rubyonrails.org/rails_on_rack.html?ref=akshaykhot.com#internal-middleware-stack). If you follow that link, you'll see things related to logging, exception handling, security, etc. -- things that aren't specific to your Rails application logic. And this is why Rails implements them as middleware.

Gems can also add onto the middleware stack. Devise does so using [warden](https://github.com/wardencommunity/warden). The warden middleware helps with session management and examining requests for authentication information. If it wasn't present in the middleware stack, it would probably be the cause our error. Rails offers a task to check to check the middleware stack: `rake middleware`. We ran this and ... `use Warden::Manager` was unfortunately present. Time to dig deeper!

## Diving into Gem Code

Other advice we received was to think about where in the request/response cycle the error is occurring. From throwing a quick `raise 'Problem'` in the controller action that serves the root page of the app, we could see that the error raises before reaching our app's controller code. This prompted me to look more deeply through the stacktrace for the error to see what's causing it. From the stacktrace, we could see that the error was originating in the middleware stack of the app. Another helper was able to track down the exact method where the problem originated: [`ActionDispatch::ShowExceptions#render_exception`](https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/show_exceptions.rb#L44-L62).

Now I can start to form a clear theory for what's happening. The request develops an error on its way through the middleware stack, the `ShowExceptions` middleware tries to render the error using `@exceptions_app`, but then `current_user` is attempted in this process. Because some part of the middleware stack isn't functioning correctly, `Warden::Manager` never does its thing, so `current_user` can't be used without raising an error.

But now we've got an interesting problem. This is library code that our application doesn't control. How should I go about debugging this code? Well, fortunately gem code does actually exist somewhere on your machine. It can be a bit annoying to manually find because of different ruby versions and gem versions. To help with this, bundler offers a tool to quickly access gem code: `bundle open`. So I ran `bundle open actionpack` (`actionpack` holds the `ActionDispatch` module) to open the relevant gem code in my editor. From there I could navigate to the above method and place breakpoints for my debugger. Here, I was able to see the error that triggers the `#render_exception` call in `ShowExceptions`:
```
Caused by NoMethodError: undefined method `remote_addr' for nil:NilClass
Did you mean?  remote_byebug
from /Users/josh/.rbenv/versions/2.7.6/lib/ruby/gems/2.7.0/gems/actionpack-5.0.0/lib/action_dispatch/middleware/remote_ip.rb:112:in `calculate_ip'
```
So there was another issue in the `ActionDispatch` middleware that's ultimately causing the problem. I used `bundle open` again and examine the code for [`ActionDispatch::RemoteIp#caluclate_ip`](https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/remote_ip.rb#L123-L163). I could see that `@req` is `nil` for some reason. Using the `backtrace` command, I could see what led to initializing `RemoteIp::GetIp` with a `nil` request. I was able to see that it was being initialized from [`WebConsole::GetSecureIp`](https://github.com/rails/web-console/blob/main/lib/web_console/request.rb#L19-L36), a class that inherits from the aforementioned `ActionDispatch` class.

From here I was able to google to fill in the rest of the gaps. I was able to track down [this issue](https://github.com/rails/web-console/issues/178), where they were discussing pretty much the same problem I was having. They mentioned in the discussion that this problem "is fixed in master". So I looked for the version of web-console that has this fixed and bumped to that version in our Gemfile. And what do ya know it worked :party:

## Special Thanks

The people who helped me throughout this process are all fellow members of [Agency of Learning](https://agencyoflearning.com/), a community of early career developers and mentors. I've been able to get a lot of support from the community on different issues I've faced at work and don't have to bother my (very busy) boss as much.

Applications are currently open for a new cohort of members. If you're reading this and you're an early career Rails developer looking to land your first role, I recommend applying. NOTE: look at mission information for Agency
