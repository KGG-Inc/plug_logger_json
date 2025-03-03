# PlugLoggerJson
[![Hex pm](http://img.shields.io/hexpm/v/plug_logger_json.svg?style=flat)](https://hex.pm/packages/plug_logger_json)
[![Build Status](https://travis-ci.org/bleacherreport/plug_logger_json.svg?branch=master)](https://travis-ci.org/bleacherreport/plug_logger_json)
[![License](https://img.shields.io/badge/license-Apache%202-blue.svg)](https://github.com/bleacherreport/plug_logger_json/blob/master/LICENSE)

A comprehensive JSON logger Plug.

## Dependencies

* Plug

## SwayDM Changes
Poison was removed from this library.  The log messages now output only "Phoenix Request Log", and
all other data captured in the plug is passed as metadata attributes.

Also, the duration of the request is now measured using System.monotonic(:nanosecond), rather than
:os time.  This should be more accurate.

## Metadata
All captured metadata is under the :plug metadata attribute.
```
[
    plug: %{
            duration_nano: 34624763,
            phoenix: %{
                method: "GET",
                path: "/wallet",
                status: 200,
                controller: "SwayDMWeb.WalletController",
                action: "show"
            },
            
            # If opts[:include_debug_logging] == true
            remote_ip: "127.0.0.1",
            x_forwarded_for: "12.13.14.15",
            client_version: "Mozilla/5.0 (Macinto...",
            params: %{...},            
    }
]
```

## Elixir & Erlang Support

The support policy is to support the last 2 major versions of Erlang and the three last minor versions of Elixir.

## Installation

1. add plug_logger_json to your list of dependencies in `mix.exs`:

   ```elixir
   def deps do
     [{:plug_logger_json, "~> 0.7.0"}]
   end
   ```

2. ensure plug_logger_json is started before your application (Skip if using Elixir 1.4 or greater):

   ```elixir
   def application do
     [applications: [:plug_logger_json]]
   end
   ```

3. Replace `Plug.Logger` with either:

   * `Plug.LoggerJSON, log: Logger.level`,
   * `Plug.LoggerJSON, log: Logger.level, extra_attributes_fn: &MyPlug.extra_attributes/1` in your plug pipeline (in `endpoint.ex` for Phoenix apps),

## Recommended Setup

### Configure `plug_logger_json`

Add to your `config/config.exs` or `config/env_name.exs` if you want to filter params or headers or suppress any logged keys:

```elixir
config :plug_logger_json,
  filtered_keys: ["password", "authorization"],
  suppressed_keys: ["api_version", "log_type"]
```

### Configure the logger (console)

In your `config/config.exs` or `config/env_name.exs`:

```elixir
config :logger, :console,
  format: "$message\n",
  level: :info, # You may want to make this an env variable to change verbosity of the logs
  metadata: [:request_id]
```

### Configure the logger (file)

Do the following:

* update deps in `mix.exs` with the following:

    ```elixir
    def deps do
     [{:logger_file_backend, "~> 0.0.10"}]
    end
    ```

* add to your `config/config.exs` or `config/env_name.exs`:

    ```elixir
    config :logger,
      format: "$message\n",
      backends: [{LoggerFileBackend, :log_file}, :console]

    config :logger, :log_file,
      format: "$message\n",
      level: :info,
      metadata: [:request_id],
      path: "log/my_pipeline.log"
    ```

* ensure you are using `Plug.Parsers` (Phoenix adds this to `endpoint.ex` by default) to parse params as well as request body:

    ```elixir
    plug Plug.Parsers,
      parsers: [:urlencoded, :multipart, :json],
      pass: ["*/*"],
      json_decoder: Poison
    ```

## Error Logging

In `router.ex` of your Phoenix project or in your plug pipeline:

* add `require Logger`,
* add `use Plug.ErrorHandler`,
* add the following two private functions:

    ```elixir
    defp handle_errors(%Plug.Conn{status: 500} = conn, %{kind: kind, reason: reason, stack: stacktrace}) do
      Plug.LoggerJSON.log_error(kind, reason, stacktrace)
      send_resp(conn, 500, Poison.encode!(%{errors: %{detail: "Internal server error"}}))
    end

    defp handle_errors(_, _), do: nil
    ```

## Extra Attributes

Additional data can be logged alongside the request by specifying a function to call which returns a map:

```elixir
def extra_attributes(conn) do
  map = %{
    "user_id" => get_in(conn.assigns, [:user, :user_id]),
    "other_id" => get_in(conn.private, [:private_resource, :id]),
    "should_not_appear" => conn.private[:does_not_exist]
  }

  map
  |> Enum.filter(&(&1 !== nil))
  |> Enum.into(%{})
end

plug Plug.LoggerJSON,
  log: Logger.level(),
  extra_attributes_fn: &MyPlug.extra_attributes/1
```

In this example, the `:user_id` is retrieved from `conn.assigns.user.user_id` and added to the log if it exists. In the example, any values that are `nil` are filtered from the map. It is a requirement that the value is serialiazable as JSON by the Poison library, otherwise an error will be raised when attempting to encode the value.

## Log Verbosity

`LoggerJSON` plug supports two levels of logging:

  * `info` / `error` will log:

    * api_version,
    * date_time,
    * duration,
    * log_type,
    * method,
    * path,
    * request_id,
    * status

  * `warn` / `debug` will log everything from info and:

    * client_ip,
    * client_version,
    * params / request_body.

The above are default. It is possible to override them by setting a `include_debug_logging` option to:

  * `false` – means the extra debug fields (client_ip, client_version, and params) WILL NOT get logged.
  * `true` – means the extra fields WILL get logged.
  * Not setting this option will keep the defaults above.

Example:

```elixir
plug Plug.LoggerJSON,
  log: Logger.level,
  include_debug_logging: true
```

## Contributing

Before submitting your pull request, please run:

  * `mix credo --strict`,
  * `mix coveralls`,
  * `mix dialyzer`,
  *  update changelog.

Please squash your pull request's commits into a single commit with a message and detailed description explaining the commit.
