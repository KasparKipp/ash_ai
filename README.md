# Ash AI
<img src="https://github.com/ash-project/ash_ai/blob/main/logos/ash_ai.png?raw=true" alt="Logo" width="300"/>

## MCP (Model Context Protocol) Server

Both the dev & production MCP servers can be installed with

`mix ash_ai.gen.mcp`

### Dev MCP Server

To install the dev MCP server, add the `AshAi.Mcp.Dev` plug to your
endpoint module, in the `code_reloading?` block. By default the
mcp server will be available under `http://localhost:4000/ash_ai/mcp`.



```elixir
  if code_reloading? do
    socket "/phoenix/live_reload/socket", Phoenix.LiveReloader.Socket

    plug AshAi.Mcp.Dev,
      # see the note below on protocol versions below
      protocol_version_statement: "2024-11-05",
      otp_app: :your_app
```

We are still experimenting to see what tools (if any) are useful while developing with agents.

### Production MCP Server

AshAi provides a pre-built MCP server that can be used to expose your tool definitions to an MCP client (typically some kind of IDE, or Claude Desktop for example).

The protocol version we implement is 2025-03-26. As of this writing, many tools have not yet been updated to support this version. You will generally need to use some kind of proxy until tools have been updated accordingly. We suggest this one, provided by tidewave. https://github.com/tidewave-ai/mcp_proxy_rust#installation

However, as of the writing of this guide, it requires setting a previous protocol version as noted above.

#### Roadmap

- Implement OAuth2 flow with AshAuthentication (long term)
- Implement support for more than just tools, i.e resources etc.
- Implement sessions, and provide a session id context to tools (this code is just commented out, and can be uncommented, just needs timeout logic for inactive sesions)

#### Installation

##### Authentication

We don't currently support the OAuth2 flow out of the box with AshAi, but the goal is to eventually support this with AshAuthentication. You can always implement that yourself, but the quikest way to value is to use the new `api_key` strategy.

Use `mix ash_authentication.add_strategy api_key` to install it if you haven't already.

Then, create a separate pipeline for `:mcp`, and add the api key plug to it:

```elixir
pipeline :mcp do
  plug AshAuthentication.Strategy.ApiKey.Plug,
    resource: YourApp.Accounts.User,
    # Use `required?: false` to allow unauthenticated
    # users to connect, for example if some tools
    # are publicly accessible.
    required?: false
end
```

##### Add the MCP server to your router

```elixir
scope "/mcp" do
  pipe_through :mcp

  forward "/", AshAi.Mcp.Router,
    tools: [
      :list,
      :of,
      :tools
    ],
    # For many tools, you will need to set the `protocol_version_statement` to the older version.
    protocol_version_statement: "2024-11-05",
    otp_app: :my_app
end
```

## `mix ash_ai.gen.chat`

This is a new and experimental tool to generate a chat feature for your Ash & Phoenix application. It is backed by `ash_oban` and `ash_postgres`, using `pub_sub` to stream messages to the client. This is primarily a tool to get started with chat features and is by no means intended to handle every case you can come up with.

To get started:
```
mix ash_ai.gen.chat --live
```

The `--live` flag indicates that you wish to generate liveviews in addition to the chat resources.

It currently requires a `user` resource to exist. If your `user` resource is not called `<YourApp>.Accounts.User`, provide a custom user resource with the `--user`
flag.

To try it out from scratch:

```sh
mix igniter.new my_app \
  --with phx.new \
  --install ash,ash_postgres,ash_phoenix \
  --install ash_authentication_phoenix,ash_oban \
  --install ash_ai@github:ash-project/ash_ai \
  --auth-strategy password
```

and then run:

```sh
mix ash_ai.gen.chat --live
```

You can then start your server and visit `http://localhost:4000/chat` to see the chat feature in action. You will be prompted to register first and sign in the first time.

## Expose actions as tool calls

```elixir
defmodule MyApp.Blog do
  use Ash.Domain, extensions: [AshAi]

  tools do
    tool :read_posts, MyApp.Blog.Post, :read
    tool :create_post, MyApp.Blog.Post, :create
    tool :publish_post, MyApp.Blog.Post, :publish
    tool :read_comments, MyApp.Blog.Comment, :read
  end
end
```

