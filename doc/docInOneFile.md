# All in one documentaion

## About

LessQL is heavily inspired by [NotORM](https://www.notorm.com/) which presents a novel, intuitive API to SQL databases. Combined with an efficient implementation, its concepts are very unique to all database layers out there, whether they are ORMs, DBALs or something else.

In contrast to ORM, you work directly with tables and rows. This has several advantages:

- **Write less:** No glue code/objects required. Just work with your database directly.
- **Transparency:** It is always obvious that your application code works with a 1:1 representation of your database.
- **Relational power:** Leverage the relational features of SQL databases, don't hide them using an inadequate abstraction.

For more in-depth opinion why ORM is not always desirable, see:

- http://www.yegor256.com/2014/12/01/orm-offensive-anti-pattern.html
- http://seldo.com/weblog/2011/08/11/orm_is_an_antipattern
- http://en.wikipedia.org/wiki/Object-relational_impedance_mismatch

---

NotORM introduced a unique and efficient solution to database abstraction. However, it does have a few weaknesses:

- The API is not always intuitive: `$result->name` is different from `$result->name()` and more.
- There is no difference between One-To-Many and Many-To-One associations in the API (LessQL uses the List suffix for that).
- There is no advanced save operation for nested structures.
- Defining your database structure is hard (involves sub-classing).
- The source code is very hard to read and understand.

LessQL addresses all of these issues.

### API

This section covers the public, user-relevant API. There are more methods mainly used for communication between LessQL components. You can always view the source, it is very readable and quite short.

### Setup

Creating a database:

```php
$db = new \LessQL\Database($pdo);
```

Defining schema information (see [Conventions](conventions.md) for usage):

```php
$db->setAlias($alias, $table);
$db->setPrimary($table, $column);
$db->setReference($table, $name, $column);
$db->setBackReference($table, $name, $column);
$db->setRequired($table, $column);
$db->setRewrite($rewriteFunc);
$db->setIdentifierDelimiter($delimiter); // default is ` (backtick)
```

Set a query callback (e.g. for logging):

```php
$db->setQueryCallback(function($query, $params) { ... });
```

### Basic finding

```php
$result = $db->table_name()
$result = $db->table('table_name')
$row = $result->fetch()      // fetch next row in result
$rows = $result->fetchAll()  // fetch all rows
foreach ($result as $row) { ... }
json_encode($result)       // finds and encodes all rows (requires PHP >= 5.4.0)

// get a row directly by primary key
$row = $db->table_name($id)
$row = $db->table('table_name', $id)
```

### Deep finding Association traversal

```php
$assoc = $result->table_name()       // get one row, reference
$assoc = $result->table_nameList()   // get many rows, back reference
$assoc = $result->referenced('table_name')
$assoc = $result->referenced('table_nameList')

$assoc = $row->table_name()          // get one row, reference
$assoc = $row->table_nameList()      // get many rows, back reference
$assoc = $row->referenced('table_name')
$assoc = $row->referenced('table_nameList')

$assoc = $row->table_name()->via($key); // use alternate foreign key
```

### Where

`WHERE` may also be applied to association results.

```php
$result2 = $result->where($column, null)    // WHERE $column IS NULL
$result2 = $result->where($column, $value)  // WHERE $column = $value (escaped)
$result2 = $result->where($column, $array)  // WHERE $column IN $array (escaped)
    // $array containing null is respected with OR $column IS NULL

$result2 = $result->whereNot($column, null)    // WHERE $column IS NOT NULL
$result2 = $result->whereNot($column, $value)  // WHERE $column != $value (escaped)
$result2 = $result->whereNot($column, $array)  // WHERE $column NOT IN $array (escaped)
    // $array containing null is respected with AND $column IS NOT NULL

$result2 = $result->where($whereString, $param1, $param2, ...) // numeric params for PDO
$result2 = $result->where($whereString, $paramArray)           // named and/or numeric params for PDO

$result2 = $result->where($array)    // for each key-value pair, call $result->where($key, $value)
$result2 = $result->whereNot($array) // for each key-value pair, call $result->whereNot($key, $value)
```

### Selected columns, Order and Limit

Note that you can order association results, but you cannot use `LIMIT` on them.

```php
$result2 = $result->select($expr)  // identfiers NOT escaped, so expressions are possible
    // multiple calls are joined with a comma

// $column will be escaped
$result2 = $result->orderBy($column);
$result2 = $result->orderBy($column, 'ASC');
$result2 = $result->orderBy($column, 'DESC');

$result2 = $result->limit($count);
$result2 = $result->limit($count, $offset);
$result2 = $result->paged($pageSize, $page);  // pages start at 1
```

Note that `Result` objects are immutable. All filter methods like where or orderBy return a new Result instance with the new `SELECT` information.

### Aggregation

Aggregation is only supported by basic results. The methods execute the query and return the calculated value directly.

```php
$result->count($expr = '*')   // SELECT COUNT($expr) FROM ...
$result->min($expr)           // SELECT MIN($expr)   FROM ...
$result->max($expr)           // SELECT MAX($expr)   FROM ...
$result->sum($expr)           // SELECT SUM($expr)   FROM ...
$result->aggregate($expr)     // SELECT $expr          FROM ...
```

### Manipulation

```php
$statement = $result->insert($row)   // $row is a data array

// $rows is array of data arrays
// one INSERT per row, slow for many rows
// supports Literals, works everywhere
$statement = $result->insert($rows)

// use prepared PDO statement
// does not support Literals (PDO limitation)
$statement = $result->insert($rows, 'prepared')

// one query with multiple value lists
// supports Literals, but not supported in all PDO drivers (SQLite fails)
$statement = $result->insert($rows, 'batch')

$statement = $result->update($set)   // updates rows matched by the result (UPDATE ... WHERE ...)
$statement = $result->delete()         // deletes rows matched by the result (DELETE ... WHERE ...)
```

### Transactions

```php
$db->begin()
$db->commit()
$db->rollback()
```

### Rows

```php
// create row from scratch
$row = $db->createRow($table, $properties = [])
$row = $db->table_name()->createRow($properties = [])

// get or set properties
$row->property
$row->property = $value
isset($row->property)
unset($row->property)

// array access is equivalent to property access
$row['property']
$row['property'] = $value
isset($row['property'])
unset($row['property'])

$row->setData($array) // sets data on row, extending it

// manipulation
$row->isClean()       // returns true if in sync with database
$row->exists()        // returns true if the row exists in the database
$row->save()          // inserts if not in database, updates changes (only) otherwise
$row->update($data) // set data and save
$row->delete()

// references
$assoc = $row->table_name()         // get one row, reference
$assoc = $row->table_nameList()     // get many rows, back reference
$assoc = $row->referenced('table_name')
$assoc = $row->referenced('table_nameList')

json_encode($row)
foreach ($row as $name => $value) { ... }  // iterate over properties
```

## Conventions (and workarounds)

LessQL relies entirely on two conventions:

- Primary key columns should be `id` and
- foreign key columns should be `<table>_id`.

A side effect of these conventions is to use **singular table names**, because plurals are irregular and product.categories_id sounds wrong.

More often than not, these conventions are not enough to work with your database, and workarounds are needed. You will most likely have join tables with compound primary keys. Or you might have two columns in one table pointing to the same foreign table.

LessQL provides solutions for all of these use cases. This section contains real-world examples showing how to define alternate primary keys, assocations and reference keys.

Because SQL is case-insensitive, you should always use `snake_case` for table names, column names, and aliases.

### Required columns

When saving complex structures, LessQL needs a way to know which columns and especially foreign keys are required (`NOT NULL`). This way, rows with required columns are saved last with all their foreign keys set.

Whenever you get "some column may not be null" exceptions, this should solve that.

```php
$db->setRequired('post', 'user_id');
```

### Alternate foreign keys

You can use alternate foreign keys in associations using `via`:

```php
foreach ($db->post() as $post) {
    // single: use post.author_id instead of post.user_id
    $author = $post->user()->via('author_id')->fetch();

    // list: use category.featured_post_id instead of category.post_id
    $featureCategories = $post->categoryList()->via('featured_post_id');
}
```

This is quick and easy, but repetitive. It is often better to define these things globally using one of the following methods.

### Table Aliasing

Let `customer` have columns `address_id` and `billing_address_id`. Both columns point to the `address` table, but the association `billing_address` would try to query the `billing_address` table. For this situation, LessQL lets you define **table aliases**:

```php
$db->setAlias('billing_address', 'address');
$db->customer()->billing_address();
```

This association will use `customer.billing_address_id` as reference key and address as table. Note how we are using the association name, *not* the table name.

### Back References

Consider again the use case from above. Let's try to access the data the other way around: Starting with an address, how would you get users that point to it via `billing_address_id`? The solution is using **back references** and aliases and looks like this:

```php
$db->address()->userList(); // works by convention

$db->setAlias('user_billing', 'user');
$db->setBackReference('address', 'user_billing', 'billing_address_id');
$db->address()->user_billingList();
```

Setting a back reference key in this case states: `address` is referenced by `user_billing` using `billing_address_id`.

### Alternate Primary Keys

Primary keys should be named `id`, but if a primary key goes by a different column name you can override it easily with `$db->setPrimary('user', 'uid')`.

Compound primary keys are also possible and especially useful for join tables: `$db->setPrimary('categorization', ['post_id', 'category_id'])`.

You should always define compound keys manually. Without these definitions, saving nested structures may fail because LessQL cannot know which columns are required for a complete primary key.

### Alternate Reference Keys

Foreign keys should be named `<table>_id`, but if a foreign key goes by a different column name you can override it easily with `$db->setReference('post', 'user', 'uid')`, which basically states the following: post references `user` using the column `uid`.

We're using the term reference to distinguish from back references, as both use foreign keys.

### Table prefixes and other custom table schemes

To add a prefix to your tables, you can define a table rewrite function:

```php
$db->setRewrite(function($table) {
    return 'prefix_' . $table;
});
```

The function is completely arbitrary and will rewrite tables directly before executing any query.

## Guide

### Installation

Install LessQL via Composer, the package name is `morris/lessql`:

```json
{
    "require": {
        "morris/lessql": "~0.3"
    }
}
```

You can also download an archive from the GitHub repository.

LessQL requires PHP >= 5.3.0 and PDO.

### Database

LessQL works on existing MySQL, PostgreSQL or SQLite3 databases. There's no schema generation or migration in LessQL, use a dedicated tool like [Phinx](https://phinx.org/) for that.

The following tables are used throughout this guide:

```
user:           id, name
post:           id, title, body, date_published, is_published, user_id
categorization: category_id, post_id
category:       id, title
```

### Setup

Create a `PDO` instance and a `LessQL\Database` using it. We will also need a few hints about our database so we define them at setup.

```php
$pdo = new \PDO('sqlite:blog.sqlite3');
$db = new \LessQL\Database($pdo);

$db->setAlias('author', 'user');
$db->setPrimary('categorization', array('category_id', 'post_id'));
```

We define `author` to be a table alias for `user` and a compound primary key for the `categorization` table. See the [Conventions](conventions.md) section for more information about schema hints.

### Finding and Traversal

The most interesting feature of LessQL is easy and performant traversal of associated tables. Here, we're iterating over four tables in an intuitive way, and the data is retrieved efficiently under the hood.

```php
foreach ($db->post()
    ->orderBy('date_published', 'DESC')
    ->where('is_published', 1)
    ->paged(10, 1) as $post) {
    // Get author of post
    // Uses the pre-defined alias, gets from user where id is post.author_id
    $author = $post->author()->fetch();

    // Get category titles of post
    $categories = array();

    foreach ($post->categorizationList()->category() as $category) {
        $categories[] = $category['title'];
    }

    // render post
    $app->renderPost($post, $author, $categories);
}
```

LessQL creates only four queries to execute this example:

```sql
SELECT * FROM `post` WHERE `is_published` = 1 ORDER BY `published` DESC LIMIT 10 OFFSET 0
SELECT * FROM `user` WHERE `id` IN (...)
SELECT * FROM `categorization` WHERE `post_id` IN (...)
SELECT * FROM `category` WHERE `id` IN (...)
```

When traversing associations, LessQL always eagerly loads all references in one query. This way, the number of queries is always constant, no matter how "deep" you are traversing your database.

Let's step through the example in some detail. The first part iterates over a subset of posts:

```php
foreach ($db->post()
    ->orderBy('date_published', 'DESC')
    ->where('is_published', 1)
    ->paged(10, 1) as $post) { /* ... */ }
```

The `orderBy` and `where` calls are basic SQL, `paged(10, 1)` limits to page 1 where pages have a size of 10 posts.

Note that `Result` objects are immutable. All filter methods like `where` or `orderBy` return a new `Result` instance with the new `SELECT` information.

Inside the loop, we have access to `$post`. It is a `Row` instance which can be worked with like an associative array or object. It can be modified, saved, deleted, and you can retrieve associated rows.

### Many-To-One

```php
// Get author of post
$author = $post->author()->fetch();
```

A post has one author, a Many-To-One-Association. LessQL will look for `author_id` in the post table and find the corresponding author, if any.

Note the explicit `fetch()` to get the row. This is required because you might want to get the author using other methods (`via`).

### One-To-Many (and Many-To-Many, too)

```php
// Get category titles of post
$categories = array();

foreach ($post->categorizationList()->category() as $category) {
    $categories[] = $category['title'];
}
```

A post has many categorizations, a One-To-Many-Association. The `List` suffix in `$post->categorizationList()` tells LessQL to look for `post_id` in the `categorization` table and find all rows that point to our post.

In turn, the categorization table points to the category table. This way we model a Many-To-Many-Association between posts and categories.

Note how we directly call `->category()` without intermediate rows.

### Saving

LessQL is capable of saving deeply nested structures with a single method call.

```php
$row = $db->createRow('post', [
    'title' => 'Fantasy Movie Review',
    'author' => [
        'name' => 'Fantasy Guy'
    ],
    'categorizationList' => [
        [
            'category' => ['title' => 'Movies']
        ],
        [
            'category' => $existingFantasyCategory
        ]
    ]
]);

// wrapping this in a transaction is a good practice and more performant
$db->begin();
$row->save();
$db->commit();
```

LessQL generates all queries needed to save the structure, including references:

```sql
INSERT INTO `post` (`title`, `author_id`) VALUES ('Fantasy Movie Review', NULL)
INSERT INTO `user` (`name`) VALUES ('Fantasy Guy')
UPDATE `post` SET `user_id` = ... WHERE `id` = ...
INSERT INTO `category` (`title`) VALUES ('Movies')
INSERT INTO `categorization` (`post_id`, `category_id`) VALUES (...)
INSERT INTO `categorization` (`post_id`, `category_id`) VALUES (...)
```

For this operation to work, two things are crucial: First, `author_id` must be nullable. Second, LessQL must know about the compound primary key of the `categorization` table.

Always define required columns and compound primary keys at setup. See [Conventions](conventions.md) for more details.
