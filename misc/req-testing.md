# Testing HTTP Adapters

## Context

HTTP adapters need tests that cover the full pipeline: request construction, response parsing, and error handling. Mock modules and adapter injection test the wrong thing — they bypass the real code. `Req.Test` intercepts at the plug level, letting the real adapter code run end-to-end.

## Decision

Use `Req.Test` stubs for all HTTP adapter tests. Configure the stub plug via application config in `config/test.exs` — no per-test environment mutation. Always disable retries in test config.

## Do

- Configure the test plug in `config/test.exs` via application config on the adapter
- Merge req options from application config in the adapter so tests can inject the stub
- Set `retry: false` in test req options — Req retries on transport errors by default
- Test three cases per adapter: success, API error response, transport error
- Use `async: true` — `Req.Test` stubs are concurrent-safe

## Don't

- Don't use `Application.put_env` per-test to inject adapters
- Don't write mock modules that replace the adapter — test the real code path
- Don't forget `retry: false` — slow tests and log noise follow

## Examples

Adapter implementation — merge req options from config:

```elixir
defmodule MyApp.Adapters.PriceAPI do
  def fetch(symbol) do
    options = Application.get_env(:my_app, :price_api_req_options, [])
    req = Req.new(url: "https://api.example.com/price/#{symbol}")
    Req.get(req, options)
  end
end
```

`config/test.exs`:

```elixir
config :my_app, :price_api_req_options,
  plug: {Req.Test, MyApp.Adapters.PriceAPI},
  retry: false
```

Adapter test:

```elixir
defmodule MyApp.Adapters.PriceAPITest do
  use ExUnit.Case, async: true

  alias MyApp.Adapters.PriceAPI

  describe "fetch/1" do
    test "returns parsed price on success" do
      Req.Test.stub(MyApp.Adapters.PriceAPI, fn conn ->
        Req.Test.json(conn, %{"price" => 24.5, "currency" => "USD"})
      end)

      assert {:ok, %{price: 24.5}} = PriceAPI.fetch("XAGUSD")
    end

    test "returns error on API error response" do
      Req.Test.stub(MyApp.Adapters.PriceAPI, fn conn ->
        Req.Test.json(conn, %{"error" => "invalid_symbol"})
        |> Plug.Conn.send_resp(422, "")
      end)

      assert {:error, _} = PriceAPI.fetch("INVALID")
    end

    test "returns error on transport failure" do
      Req.Test.stub(MyApp.Adapters.PriceAPI, fn conn ->
        Req.Test.transport_error(conn, :timeout)
      end)

      assert {:error, %Req.TransportError{reason: :timeout}} = PriceAPI.fetch("XAGUSD")
    end
  end
end
```

See also: [[context]], [[liveview]]
