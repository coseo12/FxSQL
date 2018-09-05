# MQL - Node.js 데이터베이스 쿼리 빌더

[EN](https://github.com/marpple/MQL) | [KR](https://github.com/marpple/MQL/blob/master/README_kr.md)

## Features
 - Tagged template literal
 - No models.
 - Only need functions and javascript data types.
 - Promises
 - No cost for converting to JSON.
 - More freedom in using SQL syntax.
 - Preventing SQL-injection attacks.
 - Easy to use the latest operators provided in databases.
 - Simple transaction API.
 - No models for Associations.
 - Designed to work well with PostgreSQL, MySQL.


## Overview
  - [Installation](#Installation)
  - [Connect](#Connect)
    - [PostgreSQL](#postgresql)
    - [MySQL](#mysql)
  - [Simple query](#Simple-query)
  - [Subquery, Join](#Subquery-Join)
  - [Ready to be used](#Ready-to-be-used)
  - [Helper Function](#Helper-Function)
    - [EQ](#eq)
    - [IN](#in)
    - [NOT_IN](#not_in)
    - [VALUES](#values)
    - [SET](#set)
    - [COLUMN, CL](#column-cl)
    - [TABLE, TB](#table-tb)
  - [Associations](#associations)
    - [Common use](#Common-use)
    - [Polymorphic](#polymorphic)
    - [Transaction](#transaction)
    - [Many to many](#many-to-many)
    - [Hook](#hook)
  - [Option](#Option)
  - [DEBUG](#debug)

## Installation

```
npm i mql2
```

## Connect

### PostgreSQL

```javascript
const { PostgreSQL } = require('mql2');
const { CONNECT } = PostgreSQL;
const POOL = await CONNECT({
  host: 'localhost',
  user: 'username',
  password: '1234',
  database: 'dbname'
});
```

### PostgreSQL Connection option

MQL is built on node-postgres. The parameter of CONNECT function is the same as node-postgres’. You can read the detail of [connection pool](https://node-postgres.com/api/pool) or [connecting to DB](https://node-postgres.com/features/connecting) on [node-postgres’ site](https://node-postgres.com/).

### MySQL

```javascript
const { MySQL } = require('mql2');
const { CONNECT } = MySQL;
const POOL = await CONNECT({
  host: 'localhost',
  user: 'username',
  password: '1234',
  database: 'dbname'
});
```

### MySQL Connection option

MQL is built on node-postgres. The parameter of CONNECT function is the same as the MySQL’. You can read the detail of [connection pool](https://github.com/mysqljs/mysql#pool-options) or [connecting to DB](https://github.com/mysqljs/mysql#connection-options) on [MySQL's site](https://github.com/mysqljs/mysql).

## Simple query

```javascript
const { QUERY } = POOL;
const id = 10;
const posts = await QUERY `SELECT * FROM posts WHERE id = ${id}`;
// [{ id: 10, ... }]
```

## Subquery, Join

```javascript
const type = 'TYPE1';
const limit = 10;

QUERY `
  SELECT * FROM table1 WHERE table2_id IN (
    SELECT id FROM table2 WHERE type = ${type} ORDER BY id DESC LIMIT ${limit}
  )
`;

const status = 'STATUS1';

QUERY `
  SELECT *
    FROM table1 AS t1, table2 AS t2
    WHERE t1.id = t2.table1_id AND t1.status = ${status}
    ORDER BY id DESC
    LIMIT 10
`;
```


QUERY achieved from CONNECT uses a connection pool.

## Ready to be used

```javascript
const POOL = await CONNECT();
const = {
  VALUES, IN, NOT_IN, EQ, SET, COLUMN, CL, TABLE, TB, SQL, MQL_DEBUG,
  QUERY,
  ASSOCIATE,
  LJOIN,
  TRANSACTION
} = POOL;
```

## Helper-Function

### EQ

```javascript
const users = await QUERY `SELECT * FROM users WHERE ${EQ({
  email: 'dev@marpple.com',
  password: '1234'
})}`;
// [{ id: 15, email: 'dev@marpple.com', ... }]
```

### IN

```javascript
const users = await QUERY `SELECT * FROM users WHERE ${IN('id', [15, 19, 20, 40])}`;
// [{ id: 15, ...}, { id: 19, ...} ...]
```

### NOT_IN

```javascript
const users = await QUERY `SELECT * FROM users WHERE ${NOT_IN('id', [2, 4])} LIMIT 3 ORDER BY ID`;
// [{ id: 1, ...}, { id: 3, ...}, { id: 5, ...}]
```

### VALUES

```javascript
const post = { user_id: 10, body: 'hoho' };
await QUERY `
  INSERT INTO posts ${VALUES(post)}
`;
// INSERT INTO posts ("user_id", "body") VALUES (10, 'hohoho')

await QUERY `
  INSERT INTO coords ${VALUES([
    {x: 20},
    {y: 30},
    {x: 10, y: 20}
  ])}`;
// INSERT INTO coords ("x", "y") VALUES (20, DEFAULT), (DEFAULT, 30), (10, 20)
```

### SET

```javascript
await QUERY `
  UPDATE posts ${SET({ body: 'yo!', updated_at: new Date() })} WHERE id = ${post.id}
`;
// UPDATE posts SET "body" = 'yo!', "updated_at" = '2018-08-28T23:18:13.263Z' WHERE id = 10
```

### COLUMN, CL

```javascript
COLUMN == CL; // true

await QUERY `
  SELECT
    COLUMN('id', 'bb as cc', 't2.name', 't2.name as name2', { a: 'c' }, { 't3.a': 'd' })
      ...
`;
// SELECT
//   "id", "bb" AS "cc", "t2"."name", "t2"."name" AS "name2", "a" AS "c", "t3"."a" AS "d"
//     ...
```

### TABLE, TB

```javascript
TABLE == TB; // true

await QUERY `
  SELECT
    ...
    FROM TABLE('t1'), TABLE('tt as t2')
`;
// SELECT
//   ...
//   FROM "t1", "tt" AS "t2"
```

## Associations

### Common use

ASSOCIATE uses Connection pool.

```javascript
/*
* users
*  - id
*  - name
*
* posts
*  - id
*  - user_id
*  - body

* comments
*  - id
*  - user_id
*  - post_id
*  - body
* */

const { ASSOCIATE } = POOL;

const posts = await ASSOCIATE `
  posts
    - user
    < comments
      - user
`;

posts[0].body;
posts[0]._.user.name
posts[0]._.comments[0].body
posts[0]._.comments[0]._.user.name
```

### Polymorphic

```javascript
/*
* photos
*  - attached_type
*  - attached_id
* */

await ASSOCIATE `
  posts
    - user
      p - photo
    p < photos
    < comments
      p < photos
`;
// SELECT * FROM photos WHERE attached_id IN (${map($ => $.id, posts)}) AND attached_type = 'photos';
// SELECT * FROM photos WHERE attached_id IN (${map($ => $.id, users)}) AND attached_type = 'users';
// SELECT * FROM photos WHERE attached_id IN (${map($ => $.id, comments)}) AND attached_type = 'comments';
```

### Many to many

```javascript
/*
* books
*  - id
*
* authors
*  - id
*  - name
*
* books_authors
*  - author_id
*  - book_id
* */

const books = await ASSOCIATE `
  books
    x authors
`;

books[0]._.authors[0].name;

const authors = await ASSOCIATE `
  authors
    x books ${{ xtable: 'books_authors' }}
`;

authors[0]._.books[0].name;
```

### Option

```javascript
/*
* If the tables are formed like the example below, the ASSOCIATE automatically creates the necessary table and column names for queries. the necessary names for the tables and columns for queries
* users
*  - id
* posts
*  - id
*  - user_id
* comments
*  - id
*  - post_id
*  - user_id
* likes
*  - attached_type
*  - attached_id
*  - user_id
* posts_tags
*  - post_id
*  - tag_id
* tags
*  - id
* */

ASSOCIATE `
  posts
    - user
    < comments
     - user
     p < likes
      - user
    p < likes
      - user
    x tags
`;

/*
* You can select columns or add conditions.
* Even though you don’t select a foreign key or a primary key in the option like the below, they are included in ASSOCIATE.
* */

ASSOCIATE `
  posts ${SQL `WHERE is_hidden = false ORDER BY id DESC LIMIT ${10}`}
    - user
    < comments ${{
      column: COLUMN('body', 'updated_at')
    }}
     - user
     p < likes
      - user
    p < likes
      - user
    x tags
`;


/*
* If the names of the tables and columns does not follow the ASSOCIATE rules, you need to manually insert the correct names of the tables and columns.
* members
*  - member_id
* articles
*  - id
*  - writer_id
* comments
*  - id
*  - article_id
*  - writer_id
* likes
*  - parent_name
*  - parent_id
*  - member_id
* tags_articles
*  - article_id
*  - tag_name
* tags
*  - name
* */

const posts = await ASSOCIATE `
  posts ${{
    table: 'articles' // 데이터베이스 테이블 명이 다를 때
  }}
    - user ${{ // - 를 했으므로 하나를 객체로 가져옴
      left_key: 'writer_id', // articles가 가진 members.member_id를 가리키는 컬럼
      key: 'member_id', // members 테이블이 가진 키
      table: 'members' // user의 테이블 명
    }}
    < comments ${{ // < 를 했으므로 배열로 여러개를 가져옴
      key: 'article_id' // articles의 id를 가리키는 comments가 가진 컬럼
    }}
      - user ${{
        left_key: 'writer_id', // articles가 가진 members.member_id를 가리키는 컬럼
        key: 'member_id', // members 테이블이 가진 키
        table: 'members' // user의 테이블 명
      }}
      p < likes ${{ // 하나의 likes 테이블을 통해 comments와 posts의 likes를 구현
        poly_type: { parent_name: 'comments' },
        key: 'parent_id'
      }}
    p < likes ${{ // 하나의 likes 테이블을 통해 comments와 posts의 likes를 구현
      poly_type: { parent_name: 'articles' },
      key: 'parent_id'
    }}
    x tags ${{ // x 를 통해 중간 테이블을 join 하여 다대다 관계 구현
      left_key: 'id', // articles.id (articles.id = tags_articles.article_id)
      left_xkey: 'article_id', // left_key와 매칭되는 tags_articles의 키 article_id
      xtable: 'tags_articles', // 중간 테이블 이름
      xkey: 'tag_name', // key와 매칭되는 tags_articles의 키 tag_name
      key: 'name' // tags가 가진 키 (tags_articles.tag_name = tags.name)
    }}
`;
```

If you use VIEW in databases, it's much easier. Then, you don't need to insert all correct column and table names.

### Hook

You can add virtual columns, sorting, filtering and etc by using Hook.
When all the datas are gathered below “posts”, Hook is executed.

```javascript
const users = await ASSOCIATE `
  users ${{hook: users => users.map(u =>
    Object.assign({}, u, { _popular: !!u._.posts.find(p => p._is_best) })
  )}}
    < posts ${{hook: posts => posts.map(
      p => Object.assign({}, p, { _is_best: p._.comments.length > 1 }))}}
      - user
      < comments
       - user
`;

users[0]._popular; // true
users[0]._.posts[0]._is_best; // true
users[0]._.posts[1]._is_best; // false
```

## Transaction

```javascript
const { PostgreSQL } = require('mql2');
const { CONNECT } = PostgreSQL;
const POOL = await CONNECT({
  host: 'localhost',
  user: 'username',
  password: '1234',
  database: 'dbname',
  charset: 'utf8'
});
const { TRANSACTION } = POOL;
const { QUERY, COMMIT, ROLLBACK } = await TRANSACTION();

await QUERY `
  INSERT INTO posts ${VALUES(post)}
`;
await QUERY `
  UPDATE posts ${SET({ body: 'yo!', updated_at: new Date() })} WHERE id = ${post.id}
`;
await ROLLBACK();
```


## DEBUG


```javascript
MQL_DEBUG.LOG = true;
QUERY `SELECT ${"hi~"} as ho`;

// { text: 'SELECT $1 as ho', values: ['hi'] }
```

