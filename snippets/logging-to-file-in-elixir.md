Interested in learning how to log to a file in Elixir? Here's a quick tutorial:

With the hex package `logger_file_backend`, it's easy to start logging to a file.

```elixir
config :logger, 
    backends: [{LoggerFileBackend, :error_log}]
config :logger, :error_log, 
    path: 'myLog.log'
```

By default, the logger ignores 2 of the warning levels (debug & info) and only logs warnings and errors. Here's how you can change that:

```elixir
config :logger,
    backends: [{LoggerFileBackend, :debug_log}]
config :logger, :debug_log,
    path: 'debugLog.log',
    level: :debug
```

_Shameless plug_: we're a logging company here @ Timber. We've got a great product that makes it easy to collect and analyze your logs, while automatically capturing context in the process. You can read more about our library [here](https://docs.timber.io/languages/elixir/), to be used when getting staging and production logs. 