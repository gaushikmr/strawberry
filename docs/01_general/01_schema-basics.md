---
title: Schema Basics
path: /docs/schema-basics
---

# Schema basics

GraphQL servers use a **schema** to describe the shape of the data. The schema
defines a hierarchy of **types** with fields that are populated from data
stores. The schema also specifies exactly which queries and mutations are
available for clients to execute.

This guide describes the basic building blocks of a schema and how to use
Strawberry to create one.

## Schema definition language (SDL)

There are two approaches for creating the schema for a GraphQL server. One is
called “schema-first” and the other is called “code-first”. Strawberry _only_
supports code-first schemas. Before diving into code-first, let’s first explain
what the Schema definition language is.

Schema first works using the Schema Definition Language of GraphQL, which is
included in the GraphQL spec.

Here’s an example of schema defined using the SDL:

```graphql
type Book {
  title: String!
  author: Author!
}

type Author {
  name: String!
  books: [Book!]!
}
```

The schema defines all the types and relationships between them. With this we
enable client developers to see exactly what data is available and request a
specific subset of that data.

Note: the ! sign specifies that a field is non-nullable.

Note that the schema doesn’t specify how to get the data. That comes later when
defining the resolvers.

<!-- TODO: plug Ariadne for a schema-first alternative -->

## Code first approach

As mentioned Strawberry uses a code first approach. The previous schema would
look like this in Strawberry

```python
from __future__ import annotations

import typing
import strawberry

@strawberry.type
class Book:
  title: str
  author: Author

@strawberry.type
class Author:
  name: str
  books: typing.List['Book']
```

As you can see the code maps almost one to one with the schema, thanks to
python’s type hints feature.