Expose these actions as tools. When you call `AshAi.setup_ash_ai(chain, opts)`, or `AshAi.iex_chat/2`
it will add those as tool calls to the agent.

## Prompt-backed actions

This allows defining an action, including input and output types, and delegating the
implementation to an LLM. We use structured outputs to ensure that it always returns
the correct data type. We also derive a default prompt from the action description and
action inputs. See `AshAi.Actions.Prompt` for more information.

```elixir
action :analyze_sentiment, :atom do
  constraints one_of: [:positive, :negative]

  description """
  Analyzes the sentiment of a given piece of text to determine if it is overall positive or negative.
  """

  argument :text, :string do
    allow_nil? false
    description "The text for analysis"
  end

  run prompt(
    LangChain.ChatModels.ChatOpenAI.new!(%{ model: "gpt-4o"}),
    # setting `tools: true` allows it to use all exposed tools in your app
    tools: true
    # alternatively you can restrict it to only a set of tools
    # tools: [:list, :of, :tool, :names]
    # provide an optional prompt, which is an EEx template
     # prompt: "Analyze the sentiment of the following text: <%= @input.arguments.description %>",
    # adapter: {Adapter, [some: :opt]}
  )
end
```

## Adapters

Adapters are used to determine how a given LLM fulfills a prompt-backed action. The adapter is guessed automatically from the model where possible.
See `AshAi.Actions.Prompt.Adapter` for more information.

### Setting up LangChain

For any langchain models you use, you will need to configure them. See https://hexdocs.pm/langchain/ for more information.

## Vectorization

See `AshPostgres` vector setup for required steps: https://hexdocs.pm/ash_postgres/AshPostgres.Extensions.Vector.html

This extension creates a vector search action, and provides a few different strategies for how to
update the embeddings when needed.

```elixir
# in a resource

vectorize do
  full_text do
    text(fn record ->
      """
      Name: #{record.name}
      Biography: #{record.biography}
      """
    end)

    # When used_attributes are defined, embeddings will only be rebuilt when
    # the listed attributes are changed in an update action.
    used_attributes [:name, :biography]
  end

  strategy :after_action
  attributes(name: :vectorized_name, biography: :vectorized_biography)

  # See the section below on defining an embedding model
  embedding_model MyApp.OpenAiEmbeddingModel
end
```

If you are using policies, add a bypass to allow us to update the vector embeddings:

```elixir
bypass action(:ash_ai_update_embeddings) do
  authorize_if AshAi.Checks.ActorIsAshAi
end
```

## Vectorization strategies

Currently there are three strategies to choose from:

- `:after_action` (default) - The embeddings will be updated synchronously on after every create & update action.
- `:ash_oban` - Embeddings will be updated asynchronously through an `ash_oban`-trigger when a record is created and updated.
- `:manual` - The embeddings will not be automatically updated in any way.

### `:after_action`

Will add a global change on the resource, that will run a generated action named `:ash_ai_update_embeddings`
on every update that requires the embeddings to be rebuilt. The `:ash_ai_update_embeddings`-action will be run in the `after_transaction`-phase of any create action and update action that requires the embeddings to be rebuilt.

This will make your app incredibly slow, and is not recommended for any real production usage.

### `:ash_oban`

Requires the `ash_oban`-dependency to be installed, and that the resource in question uses it as an extension, like this:

```elixir
defmodule MyApp.Artist do
  use Ash.Resource, extensions: [AshAi, AshOban]
end
```

Just like the `:after_action`-strategy, this strategy creates an `:ash_ai_update_embeddings` update-action, and adds a global change that will run an `ash_oban`-trigger (also in the `after_transaction`-phase) whenever embeddings need to be rebuilt.

You will to define this trigger yourself, and then reference it in the `vectorize`-section like this:

