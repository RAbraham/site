---
categories:
- markdown
date: '2020-05-12'
description: DataScript is an immutable database for application state management in the browser 
layout: post
title: DataScript. A modern datastore for the browser
toc: true

---
This article was originally posted at `https://blog.rajivabraham.com/posts/datascript-python`

# Purpose.
This blog article will:
 * give a brief introduction to a new way of managing state on the browser by leveraging a library called [DataScript](https://github.com/tonsky/datascript).
 * Though DataScript is a ClojureScript/JavaScript library, we will learn how to call it using [Brython](https://blog.rajivabraham.com/posts/brython), a Python implementation that runs on the browser.
 * sneak in a tutorial on [mercylog-datascript](https://github.com/RAbraham/mercylog-datascript) which is a query composer library for DataScript.

# DataScript(and Datalog)

How do you manage application state in the browser? I'm new to frontend development so I'm sure there are many solutions out there which I don't know about. But let me ask the question from a different perspective. How would a backend developer manage state? She uses a database. She writes SQL queries. Then for the browser, can we use an in-memory datastore and write queries on it?

`DataScript` is one such in-memory datastore for the browser. One can create it during page load, insert, delete rows etc. and when the user closes the page, it's gone. It's very light and fast. Also, instead of SQL, it offers a query style called Datalog which is a declarative, logic based query language. I got interested in Datalog when I saw that applications written in its variants reduced the code size by 50% or more([Overlog](https://dl.acm.org/doi/10.1145/1755913.1755937), [Yedalog](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43462.pdf)). This is why I got interested in `DataScript` in the first place. I have written a brief introduction to the concepts [here](https://blog.rajivabraham.com/posts/bashlog). In short, It's like SQL + Recursion.

But Datalog is not the only reason to use DataScript. That would be like flying in the Boeing 747 for its in-flight entertainment. It has so many other cool features but that's for another post. This post will focus on just the Datalog query interface for DataScript.

Just to complicate things a bit, `DataScript` is actually a ClojureScript library which has a `JavaScript` interface too. So in all respects, you would use it cleanly in those languages. *But*, I'm interested in Python in the browser and therefore all the examples below will be in a Python implementation in the browser called [Brython](https://brython.info/). I have written a [brief introduction](https://blog.rajivabraham.com/posts/brython) before:

All the examples below are from the following github [repo](https://github.com/RAbraham/mercylog-datascript-client). NOTE: All the files mentioned below take some time to load the data.


## There is no spoon, Neo. 
Cough, before we move on, DataScript is different from your conventional SQL datastores. In conventional SQL databases, when we wish to implement an entity e.g `customer`,  we first create a table schema `customer` with the `attributes`: `name` and `age` for e.g.
```
CREATE TABLE customer (
    name varchar,
    age int
);

```
According to Datomic, the backend database which DataScript is an in-memory implementation of, such an approach is [rigid](https://docs.datomic.com/on-prem/schema.html). Hey, I sense it but not well enough to defend it. So, let's not have twitter wars on that one... yet. 

DataScript eschews storing data as separate entities in separate tables(like `customer` or `person`). Instead, one can see DataScript as a store for *attributes* and its values only. So, DataScript prescribes that we only specify the schema for the attributes itself. E.g. `name` is `string`, `age` is `integer` but do not group them at all at design time before hand. Instead, the *application decides which attributes to group together for a particular instance of that entity*. 

In an extreme example to clarify the concepts, the application may decide to store just a customer's name(Rajiv) and his eye color(black) and give the customer an id 1. For another customer, it may store her as id 3 and just store her age(40). Simplistically(and naively incorrect), you can see the database as a single god table with columns `entity_id`, `attribute`, `value`  where each row can belong to a different `(entity_id, attribute)` composite key. E.g.

| entity_id     | attribute  | value  |
| ------------- |:----------:| ------:|
| 1             | name       | Rajiv  |
| 3             | age        | 40     |
| 2             | name       | Canada |
| 1             | eye_color  | black  |

What about the entity `2`? He/She is named Canada? Must be a cutie unlike the dull Rajiv. *But we don't know, the application knows*. It may not represent a person at all! In this example, the pair (entity id=`2`, attribute=`name`) could be one attribute and value for an instance of the entity `country` e.g. Canada(*O Canada, our home and native land... True Patriot Love .. with your public healthcare you command!*). Before you panic and ditch Datomic/DataScript, in practice, the attribute names contain the entity name as well. So it'll be ":customer/name" instead of "name" for Rajiv and ":country/name" instead of "name" for Canada. Or you could still leverage a common `name` attribute, your choice. And there is better support for ids than the pitiful example I'm giving here. Also, Imposter Alert, I'm not an expert by any means. I'm learning by doing :). That's my EULA. Please do check out the design principles behind Datomic/DataScript. I think Datomic is a masterpiece in database engineering.

Let's show you how to add data. For e.g., we can add two dictionaries(Igor and Ivan) to the database. 

```
db = datascript.empty_db()
db1 = datascript.db_with(db, [{":db/id": 1,
                                "name": "Ivan",
                                "age": 17},
                                {":db/id": 2,
                                "name": "Igor",
                                "age": 35}])
```
You'll notice that datascript takes a database (`db`) and adds data to it and returns a new database `db1`. This is because databases in DataScript are immutable. Once created, you can't change it. This makes debugging and reasoning about the code using a database easier. I've written a bit about the general concept of immutability [here](https://blog.rajivabraham.com/posts/pyrsistent)

For each dictionary, DataScript will add two rows(one for name and another for age) in the single god table. The way DataScript keeps track is with the entity id i.e. `:db/id` attribute name which is a reserved attribute name in DataScript.

So this may be represented in the god table like:

| entity_id     | attribute  | value  |
| ------------- |:----------:| ------:|
| 1             | name       | Ivan   |
| 1             | age        | 17     |
| 2             | name       | Igor   |
| 2             | age        | 35     |

Finally, the query. The query follows the Clojure style of a mixed list. If we want to know the age of the entity whose name is `Igor`,
the query would look like `[:find ?a :where [?e "name" "Igor"] [?e "age" ?a]]`. The variables with a question mark are called logic variables. If you are familiar with inner joins in SQL, it's very similar. We are saying that if there is some `:db/id` (i.e. `?e`) whose name is `Igor`, for that value of `?e`(i.e `2`) find a relation for `age` (i.e. `[?e "age" ?a]`) and return his age(i.e `?a`). We should get back `35`. The query and the call to the database is: 

```
result = datascript.q('[:find ?a :where [?e "name" "Igor"] [?e "age" ?a]]', db1)
```

The full example is below:
```
<!doctype html>
<html>

<head>
    <meta charset="utf-8">
    <script type="text/javascript" src="brython.js"></script>
    <script type="text/javascript" src="brython_stdlib.js"></script>
    <script src="https://github.com/tonsky/datascript/releases/download/0.18.10/datascript-0.18.10.min.js"></script>
</head>

<body onload="brython(1)">
<script type="text/python">
from browser import window, alert
datascript = window.datascript

db = datascript.empty_db()
db1 = datascript.db_with(db, [{":db/id": 1,
                                "name": "Ivan",
                                "age": 17},
                                {":db/id": 2,
                                "name": "Igor",
                                "age": 35}])
result = datascript.q('[:find ?a :where [?e "name" "Igor"] [?e "age" ?a]]', db1)
alert(result)
</script>
</body>

</html>
``` 

Other notes for the above example:
 * We load the `Brython` libraries: `brython.js` and `brython_stdlib.js` to run Python in the browser. 
 * We load the `datascript` library from GitHub using the `script` tag . Brython will automatically create a reference for `datascript` under the `window` object. So we can refer to the module using `window.datascript`
 * When you open [igor-just-datascript.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/igor-just-datascript.html) in a browser, it should return [[35]]. The return value is always a list.

## Mercylog Datascript
Now you could happily continue and query the database with strings like `[:find ?a :where [?e "name" "Igor"] [?e "age" ?a]]`. Personally, I find writing code as strings works for simple queries. But it's not easily composable and reusable. So I wrote a Brython library called [mercylog-datascript](https://github.com/RAbraham/mercylog-datascript) which allows us to use Python to construct the queries. So the same query in `mercylog-datascript` becomes


```
# The original datascript style as str_query
str_query = '[:find ?a :where [?e "name" "Igor"] [?e "age" ?a]]'

# mercylog-datascript style
from mercylog_datascript import DataScriptV1
m = DataScriptV1()
A, E = m.variables('a', 'e')
query = m.query(find=[A], where=[[E, "name", "Igor"], [E, "age", A]])
assert str_query == query.code()
```

Granted, it's more code than a simple string but I think when the queries become complex and begin to have reusable parts, this style may start paying off.

All you need to access the `mercylog-datascript` library is add the following script tag:

```
<script src="https://github.com/RAbraham/mercylog-datascript/releases/download/v0.1.4/mercylog_datascript.brython.js"></script>`
``` 
The full script is at [igor-mercylog-datascript.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/igor-mercylog-datascript.html)


### Mercylog-DataScript Queries
Now that you have seen an example, let me show you the current query feature set with a sample dataset of actors, directors and movies. The full dataset is [here](https://github.com/RAbraham/mercylog-datascript-client/blob/master/data.json). Here is an elided subset.

```
[
 {":db/id":  100,
  ":person/name": "James Cameron",
  ":person/born": "1954-08-16"},

 {":db/id":  131,
  ":person/name": "Charles Napier",
  ":person/born": "1936-04-12",
  ":person/death": "2011-10-05"},
  ....

  {":db/id":  200,
  ":movie/title": "The Terminator",
  ":movie/year":  1984,
  ":movie/director":  100,
  ":movie/cast":  [101,
               102,
               103],
  ":movie/sequel":  [207]},

 {":db/id":  201,
  ":movie/title": "First Blood",
  ":movie/year":  1982,
  ":movie/director":  104,
  ":movie/cast":  [105,
               106,
               107],
  ":movie/sequel":  [209]},
....

]
```

Here is the code to access the raw dataset and a simple pattern access as above with the Igor example. The code is [here](https://github.com/RAbraham/mercylog-datascript-client/blob/master/simple_data_pattern.html). Below we want to know the birthdate of `Linda Hamilton` and it will return `[['1956-09-26']]`

```
<!doctype html>
<html>

<head>
    <meta charset="utf-8">
    <script type="text/javascript" src="brython.js"></script>
    <script type="text/javascript" src="brython_stdlib.js"></script>
    <script src="https://github.com/RAbraham/mercylog-datascript/releases/download/v0.1.4/mercylog_datascript.brython.js"></script>

    <script src="https://github.com/tonsky/datascript/releases/download/0.18.10/datascript-0.18.10.min.js"></script>
</head>

<body onload="brython(1)">
<script type="text/python">
from browser import window, alert, console
datascript = window.datascript

from mercylog_datascript import DataScriptV1
m = DataScriptV1()
import urllib.request, json
data_file_url = 'https://raw.githubusercontent.com/RAbraham/mercylog-datascript-client/master/data.json'
console.log('Loading File')
with urllib.request.urlopen(data_file_url) as url:
    result = json.loads(url.read())
console.log('End loading file')

db = datascript.empty_db()
db2 = datascript.db_with(db, result)

e, name, born = m.variables('e', 'name', 'born')

query = m.query(find=[born], where=[[e, ":person/name", "Linda Hamilton"], [e, ":person/born", born]])
q = query.code()
result = datascript.q(q, db2)
alert(result)  # [['1956-09-26']]


</script>
</body>

</html>
```

Henceforth, I'll only focus on the `mercylog-datascript` query builder and how it supports DataScript.

Let's start with something simple. How do we just get the id(i.e `:db/id`) of a person? As shown in [entity.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/entity.html), the following query returns `[[102]]` for  `Linda Hamilton`.

```
e = m.variables('e')
query = m.query(find=[e], where=[[e, ":person/name", "Linda Hamilton"]])
```

That's great, but sometimes you want to create a parameterized query i.e. a query that can be used with different values. Let's generalize the query above and then we can use it for different values. We do this by adding a `parameters` key to our `query` function.

In [parameterized_queries.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/parameterized_queries.html), you'll see:
```
# [:find ?e :in $ ?name :where [?e ":person/name" ?name]]
e, name = m.variables('e', 'name')
query = m.query(find=[e], parameters=[name], where=[[e, ":person/name", name]])
q = query.code()
result1 = datascript.q(q, db2, 'Linda Hamilton')
result2 = datascript.q(q, db2, 'Sylvester Stallone')
```
As you can see above, the same query can be used to query `Linda Hamilton`  and `Sylvester Stallone`.

Sometimes, you don't care about the entity, when doing a search. For e.g, you just want all the movie titles. In that case, `mercylog-datascript` provides the `m._` variable.

In [underscore.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/underscore.html), you'll find:
```
# [:find ?title :where [_ ":movie/title" ?title]]
title = m.variables('title')
query = m.query(find=[title], where=[[m._, ":movie/title", title]])
```
This would return 
```
[['First Blood'], ['Terminator 2: Judgment Day'], ....  ['Terminator 3: Rise of the Machines']]
```
In [attr.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/attr.html), we show how to find out the attributes which are commonly associated with a particular attribute(i.e. ":person/name"). To reiterate, as DataScript does not have a fixed schema, it's possible for a customer record c1 to only have a name but another record c2 to have a name and age.

```
# DataScript Query: [:find ?attribute :where [?person ":person/name"] [?person ?attribute]]
person, attribute = m.variables('person', 'attribute')
query = m.query(find=[attribute],
                where=[[person, ":person/name"],
                       [person, attribute]])

q = query.code()
result = datascript.q(q, db2) # [[':person/born'], [':person/name'], [':person/death']]
```
Above, we find out that an entity which has an attribute `:person/name` can also have one or more of `[[':person/born'], [':person/name'], [':person/death']]`. The slow reader who didn't try to read this in the elevator may have noticed that previously our list in the `where` clause were of size three but when we do such kind of meta searches, we can just pass lists of size 2(e.g. `[person, attribute]`)

Now, you sigh and say, this is all good but no language is a language unless it allows you to create functions. In [transformation.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/transformation.html), we see an example of passing a user defined function(`get_age`) to be executed against the database.
In the code snippet below, notice `[get_age(born), age]`. You could see this as `age = get_age(born)` for different values of `born` and you could reuse that in your query. In our case, we just ask for it directly in `find`.  
 Since it's user defined, we add `get_age` to the `parameters` argument for `query` as well and pass `get_age.function` to the DataScript query engine as well.


```
# DataScript Query:[:find ?age :in $ ?get_age ?name :where [?p ":person/name" ?name] [?p ":person/born" ?born] [(?get_age ?born) ?age]]
from datetime import datetime
current_year = int(datetime.today().strftime('%Y'))
get_age = m.function('get_age', lambda born: (current_year - int(born.split('-')[0])) )

born, p, age, name = m.variables('born', 'p', 'age', 'name')
query = m.query(find=[name, age],
                parameters=[get_age, name],
                where=[[p, ":person/name", name],
                       [p, ":person/born", born],
                       [get_age(born), age]])

q = query.code()
result = datascript.q(q, db2, get_age.function, "Richard Crenna")
alert(result) #[['Richard Crenna', 94]]


```

Ok, you say this is simple stuff, yawn, what about SQL like aggregate functions? DataScript has you covered. In [aggregates.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/aggregates.html), we show the use of inbuilt functions of DataScript i.e. `agg.max(age)`. As a bonus, we now show how to combine that with a user defined function(`get_age`). 

```
# DataScript Query:[:find (max ?age) :in $ ?get_age :where [?p ":person/name" ?name] [?p ":person/born" ?born] [(?get_age ?born) ?age]]
from datetime import datetime
current_year = int(datetime.today().strftime('%Y'))
get_age = m.function('get_age', lambda born: (current_year - int(born.split('-')[0])) )
agg = m.agg
born, p, age, name = m.variables('born', 'p', 'age', 'name')
query = m.query(find=[agg.max(age)],
                parameters=[get_age],
                where=[[p, ":person/name", name],
                       [p, ":person/born", born],
                       [get_age(born), age]])

q = query.code()
result = datascript.q(q, db2, get_age.function)
alert(result) # [[94]]

```

What about SQL where like clauses? We have filters in DataScript too. It's basically a user defined function in it's own row. For e.g, in [predicate2.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/predicate2.html)(because getting `predicate1.html` to work made me cry), I'm going to make a simple filter function called `born_before_1950` to filter out the wiser actors.

```
# DataScript Query: [:find ?person :in $ ?born_before_1950 :where [?person ":person/born" ?birth_date] [(?born_before_1950 ?birth_date)]]
person, birth_date = m.variables('person', 'birth_date')
born_func = lambda x: int(x.split('-')[0]) < 1950
born_before_1950 = m.function('born_before_1950', born_func)
query = m.query(find=[person],
                parameters=[born_before_1950],
                where=[[person, ":person/born", birth_date],
                       [born_before_1950(birth_date)]])
q = query.code()
result = datascript.q(q, db2, born_before_1950.function)
alert(result) # [[148], [119], [146], [105], [136], [116], [135], [145], [111], [114], [118], [115], [147], [104], [139], [110], [133], [131], [140], [101], [123], [142], [137], [138], [106], [107], [113], [124], [130]]
 
``` 


We can leverage ClojureScript inbuilt functions too. As seen in [inbuilt_function3.html](https://github.com/RAbraham/mercylog-datascript-client/blob/master/inbuilt_function3.html)(why `3`?, you guessed it ;)), we can call inbuilt functions like `<`. You still leverage the `m.function` method but since you don't pass your own user defined function, `mercylog-datascript` will assume that you are trying to call an inbuilt function. You can call that convenient or over smart, only time will tell.  

```
# DataScript Query: [:find ?title :where [?movie ":movie/title" ?title] [?movie ":movie/year" ?year] [(< ?year 1984)]]
title, movie, year = m.variables('title', 'movie', 'year')
lt = m.function("<")
query = m.query(find=[title],
                where=[[movie, ":movie/title", title],
                       [movie, ":movie/year", year],
                       [lt(year, 1984)]])

q = query.code()
alert(q)
result = datascript.q(q, db2)
alert(result) # [['First Blood'], ['Alien'], ['Mad Max'], ['Mad Max 2']

```
But most of the fun happens when we link different sources of data together to make money. For e.g. you may call a service which gives you the box office numbers for popular movies and you want to link that with the data in your database and find out the corresponding directors. 

 This is similar to an inner join in SQL between two tables but in this case, one side is a 'table' in DataScript and the other side is your in-memory structure which you obtain after calling the API for the service. Suppose you store the result of the API call in an in-memory list(`title_box_office_pairs` below). You need to tell DataScript about it's structure. So you specify it in the `parameters` argument as `m.collection([title, box_office])` and then later pass `title_box_office_pairs` to `datascript.q()` later.  Then you could use the logic variable `title` and use that to link to the data in the DataScript store. Code snippet below and full code [here](https://github.com/RAbraham/mercylog-datascript-client/blob/master/parameterized_queries_relations.html), 

```
# DataScript query: [:find ?director ?box_office :in $ [[?title ?box_office]] :where [?p ":person/name" ?director] [?movie ":movie/director" ?p] [?movie ":movie/title" ?title]]

movie, p, title, box_office, director = m.variables('movie', 'p', 'title', 'box_office', 'director')
# title_box_office_pairs below could have been obtained from some api call.
title_box_office_pairs = [
 ["Die Hard", 140700000],
 ["Alien", 104931801],
 ["Lethal Weapon", 120207127],
 ["Commando", 57491000],
]
query = m.query(find=[director,
                      box_office],
                parameters=[m.collection([title, box_office])],
                where=[[p, ":person/name", director],
                       [movie, ":movie/director", p],
                       [movie, ":movie/title", title]])

q = query.code()
result = datascript.q(q, db2, title_box_office_pairs)
alert(result) # [['Richard Donner', 120207127], ['Mark L. Lester', 57491000], ['John McTiernan', 140700000], ['Ridley Scott', 104931801]]

```

Finally, we can define rules. Let's say we want to make a rule: Two people are mates if their names match(hey, that's a good reason to be mates, no?). In Datalog, one would write it like
```
Mate(E1, E2) <= Name(N, E1), Name(N, E2)
```
i.e. any person `E1` is a mate of any person `E2` if they both have the name `N`. Again `N` here is a logic variable which could represent all the names in the database.
In mercylog-datascript, you would write the above as:
```
e1, e2, n = m.variables('e1', 'e2', 'n')

mate = m.relation('mate')
r = m.rule(mate(e1, e2), [[e1, "name", n],
                          [e2, "name", n]])
```

A complete code listing is below. We also show to add a `m.function` to choose those rows where `e1` has a bigger id than `e2`. I know! It does not make sense but I've been labouring on this post for three weeks and it's time to wrap up :P:

NOTE: There is some boilerplate code for now to send the rule as a parameter to the query:`rule_code = '[' + r.code() + ']'`. I'll look into improving it if a thousand of you star this project on Github(Yeah, ain't going to happen, I know :)) 
```
from mercylog_datascript import DataScriptV1
m = DataScriptV1()
db = datascript.empty_db()

db2 = datascript.db_with(datascript.empty_db({"age": {":db/index": True}}),
                 [{ ":db/id": 1, "name": "Ivan", "age": 15 },
                  { ":db/id": 2, "name": "Petr", "age": 37 },
                  { ":db/id": 3, "name": "Ivan", "age": 37 }]);

e1, e2, p, title, n = m.variables('e1', 'e2', 'p', 'title', 'n')
mate = m.relation('mate')
gt = m.function('<')
query = m.query(find=[e1, e2],
                where=[mate(e1, e2),
                       [gt(e1, e2)]])
q = query.code()
alert(q)

r = m.rule(mate(e1, e2), [[e1, "name", n],
                          [e2, "name", n]])
rule_code = '[' + r.code() + ']'
alert(rule_code)
result = datascript.q(q, db2, rule_code)

alert(result) # [[1,3]]

```
That's about it. If you came so far, I love you. I really do. Call me.

### If you loved the above. 
* DataScript has two more APIs in addition to the Datalog API:
    * Entity API: I think this is similar in concept to graph database like query languages. I leave it to the reader to investigate
    * Pull API: Similar to GraphQL
* If you want to learn about more about the DataScript syntax, check out this [site](http://www.learndatalogtoday.org/)
* The Datomic Data Model which DataScript is inspired about is mentioned [here](https://docs.datomic.com/cloud/whatis/data-model.html)

