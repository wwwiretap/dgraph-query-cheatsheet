# GraphQL+- Cheetsheet

[reference](https://tour.dgraph.io)

## Data import

### Load Schema

Define the `predicate` property
```GraphQL
name: string @index(term) @lang .
age: int @index(int) .
friend: uid @count .
```
| predicate | type | indices |
| --------- | ---- | ------- |
| name | string | term |
| age | int | int |
|friend | uid | count |

### Load Data
Syntex:
```GraphQL
{
    set{
        _: michael <name> "Michael" .
        _: michael <age> "39" .
        _: michael <friend> _: amit .

        _amit <name> "Amit"@en .
        _amit <age> "30 .
    }
}
```

## Query syntex and examples

### Query Example 1 - using name
```
{
  find_michael(func: eq(name@., "Michael")) {
    uid
    name@.
    age
  }
}
```

`find_michael` customised query alias.

`func: ...` matches nodes.

`eq` => equal. does what you'd expect. `eq(name@., "Michael")` will match nodes wiith a `name` equalling `"Michael"`.

```
{
    uid
    name@.
    age
}
```
defines the field each query result will contain.

### Query Example 2 - using uid
```
{
  find_uid(func:uid("0xf6965")){
    name@.
    age
  }
}
```
`uid` can be directly queried. Query results will contain 2 fields, the `name@.` and `age`.

### Query Example 3 - results as graph

```
{
  michaels_friends(func: eq(name, "Michael")) {
    name
    age
    friend {
      name@.
    }
  }
}
```
This query starts from a node with `name` equals to `"Michael"` and finds its `name`, `age` and all the nodes(its `name` attribute will be returned) that are linked to the root node through `friend` predicate.

### Query Example 4 - Get predicate information
```
schema(pred: [name, age, friend, owns_pet]) {
  type
  index
}
```
`type` query the predicate data type.
`index` -> bool, indicate if the predicate is indexed or not.

### Query Example 5 - UTF-8 support
When do the triplet insertion, it can be taged with different language tags, such as:
```
    _:amit <name> "अमित"@hi .
    _:amit <name> "অমিত"@bn .
    _:amit <name> "Amit"@en .
    _:amit <name> "Amitssss" .
```
In the query, `predicate@tag` can be used to find specific nodes, such as:
```
{
  language_support(func: allofterms(name@hi, "अमित")) {
    name@bn:hi:en
    age
    friend {
      name@ko:ru
      age
    }
  }
}
```
`name@bn:hi:en` will return `অমিত` from `_:amit <name> "অমিত"@bn .`. If associate name has no `bn, hi, en` defined, it will not return anything. if `.` is added, as`name@bn:.`, it will return the `Amitssss` from `_:amit <name> "Amitssss" .`.

### Query Example 6 - {allOfTerms, anyOfTerms} - Edge logical operators
- **allOfTerms**

    `allOfTerms(edge_name, "term1 ... termN")`: Matches nodes with an outgoinf `string` edge with `edge_name` where the string contains all listed terms.
- **anyOfTerms**

    `anyOfTerms(edge_name, "term1 ... termN")`: As with allOfTerms, but matches at least one term.

### Query Example 7 - {eq, ge, le, gt, lt} - Edge logical operators
These operators can be applied to edges of type : `int, float, string, date`
```
eq(edge_name, value) # equal to
ge(edge_name, value) # greater than or equal to
le(edge_name, value) # less than or equal to
gt(edge_name, value) # greater than
lt(edge_name, value) # less than
```

### Query Example 8 - {AND, OR, NOT} - logical connectives for filter
```
{
  michael_friends_and(func: allofterms(name, "Michael")) {
    name
    age
    friend @filter(ge(age, 27) AND le(age, 48)) {
      name@.
      age
    }
  }
}
```

### Query Example 9 - {orderasc, orderdesc} - sorting
```
{
  michael_friends_sorted(func: allofterms(name, "Michael")) {
    name
    age
    friend (orderasc: age) {
      name@.
      age
    }
  }
}
```
The `friend` will be ordered as ascending order by age.

### Query Example 10 - {first, offset, after} - pagination

It’s not uncommon to have thousands of results for a query.
But you might want to select only the top-k answers, paginate the results for display, or limit a large result.

- `first: N` Return only the first N results
- `offset: N` Skip the first N results
- `after: uid` Return the results after uid

```
{
  michael_friends_first(func: allofterms(name, "Michael")) {
    name
    age
    friend (orderasc: name@., offset: 1, first: 2) {
      name@.
    }
  }
}
```

### Query Example 11 - {count} - count
```
{
  michael_number_friends(func: allofterms(name, "Michael")) {
    name
    age
    count(friend)
  }
}
```
This will return the total numbers of outgoing edges rather than a list of edges.

## Understand Dgraph

### Dgraph root `func: `
The graphs in Dgraph can be huge, so starting searching from all nodes isn’t efficient. Dgraph needs a place to start searching, that’s the `root node`.

At root, we use `func:` and a function to find an initial set of nodes. So far we’ve used `eq` and `allofterms` for string search, but we can also search on other values like `dates, numbers`, and also **filters** on `count`.

Hence, root node should be indexed otherwise dgraph has to search through all the databases to find all matching values.

The **root** `func:` only accepts a single function and doesn’t accept `AND, OR and NOT` connectives as in `filters`. So the syntax `(func: ...) @filter(... AND ...)` is required when filtering on multiple properties at the root.

Example query: find all people who are in their 20s, and have at least 2 friends.
```
{
  lots_of_friends(func: ge(count(friend), 2)) @filter(ge(age, 20) AND le(age, 30)) {
    name@.
    age
    friend {
      name@.
    }
  }
}
```

### {has} function
The function `has(edge_name)` returns nodes that have an outgoing edge of the given name.
```
{
  have_friends(func: has(friend)) {
    name@.
    age
    number_of_friends : count(friend)
  }
}
```

### Alias capability[?](https://tour.dgraph.io/basic/13/)
The output graph can set names for edges in the output with aliasing.
```
{
  michael_number_friends(func: allofterms(name, "Michael")) {
    persons_name : name
    age
    number_of_friends : count(friend)
  }
}
```
Here, `persons_name` is an alias of `name`, `number_of_friend` is also an alias.

### {cascade} directive
The `@cascade` directive removes any nodes that don’t have all matching edges in the query.

Example explain:
```
{
  michael_friends(func: allofterms(name, "Michael")) {
    name
    age
    friend {
      name@.
      friend @filter(ge(age, 27)) {
        name@.
        age
      }
    }
  }
}
```
In the query above, Dgraph returns all Michael’s friends, and only the friends of friends who are over 27.

If we apply the `cascade` directive:
```
{
  michael_friends(func: allofterms(name, "Michael")) @cascade {
    name
    age
    friend {
      name@.
      friend @filter(ge(age, 27)) {
        name@.
        age
      }
    }
  }
}
```
With the `@cascade` directive, friends of Michael that don’t have friends who are over 27 are not included in the result.

### {normalize} directive
The `@normalize` directive
- returns only edges listed with an alias
- flattens the result to remove nesting

Example:
```
{
  michael_number_friends(func: allofterms(name, "Michael")) @normalize {
    name : name
    age
    number_of_friends : count(friend)
  }
}
```
`name` and `number_of_friends` are aliases.
Sample response
```JSON
"data": {
    "michael_number_friends": [
      {
        "name": "Michael",
        "number_of_friends": 5
      }
    ]
  },
```

### Query comments
Anything after `#` on a line is a comment and ignored for query processing.

### Facets: edge attributes
Dgraph supports **facets** — `key value pairs on edges` — as an extension to RDF triples. That is, facets add properties to *edges*, rather than to *nodes*. For example, a `friend` edge between two nodes may have a boolean property of `close` friendship. Facets can also be used as `weights` for edges.

Though you may find yourself leaning towards facets many times, they should not be misused. It wouldn’t be correct modeling to give the `friend` edge a facet `date_of_birth`. That should be an edge for the friend. However, a facet like `start_of_friendship` might be appropriate. Facets are however *not first class citizen* in Dgraph like predicates. [detail examples](https://docs.dgraph.io/query-language/#facets-edge-attributes)

Facet keys are strings and values can be `string, bool, int, float and dateTime`. For `int` and `float`, only decimal integers upto 32 signed bits, and 64 bit float values are accepted respectively.

The following mutation is used throughout this section on facets. The mutation adds data for some peoples and, for example, records a `since` facet in `mobile` and `car` to record when Alice bought the car and started using the mobile number.

Example:

- Create schema
    ```
    name: string @index(exact, term) .
    rated: uid @reverse @count .
    ```
- Load data
    ```
    {
        set {

            # -- Facets on scalar predicates
            _:alice <name> "Alice" .
            _:alice <mobile> "040123456" (since=2006-01-02T15:04:05) .
            _:alice <car> "MA0123" (since=2006-02-02T13:01:09, first=true) .

            _:bob <name> "Bob" .
            _:bob <car> "MA0134" (since=2006-02-02T13:01:09) .

            _:charlie <name> "Charlie" .
            _:dave <name> "Dave" .


            # -- Facets on UID predicates
            _:alice <friend> _:bob (close=true, relative=false) .
            _:alice <friend> _:charlie (close=false, relative=true) .
            _:alice <friend> _:dave (close=true, relative=true) .


            # -- Facets for variable propagation
            _:movie1 <name> "Movie 1" .
            _:movie2 <name> "Movie 2" .
            _:movie3 <name> "Movie 3" .

            _:alice <rated> _:movie1 (rating=3) .
            _:alice <rated> _:movie2 (rating=2) .
            _:alice <rated> _:movie3 (rating=5) .

            _:bob <rated> _:movie1 (rating=5) .
            _:bob <rated> _:movie2 (rating=5) .
            _:bob <rated> _:movie3 (rating=5) .

            _:charlie <rated> _:movie1 (rating=2) .
            _:charlie <rated> _:movie2 (rating=5) .
            _:charlie <rated> _:movie3 (rating=1) .
        }
    }
    ```
- query facets, facets on scalar predicates

    The syntax `@facets(facet-name)` is used to query facet data. For Alice the `since` facet for `mobile` and `car` are queried as follows.
    ```
    {
        data(func: eq(name, "Alice")) {
            name
            mobile @facets(since)
            car @facets(since)
        }
    }
    ```
    Response for this query is:
    ```JSON
    {
        "data": {
            "data": [
            {
                "name": "Alice",
                "mobile|since": "2006-01-02T15:04:05Z",
                "mobile": "40123456",
                "car|since": "2006-02-02T13:01:09Z",
                "car": "MA0123"
            }
            ]
        }
    }
    ```
    Facets are retuned at the same level as the corresponding edge and have keys like `edge|facet`.

    All facets on an edge are queried with `@facets`.
    ```
    {
        data(func: eq(name, "Alice")) {
            name
            mobile @facets
            car @facets
        }
    }
    ```
    Response:
    ```JSON
    {
        "data": {
            "data": [
            {
                "name": "Alice",
                "mobile|since": "2006-01-02T15:04:05Z",
                "mobile": "40123456",
                "car|first": true,
                "car|since": "2006-02-02T13:01:09Z",
                "car": "MA0123"
            }
            ]
        }
    }
    ```
- TODO: Alias with facets
- TODO: Facets on UID predicates
- TODO: Filtering on facets
- TODO: Sorting using facets
- TODO: Assigning Facet values to a variable
- TODO: Facets and Variable Propagation
- TODO: Facets and Aggregation

### Add multiple attributes to a third single node
Let’s say You have a existing Node that’s your target his UID is `“0x0f43”`. So you need to setup your RDF like this.

```
_:MyNode <predicate1> <0x0f43> ( foo=bar, and=1 ) .
_:MyNode <predicate2> <0x0f43> ( bar=bar, and=2 ) .
_:MyNode <predicate3> <0x0f43> ( bar=bar, and=3 ) .
```