```elixir
defmodule MyApp.Artist do
  use Ash.Resource, extensions: [AshAi, AshOban]

  vectorize do
    full_text do
      ...
    end

    strategy :ash_oban
    ash_oban_trigger_name :my_vectorize_trigger (default name is :ash_ai_update_embeddings)
    ...
  end

  oban do
    triggers do
      trigger :my_vectorize_trigger do
        action :ash_ai_update_embeddings
        worker_read_action :read
        worker_module_name __MODULE__.AshOban.Worker.UpdateEmbeddings
        scheduler_module_name __MODULE__.AshOban.Scheduler.UpdateEmbeddings
        scheduler_cron nil
        list_tenants MyApp.ListTenants
      end
    end
  end
end
```

### `:manual`

Will not automatically update the embeddings in any way, but will by default generated an update action
named `:ash_ai_update_embeddings` that can be run on demand. If needed, you can also disable the
generation of this action like this:

```elixir
vectorize do
  full_text do
    ...
  end

  strategy :manual
  define_update_action_for_manual_strategy? false
  ...
end
```

### Embedding Models

Embedding models are modules that are in charge of defining what the dimensions
are of a given vector and how to generate one. This example uses `Req` to
generate embeddings using `OpenAi`. To use it, you'd need to install `req`
(`mix igniter.install req`).

```elixir
defmodule Tunez.OpenAIEmbeddingModel do
  use AshAi.EmbeddingModel

  @impl true
  def dimensions(_opts), do: 3072

  @impl true
  def generate(texts, _opts) do
    api_key = System.fetch_env!("OPEN_AI_API_KEY")

    headers = [
      {"Authorization", "Bearer #{api_key}"},
      {"Content-Type", "application/json"}
    ]

    body = %{
      "input" => texts,
      "model" => "text-embedding-3-large"
    }

    response =
      Req.post!("https://api.openai.com/v1/embeddings",
        json: body,
        headers: headers
      )

    case response.status do
      200 ->
        response.body["data"]
        |> Enum.map(fn %{"embedding" => embedding} -> embedding end)
        |> then(&{:ok, &1})

      _status ->
        {:error, response.body}
    end
  end
end
```

Opts can be used to make embedding models that are dynamic depending on the resource, i.e

```elixir
embedding_model {MyApp.OpenAiEmbeddingModel, model: "a-specific-model"}
```

Those opts are available in the `_opts` argument to functions on your embedding model

## Using the vectors

You can use expressions in filters and sorts like `vector_cosine_distance(full_text_vector, ^search_vector)`. For example:

```elixir
read :search do
  argument :query, :string, allow_nil?: false

  prepare before_action(fn query, context ->
    case YourEmbeddingModel.generate([query.arguments.query], []) do
      {:ok, [search_vector]} ->
        Ash.Query.filter(
          query,
          vector_cosine_distance(full_text_vector, ^search_vector) < 0.5
        )
        |> Ash.Query.sort(
          {calc(vector_cosine_distance(full_text_vector, ^search_vector),
             type: :float
           ), :asc}
        )
        |> Ash.Query.limit(10)

      {:error, error} ->
        {:error, error}
    end
  end)
end
```

# Roadmap

- more action types, like:
  - bulk updates
  - bulk destroys
  - bulk creates.

# How to play with it

1. Setup `LangChain`
2. Modify a `LangChain` using `AshAi.setup_ash_ai/2` or use `AshAi.iex_chat` (see below)
2. Run `iex -S mix` and then run `AshAi.iex_chat` to start chatting with your app.
3. Build your own chat interface. See the implementation of `AshAi.iex_chat` to see how its done.

## Contributing 

1. make sure to run `mix test.create && mix test.migrate` to set up locally
1. ensure that `mix check` passes

## Using AshAi.iex_chat

```elixir
defmodule MyApp.ChatBot do
  alias LangChain.Chains.LLMChain
  alias LangChain.ChatModels.ChatOpenAI

  def iex_chat(actor \\ nil) do
    %{
      llm: ChatOpenAI.new!(%{model: "gpt-4o", stream: true}),
      verbose: true
    }
    |> LLMChain.new!()
    |> AshAi.iex_chat(actor: actor, otp_app: :my_app)
  end
end
```
