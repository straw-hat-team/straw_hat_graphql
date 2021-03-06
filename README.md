# StrawHat.GraphQL

[![Build Status](https://travis-ci.org/straw-hat-team/straw_hat_graphql.svg?branch=master)](https://travis-ci.org/straw-hat-team/straw_hat_graphql)
[![codecov](https://codecov.io/gh/straw-hat-team/straw_hat_graphql/branch/master/graph/badge.svg)](https://codecov.io/gh/straw-hat-team/straw_hat_graphql)
[![Inline docs](http://inch-ci.org/github/straw-hat-team/straw_hat_graphql.svg)](http://inch-ci.org/github/straw-hat-team/straw_hat_graphql)

GraphQL utils.

## Installation

```elixir
def deps do
  [
    {:straw_hat_graphql, "~> 0.1"}
  ]
end
```

## Usage

The package come with predefined types that you could use in your project.
Most of those types dictate how your data structure and responses from your
mutations will look like but it will help you to speed up your development.

```elixir
defmodule YourApp.AbsintheSchema do
  use Absinthe.Schema
  use Absinthe.Schema.Notation

  # Should be include it before you use it

  import_types StrawHat.GraphQL.Types
  import_types StrawHat.GraphQL.Scalars

  # ...
end
```

Check `StrawHat.GraphQL.Types` and `StrawHat.GraphQL.Scalars` for more
information about the included types.

### Paginations

There is some types for doing paginations. We follow Scrivener for formatting
the response.

**Note:** For now there is some boilerplate code for the paginations. We are
aware of using macros but to be honest sometimes is better a repeated code
that shoot ourself in the foot, still thinking about creating macros for this.

```elixir
object :person do
  interface :straw_hat_node

  field :id, non_null(:id)

  # ... more fields
end

# We recommend to use `_pagination`, this is another place where we could macros.
object :people_pagination do
  # This will force you to defined the schemas
  interface(:straw_hat_pagination)

  field(:page, non_null(:straw_hat_pagination_page))
  field(:total_entries, non_null(:integer))
  field(:total_pages, non_null(:integer))

  # This is where it gets tricky and probably we could use macros
  # the reason is that we can't put this field inside `:straw_hat_pagination`
  # because the type is not the same for all the objects.
  field(:entries, list_of(:person))
end
```

Now let's create the query.

```elixir
field :people, :people_pagination do
  arg(:pagination, non_null(:straw_hat_pagination_page_input))
  resolve(&PeopleResolver.get_people/2)
end
```

### Mutations

Mutations is something that comes really handy and with a really nice API.

**Note:** For now there is some boilerplate code for the mutations. We are
aware of using macros but to be honest sometimes is better a repeated code
that shoot ourself in the foot, still thinking about creating macros for this.

First, you need to create the mutation response which normally should extend
from the interface of `mutation_response`

```elixir
object :person do
  interface :straw_hat_node

  field :id, non_null(:id)

  # ... more fields
end

# We recommend to use `_payload`, this is another place where we could macros
object :person_payload do
  # This will force you to defined the schemas
  interface :straw_hat_mutation_response

  field :successful, non_null(:boolean)
  field :errors, list_of(:straw_hat_error)

  # This is where it gets tricky and probably we could use macros
  # the reason is that we can't put this field inside `:straw_hat_mutation_response`
  # because the type is not the same for all mutations.
  field :payload, :person
end
```

Now we create the mutation

```elixir
object :person_mutations do
  field :person_update, :person_payload do
    arg :person, non_null(:person_input)

    # other resolvers, maybe authorization
    resolve &MyApp.PersonResolvers.update_person/2
  end
end
```

Now what is left is the resolver function itself

```elixir
defmodule MyApp.PersonResolvers do
  alias StrawHat.GraphQL.MutationResponse

  def update_person(%{person: person_params}, _) do
    response = MyApp.Person.update_person(person_params)

    # We do not want to force anything the responses from your modules
    # it up to you, make sure you call the correct function for format
    # the data structure
    case response do
      {:ok, response} -> MutationResponse.succeeded(response)
      {:error, reason} -> MutationResponse.failed(reason)
    end
  end
end
```

If you want to avoid doing the pattern matching constantly which it is up to your
application design, you could create a function that does it for you

```elixir
defmodule MyApp.ResolverHelpers
  alias StrawHat.GraphQL.MutationResponse

  def response_with({:ok, response}), do: MutationResponse.succeeded(response)
  def response_with({:error, reason}), do: MutationResponse.failed(reason)
  def response_with(_), do: raise(ArgumentError)
end
```

then your resolver becomes

```elixir
import MyApp.ResolverHelpers

def update_person(%{person: person_params}, _) do
  person_params
  |> MyApp.Person.update_person()
  |> response_with()
end
```

And now you are go to go. Or use `StrawHat.GraphQL.MutationResponse.response_with/1`
but be aware that it is constrain by the inputs.

Any suggestion and code pattern that you notice in your app, open an issue
and let's talk about it.
