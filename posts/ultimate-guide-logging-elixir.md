# The Ultimate Guide to Logging in Elixir

Last Updated: _August 2, 2018_

## Preface: How to get started with this guide

### Versions used in this guide

- Elixir: v1.6.5
- Hex: v0.17.7

This tutorial was built on a Mac OS X 10.12.6 system.

### Assumptions

1.  You understand a minimum of Elixir syntax and concepts
2.  You have all of the pre-requisite software installed. Pre-reqs are:
    - Elixir ([Installation Guide](https://elixir-lang.org/install.html))
    - A code editor or IDE (I'll be using [Visual Studio Code](https://code.visualstudio.com/Download))

### About this guide

Throughout this guide, you'll see a lot of useful stuff, whether it's descriptions, code, or shell commands! Any time you see some code like this:

    $ do the thing

These blocks represents shell commands being run (specifically, anything with a `$` in front of it). Any additional lines will be output from the command run. Sometimes `...` will be used to indicate long blocks of text cut out in interest of brevity.

IEx (Elixir's Interactive REPL) has commands will be represented like this:

    iex(1)> "do the thing"

And any additional lines will be output from that operation. Don't worry about the number inside of the parantheses. These just indicate what command number your IEx shell is currently on and these may not match up.

Code will be represented in code blocks, like this:

```
name = "brandon"
name
|> String.upcase()
|> String.reverse()
```

## Introduction

The best way to get the most out of this tutorial is to follow along as you build the same project as us (or something equivalent). The code will be available on Github with checkpoint tags at each step allowing you to double-check your code against the finished product per each checkpoint!

Ultimately, logging is a critical part of the long term maintenance and observability of your application in any language, tool, or suite you're going to work with. In some languages, dealing with logging is really more of an afterthought, or lacking any sort of cohesive tooling and structure to make handling logs and events in different ways a real chore.

Logging in Elixir, however, is **fantastic**. It's built to accomodate the million use cases for logs and events, and is built on top of the OTP event structure that not just allows you handle logging in a sane way, but also allows you to have multiple simultaneous ways to handling logging so that nothing gets lost in the shuffle! Want to have a log file, a stdout log, and send your logs along to a service like Timber.io? Great, you can do that, and even better, you can do that **without affecting your service's response times**!

Elixir provides not just a standard logging interface that is part of the core language itself (so no scenario where logging in Phoenix is great but logging elsewhere is tricky), but a really simple way to modify and add new logging handlers into your codebase no matter what type of Elixir project you've written!

### Starting out with a template project

You can clone [my starter project from Github](https://github.com/richeyb/runnable-elixir) to start out your project. We'll be using that as the base to mess around with our logging code as we go along. To run our examples, we're going to use `mix run`, and we'll be making modifications to `lib/runnable.ex`. Specifically, we'll be modifying the `log/0` function.

### Example using IO.puts/IO.inspect

#### Using IO.puts

Logging is one of your most powerful debugging tools we can work with, so we're going to start off using the most simple implementation of logging: using `IO.puts` and `IO.inspect`.

`IO.puts` takes either one or two arguments. If you're using the two-argument version, then the first argument is an atom specifying the destination of the message (for example, `:stdout` or `:stderr`). If you're using the one-argument version, then the destination is assumed to be `:stdout` and the message will just be output directly. For example, the line of code that we already have in our `log/0` function is:

    defp log do
      IO.puts "Starting Application..."
    end
    
Since we're not specifying the destination for the message, it is assumed by default to be `:stdio`. We can verify this by changing that line to:

    IO.puts :stdio, "Starting Application..."
    
And we'll run our application again:

    $ mix run
    Compiling 1 file (.ex)
    Starting Application...
    
We could then remove the first `:stdio` argument to our IO.puts function and get the same result! If we wanted to target a different device, such as `stderr`, we could do that with the `:stderr` atom used as the first argument to `IO.puts`.

#### Using IO.inspect

`IO.inspect` is a different beast. While `IO.puts` is expecting anything you pass to it to be a string or string-compatible, `IO.inspect` can handle much more complex forms. You can actually see this really easily by adding the following line to your code: `IO.puts self()`. Attempting to run this will give you an error message similar to the following:

    ** (Protocol.UndefinedError) protocol String.Chars not implemented for #PID<0.130.0>.
    
Running `IO.inspect self()`, however, will work (although it's not the most useful output in the world).

    $ mix run
    Compiling 1 file (.ex)
    I'm a log
    #PID<0.130.0>

The good news: this will be enough to get you some good enough logging during the development phases of building your application. The bad news, however, is that this is not a great approach to a Production system and shouldn't be relied on to monitor your application.

## Doing it the robust way

There's more good news, though: Elixir can, out of the box, take advantage of Elixir's built-in Logger module to handle logging. Even better, the default behavior is very easily extended beyond just a simple console logger! We'll start off with a base implementation of Elixir's logger and then we'll dive into extending it to be able to handle more complicated cases and functionality using Elixir's Logger Backend system!

### Using Logger

By default, `Logger` is included with your application! Remember back to our `mix.exs` file where we saw this bit of configuration:

    def application do
      [
        extra_applications: [:logger]
      ]
    end
    
Logger is there and usable, so let's start working with it! We'll head back to our code in `lib/runnable.ex` and delete our old logging code and use the more idiomatic Elixir way to log instead. Let's change our code to instead use a call to Logger's debug function to still say "Starting Application...". Before we can do that, we'll need to require Logger into our application, so up at the top, below the `use Application` line, add the following line:

    require Logger
    
Next, we'll modify the log function to call out via Logger.debug:

    defp log do
      Logger.debug "Starting Application..."
    end
    
When you run this in your console, you should see (possibly in cyan, if your terminal supports colors):

    $ mix run
    Compiling 1 file (.ex)

    14:44:32.694 [debug] Starting Application...
    
Great! We're using the debug level here for our output, but we could include other levels of output in our code as well. Logger supports the following log levels for output: `debug`, `info`, `warn`, and `error` (which are in order of how critical they are). One of the most helpful things about using `Logger` is that it will respect whatever configured log level you specify when logging out, so if you set the application's log level to be `:info`, it will only include `info` level logs and above (so `warn` and `error` would also be included but not `:debug`). Let's test this out by modifying our config file to set the log level to `:info` and see if we still see our debug message when we run `mix run`. Open up `config/config.exs` and add the following line:

    config :logger, level: :info
    
And then we'll run `mix run` again and see what happens:

    $ mix run
    Compiling 3 files (.ex)
    Generated runnable app
    
But no output! Great!

#### Understanding Logger Configuration

Logger can be configured either via a general `config/config.exs` file, which will affect all environments, or by specifically using a configuration file for each environment (`dev.exs`, `test.exs`, `prod.exs`, etc). We'll stick with just using the general config right now as environment configuration is a bit outside of the scope of this article.

##### Application Configuration

There are two different levels to logging and they all have to be set up slightly differently. The first is **Application Configuration**, which allows you to set a few different configuration options that affect ALL of your logging. This distinction is important, since it affects which options can be changed during runtime or not! So, if you want to use the Application configuration settings, you have a few options that you can set: `:backends`, `:compile_time_purge_level`, and `:compile_time_application`. These are set by creating a config entry that looks like this:

    config :logger,
        backends: [:console],
        compile_time_purge_level: :debug
        
`:backends` specifies which logging backends should handle your log messages. By default, Logger comes with a console logger backend, so the default is just `[:console]`, but you can install hex packages with other logging backends or even write your own (more on this later).

`:compile_time_purge_level` allows you to set what level of log statements should be completely ignored and removed at compile time. In Elixir v1.6, you can't specify anything outside of the level of logs you want removed. This can be helpful when you're concerned about any possible extra overhead existing in your application when it's actually running.

`:compile_time_application` sets what the value of the `:application` metadata would be at compile-time. This could be helpful especially in the context of environment-specific configurations, where maybe you want to purge any log statements that match (or do not match) a specific application!

These cannot be changed at any point by the Application and in fact must be set in your config BEFORE the logger application is even started!

##### Runtime Configuration

Next, you have your runtime options, which are set via the same config statements. The difference is that these can actually be overridden at any point during runtime. There are a lot more configuration options here, so I won't go into listing them all, but the three most common configuration options you would use will likely be `:level`, `:utc_log`, and `:truncate`.

`:level` specifies which log levels should be used or ignored. Anything below the level you specify will be ignored, and everything else will be used. So, bearing in mind the order of `:debug`, `:info`, `:warn`, and `:error`, if you specify the log level to `:info`, then `:debug` will be ignored and the rest will be used! This is different from `:compile_time_purge_level`, however, in that the statements are not stripped from your application's runtime, so evaluations may still be used if not wrapped appropriately.

`:utc_log` tells Elixir to use UTC times when outputting your logs instead of local times. By default, this is set to `false`.

`:truncate` is the maximum message sizes to log out in bytes, which defaults to 8192 bytes. Anything affected by this will have `(truncated)` appended to the end of the message. If you just want everything without truncation, you can use `:infinity`.

You can see more configuration options at [the Logger docs](https://hexdocs.pm/logger/1.6.6/Logger.html#module-runtime-configuration).

Finally, there is the Error logger configuration, which tells Elixir how to handle error logs. By default, one of the options (`:handle_otp_reports`) is set to `true`, so all OTP reports will instead be redirected to Logger and formatted in Elixir terms. There is also `:handle_sasl_reports`, which also redirects any supervisor, crash, or progress reports to Logger as well. This relies on the `:handle_otp_reports` option to be set to true for this to work, but is set to `false` by default.

#### Configuring the Console backend for Logger

You can also specifically configure any backends used by Logger. The default backend, `:console`, has a number of different configuration options, but the three most frequent ones you'll likely use will be `:level`, `:format`, and `:metadata`. The syntax for these configurations are slightly different than just configuring Logger as a whole in your `config.exs` file.

You configure these calls to the console backend by adding the `:console` atom to your config statement directly after your `config :logger` statement, like so:

    config :logger, :console,
        level: :info
        
Note that an atom directly after the `config :logger` statement tells Elixir that we are specifically configuring the backend specified by that atom. In the case above, we're specifically trying to configure the `:console` backend!

`:level` you already know how to configure; the only important thing to note is that this is specifying the level for the console logger backend only!

Next, we have `:format`. This tells the backend console logger how to output messages to the console. By default, the format is `"\n$time $metadata[$level] $levelpad$message\n"`. You can instead modify this to be a different format, using the substitution variables for time, metadata, level (or padded with levelpad), and message.

Finally, the other common configuration you'll use is `:metadata`. This allows you to whitelist any metadata that you want output as part of your console log messages. This can be really handy when setting up default formats or values to include in all of your logs to make parsing them or understanding them easier in the future. By default all metadata is disabled. If you pass in `:all` as the configuration value for metadata, ALL metadata will be output. For example, let's set that now in our config file:

    # Our Console Backend-specific configuration
    config :logger, :console,
      format: "\n##### $time $metadata[$level] $levelpad$message\n",
      metadata: :all
      
Our entire `config/config.exs` file should look like this when we're done:

```
use Mix.Config

# Our Logger general configuration
config :logger,
  backends: [:console],
  compile_time_purge_level: :debug

# Our Console Backend-specific configuration
config :logger, :console,
  format: "\n##### $time $metadata[$level] $levelpad$message\n",
  metadata: :all
  
# Uncomment this if you want to set up environment-level configuration
# import_config "#{Mix.env}.exs"
```
      
Now when we run our application we should see something like the following:

    $ mix run
    Compiling 3 files (.ex)
    Generated runnable app
    
    ##### 14:51:12.821 pid=<0.158.0> application=runnable module=Runnable function=log/0 file=/Users/brandon.richey/Development/elixir/runnable/lib/runnable.ex line=27 [debug] Starting Application...
    
What if we want to use some metadata in our log statements? Well, we can head back to our `lib/runnable.ex` file, and change our private `log/0` function:

    defp log do
      Logger.metadata(request_id: "ABCDEF")
      Logger.debug "Starting Application..."
    end
    
Now when we run our application we should instead see:

    $ mix run
    Compiling 1 file (.ex)
    
    ##### 14:52:32.810 pid=<0.130.0> request_id=ABCDEF application=runnable module=Runnable function=log/0 file=/Users/brandon.richey/Development/elixir/runnable/lib/runnable.ex line=28 [debug] Starting Application...
    
Note the "request_id=ABCDEF" at the start of our line! We can also restrict the list of metadata down to a whitelist by specifying the metadata keys we want to allow. Return to our configuration, and set the `metadata` configuration to be `metadata: [:request_id]`, and then re-run our application:

    $ mix run
    Compiling 3 files (.ex)
    Generated runnable app
    
    ##### 14:53:00.022 request_id=ABCDEF [debug] Starting Application...
    
Elixir Logging metadata is INCREDIBLY powerful! To learn more about it, please also read the article [Elixir Logger and the Power of Metadata](https://timber.io/blog/elixir-logger-and-the-power-of-metadata/)!

## Advanced Logging: Writing our own formatter

### Writing a Log Formatter

Let's say that we were working with a more complicated system that maybe stores sensitive information, or we have a very particular way we'd like our data to be formatted to make our lives easier. We can handle this by writing our own formatter instead of using the default one provided by Logger!

You can specify a module and a 4-arity function that will take in the level, message, timestamp, and metadata to turn into a log message. We'll create a new file under `lib/runnable` called `log_formatter.ex`. For example, we may want to specifically output timestamps with the ISO8601Z format instead of the default timestamp format provided to us. First, we'll change our configuration to make sure the log formatter is pointed at the right module and function. In `config/config.exs`:

    config :logger, :console,
      format: {Runnable.LogFormatter, :format},
      metadata: [:request_id]
      
Next, we'll begin implementing our format function in `lib/runnable/log_formatter.ex`:

    def format(level, message, timestamp, metadata) do
      "##### #{fmt_timestamp(timestamp)} #{inspect(metadata)} [#{level}] #{message}\n"
    rescue
      _ -> "could not format message: #{inspect({level, message, timestamp, metadata})}\n"
    end
    
Since we need to format our timestamp separately from the rest of Logger's default implementation, we'll also need to define the `fmt_timestamp/1` function:

    defp fmt_timestamp({date, {hh, mm, ss, ms}}) do
      with {:ok, timestamp} <- NaiveDateTime.from_erl({date, {hh, mm, ss}}, {ms * 1000, 3}),
        result <- NaiveDateTime.to_iso8601(timestamp)
      do
        "#{result}Z"
      end
    end
    
Save that file and rerun `mix run` and we should now see a different formatted log message:

    $ mix run
    Compiling 1 file (.ex)
    ##### 2018-08-01T16:14:49.030Z [request_id: "ABCDEF"] [debug] Starting Application...

And there you are! You could even take this further by changing what values you spit out as part of the logging process, maybe doing something like scrubbing/redacting protected information before it gets output via Logger! Here's an example `Runnable.LogFormatter` that implements a sensitive data scrubber:

    defmodule Runnable.LogFormatter do
      @protected [:request_id]
    
      def format(level, message, timestamp, metadata) do
        "##### #{fmt_timestamp(timestamp)} #{fmt_metadata(metadata)} [#{level}] #{message}\n"
      rescue
        _ -> "could not format message: #{inspect({level, message, timestamp, metadata})}\n"
      end
    
      defp fmt_metadata(md) do
        md
        |> Keyword.keys()
        |> Enum.map(&(output_metadata(md, &1)))
        |> Enum.join(" ")
      end
    
      def output_metadata(metadata, key) do
        if Enum.member?(@protected, key) do
          "#{key}=(REDACTED)"
        else
          "#{key}=#{metadata[key]}"
        end
      end
    
      defp fmt_timestamp({date, {hh, mm, ss, ms}}) do
        with {:ok, timestamp} <- NaiveDateTime.from_erl({date, {hh, mm, ss}}, {ms * 1000, 3}),
          result <- NaiveDateTime.to_iso8601(timestamp)
        do
          "#{result}Z"
        end
      end
    end

### Writing a Log Backend

Writing a Logger Backend is a bit more complicated, as there is a lot of setup and boilerplate code you need to include to make sure your code behaves appropriately. Each Logger backend should have, at a minimum, an init function, an event handler for `:flush`, an event handler for the actual log event, and a call handler for `configure`. They're complicated to write at first, but incredibly easy to build on top of (just like most things in Elixir; although there may be some initial complication, everything beyond that is generally smooth sailing and allows you to develop with confidence)! We'll start off by creating a new file, `lib/runnable/log_backend.ex` and giving it our standard module definition. This will be the base to build out the rest of our code.

    defmodule Runnable.LogBackend do
    
    end
    
We'll need an `init` handler for when the module is first starting up. It will take a tuple as the first argument that will be the name of the module (the same value that `__MODULE__` would return) and the second will be the atom-ized backend name (in our case, it will end up being `:log_backend`). You'll also want to do your initial setup stuff here, so we'll write a base-level `configure` function to work with this:

    # Initialize the configuration
    def init({__MODULE__, name}) do
      {:ok, configure(name, [])}
    end
    
    defp configure(name, []) do
      base_level = Application.get_env(:logger, :level, :debug)
      Application.get_env(:logger, name, []) |> Enum.into(%{name: name, level: base_level})
    end
    
So, notice that in our configure function, we specifically need to take in the level defined in the base-level Logger config first. We'll allow people to override our logging backend by specifying their own level in `config/config.exs` through a call that might look something like this:

    config :logger, :log_backend,
      level: :error
      
Next, we'll need to handle the `:flush` event, which will just be a no-op that returns out the current state:

    # Handle the flush event
    def handle_event(:flush, state) do
      {:ok, state}
    end
    
We'll also need to handle any `call` statements made to configure the backend:

    def handle_call({:configure, opts}, %{name: name}=state) do
      {:ok, :ok, configure(name, opts, state)}
    end
    
The response for our configure call should let both Elixir know that the call statement was ok and that the configuration change was also ok, thus the double `:ok` at the start. Next, we'll need to return out the new modified state as a result of this new configure call, so we'll make a call to a new variation of configure that allows configuration after the backend has already been started. Since this is a relatively simple example, we'll only worry about being able to reconfigure the log level, but you could easily add more to it. We'll just need to take the newly passed-in level and merge that with the existing state. We'll also need a fall-through statement just in case someone tries to configure an option that doesn't exist:

    defp configure(_name, [level: new_level], state) do
      Map.merge(state, %{level: new_level})
    end
    defp configure(_name, _opts, state), do: state
    
Finally, we'll write the event handler for when a log is passed in to us! Any log events are passed in the following format:

    { log_level, group_leader, {Logger, message, timestamp, metadata} }
    
And, of course, we'll need to include the state for any `handle_?` functions, so our function definition should be `def handle_event({level, _group_leader, {Logger, message, timestamp, metadata}}, %{level: min_level}=state) do`. In our case, we don't actually care about who or what the group leader is, so we toss that argument out and focus on the rest. We also need to verify if the log level being passed in relation to the configured minimum log level is appropriate, so we'll need to pass that in as well. Writing this in a separate helper function makes your logic-handling a little easier (since if you receive a nil as the configured minimum log level just means all log messages should be handled), so we'll add a new function called `right_log_level?` that takes in the minimum level to compare it to and the log level being called as part of the log event itself. For the actual comparison, we can use the Logger utility function called `compare_levels` that takes in the log level from the message and the log level configured to tell you if the level is appropriate or not. If it's less than the configured level, it will return `:lt`, so we can use this as our check:

    defp right_log_level?(nil, _level), do: true
    defp right_log_level?(min_level, level) do
      Logger.compare_levels(level, min_level) != :lt
    end

Finally, if it IS an appropriate log message for our configured level, we'll just keep our simple and run it through our log formatter and then output it via `IO.puts/1`, although we could easily start to modify the behavior and write it to a file, or send it somewhere. We'll keep our behavior simple, but you can always spice things up to accomodate whatever needs you may specifically have for logging:

    # Handle any log messages that are sent across
    def handle_event({level, _group_leader, {Logger, message, timestamp, metadata}}, %{level: min_level}=state) do
      if right_log_level?(min_level, level) do
        Runnable.LogFormatter.format(level, message, timestamp, metadata)
        |> IO.puts()
      end
      {:ok, state}
    end
    
The only other thing we need to have is to add our new backend to Logger so that it knows how to start using it, so our new top-level Logger config should look like this:

    config :logger,
      backends: [
        :console,
        {Runnable.LogBackend, :log_backend}
      ],
      level: :debug,
      compile_time_purge_level: :debug
      
To verify config works, we'll also toss a configuration in for our backend:

    config :logger, :log_backend,
      level: :info

Finally, let's go back to our Runnable module `lib/runnable.ex`, and in the log function, let's add some code that will hit both of our logging backends:

    defp log do
      Logger.metadata(request_id: "ABCDEF")
      Logger.debug "Starting Application..."
      Logger.info "I should get output twice"
    end
    
When we compile and run our application with `mix run`, we should see some fancy new output:

    $ mix run
    Compiling 1 file (.ex)
    ##### 2018-08-02T11:02:12.070Z request_id=(REDACTED) [debug] Starting Application...
    ##### 2018-08-02T11:02:12.070Z pid=#PID<0.131.0> request_id=(REDACTED) application=:runnable module=Runnable function="log/0" file="/Users/brandon.richey/Development/elixir/runnable/lib/runnable.ex" line=29 [info] I should get output twice
    
    ##### 2018-08-02T11:02:12.070Z request_id=(REDACTED) [info] I should get output twice
    
And that's it! We have a working (but very barebones) logging backend written from scratch and able to handle log events simultaneously with Logger's Console backend!

## Changes in Elixir v1.7

The biggest change relevant to this guide in Elixir v1.7 is the change to Logger's compile-time purging options. Gone is the `:compile_time_purge_level` configuration, replaced instead with `:compile_time_purge_matching`, which allows you to not only purge on a specific log level, but also on any other metadata that may be present in your logging call.

For example, a v1.6 configuration option for logging written as:

    configure :logger,
      compile_time_purge_level: :debug
      
Would instead be written as:

    configure :logger,
        compile_time_purge_matching: [
            [level_lower_than: :info]
        ]
        
An example from the Elixir release notes also includes how to match and purge based on metadata as well:

    config :logger,
      compile_time_purge_matching: [
        [application: :foo, level_lower_than: :info],
        [module: Bar, function: "foo/3"]
      ]

For more information, [read the release notes](https://elixir-lang.org/blog/2018/07/25/elixir-v1-7-0-released/)!


## Summary

Now that you really understand the inner workings of Logging in Elixir, you can really do some amazing things and make sure that Logging is treated in an idiomatic way in all of your Elixir projects. We never brought Phoenix or any other major frameworks into our application and still are able to log everything that happens in our application with no major issues.

Elixir v1.7 brings a few new changes to how logging is handled (albeit not a ton), so we're also up and ready to go and work with the latest and greatest in Elixir logging technology! Logging is often one of your first lines of defense, not only in development, but in the more long-term running and maintenance of your application. It pays to take the time to understand the problem and do it right, and it pays even more to understand the Logging ecosystem that your Elixir application provides to you out of the box!

If you're interested in checking out the final code for this tutorial, the project is available on Github at [https://github.com/richeyb/runnable-elixir](https://github.com/richeyb/runnable-elixir)! Just clone that repo and check out the the `logging_example` branch and you'll have everything you need to see our Logging code, including the custom backend and formatter, in action!