The `__future__` import allows us to annotate types that are defined later in
the file, as per [PEP 563](https://www.python.org/dev/peps/pep-0563/). If you
can’t use this feature (because your other code is incompatible with it), you
need to quote the `Author` annotation.

```python
    ...
    author: 'Author'
    ...
```

Note that here we are also not specifying how to fetch data, that will be
explained in the resolvers section.

## Supported types

GraphQL supports a few different types:

- Scalar types
- Object types
- The Query type
- The Mutation type
- Input types

## Scalar types

<!--alex ignore-->

Scalar types are similar to python primitive types. Here’s the list of the
default scalar types in GraphQL:

1. Int, a signed 32-bit integer, maps to python’s int
2. Float, a signed double-precision floating-point value, maps to python’s float
3. String, maps to python’s str
4. Boolean, true or false, maps to python’s bool
5. ID, a unique identifier that usually used to refetch an object or as the key
   for a cache. Serialized as string and available as `strawberry.ID(“value”)`

> **NOTE:** strawberry also includes support for date, time and datetime
> objects, they are not officially included with the GraphQL spec, but they are
> usually needed in most servers. They are serialized as ISO-8601.

<!--alex ignore-->

These primitives work for the majority for the uses cases, but you can also
specify your [own scalar types](/docs/types/scalars).

## Object types

Most of the types you define in a GraphQL schema are object types. An object
type contains a collection of fields, each of which can be either a scalar type
or another object type.

Object types can refer to each other, as we had in our schema earlier:

```python
import typing
import strawberry

@strawberry.type
class Book:
  title: str
  author: Author

@strawberry.type
class Author:
  name: str
  books: typing.List[Book]
```

## The Query type

The `Query` type defines exactly which GraphQL queries (i.e., read operations)
clients can execute against your data. It resembles an object type, but its name
is always `Query`.

Each field of the `Query` type defines the name and return type of a different
supported query. The `Query` type for our example schema might resemble the
following:

```python
@strawberry.type
class Query:
    books: typing.List[Book]
    authors: typing.List[Author]
```

This Query type defines two available queries: books and authors. Each query
returns a list of the corresponding type.

With a REST-based API, books and authors would probably be returned by different
endpoints (e.g., /api/books and /api/authors). The flexibility of GraphQL
enables clients to query both resources with a single request.

### Structuring a query

When your clients build queries to execute against your data graph, those
queries match the shape of the object types you define in your schema.

Based on our example schema so far, a client could execute the following query,
which requests both a list of all book titles and a list of all author names:

```graphql
query {
  books {
    title
  }

  authors {
    name
  }
}
```

Our server would then respond to the query with results that match the query's
structure, like so:

```json
{
  "data": {
    "books": [{ "title": "Jurassic Park" }],
    "authors": [{ "name": "Michael Crichton" }]
  }
}
```

Although it might be useful in some cases to fetch these two separate lists, a
client would probably prefer to fetch a single list of books, where each book's
author is included in the result.

Because our schema's Book type has an author field of type Author, a client
could structure their query like so:

```graphql
query {
  books {
    title
    author {
      name
    }
  }
}
```

And once again, our server would respond with results that match the query's
structure:

```json
{
  "data": {
    "getBooks": [
      { "title": "Jurassic Park", "author": { "name": "Michael Crichton" } }
    ]
  }
}
```

## The Mutation type

The `Mutation` type is similar in structure and purpose to the Query type.
Whereas the Query type defines your data's supported read operations, the
`Mutation` type defines supported write operations.

Each field of the `Mutation` type defines the signature and return type of a
different mutation. The `Mutation` type for our example schema might resemble
the following:

```python
@strawberry.type
class Mutation:
  @strawberry.field
  def add_book(self, title: str, author: str) -> Book:
    ...
```

This Mutation type defines a single available mutation, `addBook`. The mutation
accepts two arguments (title and author) and returns a newly created Book
object. As you'd expect, this Book object conforms to the structure that we
defined in our schema.

> **NOTE:** strawberry converts fields names from snake case to camel case
> automatically.

### Structuring a mutation

Like queries, mutations match the structure of your schema's type definitions.
The following mutation creates a new Book and requests certain fields of the
created object as a return value:

```graphql
mutation {
  addBook(title: "Fox in Socks", author: "Dr. Seuss") {
    title
    author {
      name
    }
  }
}
```

As with queries, our server would respond to this mutation with a result that
matches the mutation's structure, like so:

```json
{ "data": { "addBook": { "title": "Fox in Socks", "author": { "name": "Dr.
Seuss" } } } }
```

A single client request can include multiple mutations to execute. To prevent
race conditions, mutations are executed serially in the order they're listed.

## Input types

Input types are special object types that allow you to pass objects as arguments
to queries and mutations (as opposed to passing only scalar types). Input types
help keep operation signatures clean.

Consider our previous mutation to add a book:

```python
@strawberry.type
class Mutation:
  @strawberry.field
  def add_book(self, title: str, author: str) -> Book:
    ...
```

Instead of accepting two arguments, this mutation could accept a single input
type that includes all of these fields. This comes in extra handy if we decide
to accept an additional argument in the future, such as a publication date.

An input type's definition is similar to an object type's, but it uses the input
keyword:

```python
@strawberry.input:
class AddBookInput:
  title: str
  author: str


@strawberry.type
class Mutation:
  @strawberry.field
  def add_book(self, book: AddBookInput) -> Book:
    ...
```

Not only does this facilitate passing the PostAndMediaInput type around within
our schema, it also provides a basis for annotating fields with descriptions
that are automatically exposed by GraphQL-enabled tools:

```python
@strawberry.input:
class AddBookInput:
  title: str = strawberry.field(description="The title of the book")
  author: str = strawberry.field(description="The name of the author")
```

Input types can sometimes be useful when multiple operations require the exact
same set of information, but you should reuse them sparingly. Operations might
eventually diverge in their sets of required arguments.

## More

If you want to learn more about schema design make sure you follow the
[documentation provided by Apollo](https://www.apollographql.com/docs/apollo-server/schema/schema/#growing-with-a-schema).
