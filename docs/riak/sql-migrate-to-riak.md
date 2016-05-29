## 例子

关系数据库: 创建表

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    author VARCHAR(30) NOT NULL,
    title VARCHAR(50) NOT NULL,
    body TEXT NOT NULL,
    created DATE NOT NULL
);
```

关系数据库: 查询

```sql
SELECT * FROM posts WHERE id = 99;
```

关系数据库: 查询结果

```
 id |   author   |             title              |            body            |  created
----+------------+--------------------------------+----------------------------+------------
 99 | John Daily | Riak Development Anti-Patterns | Writing an application ... | 2014-01-07
```

基本转换和存储过程如下

- 每一个表行被转换为一个JSON对象,包含除`id`字段外的所有其他字段.
- `id`字段不会存储在Riak中的JSON对象中. 它不会作为对象的键. 而是把标题作为键, 30字符长度限制, 小写, 并且用`-`作为单词之间的连字符. 比如上面的例子把`riak-development-anti-patterns`作为键.
- 所有产生自`posts`表的JSON对象存储在名为`posts`的单个Riak桶当中.
- 键存在在Riak set中, 需要的时候, 使所有对象可以被一次查询出来.


## 把Table转换为List

下面, 我们使用`psycopg2`库来把表转换为列表

```py
import psycopg2
connection = psycopg2.connection('dbname=blog_db')
cursor = connection.cursor()
```

用该游标(cursor)执行`SELECT * FROM posts`查询,然后使用`fetchall`函数从游标当中获取信息:

```py
cursor.execute('SELECT * FROM posts')
table = cursor.fetchall()
```

`table`对象由一个Python元组列表构成, 如下:

```py
[(1, 'John Doe', 'Post 1 title', 'Post body ...', datetime.date(2014, 1, 1)),
 (2, 'Jane Doe', 'Post 2 title', 'Post body ...', datetime.date(2014, 1, 2)),
 # more posts in the list
]
```


## 转换行为JSON对象

上面的代码把`posts`表中的每一行数据转换为包含5个元素的元组, 为了更好的进行转换, 我们把元组转换为能够在Riak中更好的处理的Python字典数据类型, 官方的[Riak Python客户端](https://github.com/basho/riak-python-client)能够自动地把Python字段转换为能够在Riak中存储的JSON对象. 一旦我们有了一个字典列表, 就可以直接在Riak中存储这些字典类型的数据.

下面通过Python代码把数据库中的数据转换为JSON格式:

```py
import datetime

def convert_row_to_dict(row):
    return {
        'author': row[1],
        'title': row[2],
        'body': row[3],
        'created': row[4].strftime('%m-%d-%Y')
    }
```

产生如下的JSON数据:

```js
{
  'author': 'John Daily',
  'title': 'Riak Development Anti-Patterns',
  'body': 'Writing an application ...',
  'created': '01-07-2014'
}
```

## 存储行对象

现在使用`store_row_in_riak`把`posts`表中的数据存储到Riak中:

- 用博客的标题来构造键, 获取签名30字符, 字符全部转换为小写, 并用`-`替换空格.
- 每行将被转换为一个正确的Riak对象,并存储

下面是这个函数的实现:

```py
bucket = client.bucket('posts')

def store_row_in_riak(row):
    key = row[2][0:29].lower().replace(' ', '-') 
    obj = RiakObject(client, bucket, key)
    obj.content_type = 'application/json'
    obj.data = convert_row_to_dict(row)
    obj.store()
```

上面我们在一个[Riak set](http://docs.basho.com/riak/latest/theory/concepts/crdts/#sets)存储所有对象的键, 以帮助我们在将来能够查询这些对象. 我们修改上的`store_row_in_riak`函数添加每一个键到一个`set`(集合):

```py
from riak.datatypes import Set

objects_bucket = client.bucket('posts')
key_set = Set(client.bucket_type('sets').bucket('key_sets'), 'posts')

def store_row_in_riak(row):
    key = row[0]
    obj = RiakObject(client, bucket, key)
    obj.content_type = 'application/json'
    obj.data = convert_row_to_dict(row)
    obj.store()
```

现在,我们编写一个迭代器用于存储所有的行:

```py
# Using our "table" object from above:

for row in table:
    store_row_in_riak(row)
```

一旦所有对象存储到Riak, 我们可以执行正常的键/值操作来一个一个地获取. 下面是一个使用[Riak HTTP API](http://docs.basho.com/riak/latest/dev/references/http/)的例子:

```
curl http://localhost:8098/buckets/posts/keys/99
```

这将返回一个博客(Blog)对象:

```json
{
  "author": "John Daily",
  "title": "Riak Development Anti-Patterns",
  "body": "Writing an application ...",
  "created": "01-07-2014"
}
```

需要的时候, 我们也可以一次获取所有这些对象. 前面我们在`Raik set`中存储了所有对象的键. 我们可以编写一个函数从`set`中获取所有对象的key, 以及和这些key相关的所有对象:

```py
from riak.datatypes import Set

set_bucket = client.bucket_type('sets').bucket('key_sets')
posts_bucket = client.bucket('posts')

def fetch_all_objects(table_name):
    keys = Set(client, bucket, table_name)
    for key in keys:
        return posts_bucket.get(key)

fetch_all_objects('posts')
```

这回返回之前存储的Python字典的完整列表.

## 使用二级索引

添加一个关键字字段(`keywords`), 来描述一篇博客的关键字列表:

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    author VARCHAR(30) NOT NULL,
    title VARCHAR(50) NOT NULL,
    body TEXT NOT NULL,
    created DATE NOT NULL,
    keywords TEXT[] NOT NULL
);
```

插入数据:

```sql
INSERT INTO posts (author, title, body, created, keywords) VALUES
    ('Basho', 'Moving from MySQL to Riak', 'Traditional database architectures...',
    current_date, '{"mysql","riak","migration","rdbms"}');
```

为没一个博客的关键字列表添加一个二级索引. 下面我们编写一个函数对每一个关键字附加一个二级索引.

```py
def add_keyword_2i_to_object(obj, keywords):
    for keywork in keywords:
        obj.add_index('keywords_bin', keyword)
```

然后我们可以插入该函数到`store_row_in_riak`函数中:

```py
bucket = client.bucket('posts')

def store_row_in_riak(row):
    obj = RiakObject(client, bucket, row[0])
    obj.content_type = 'application/json'
    obj.data = convert_row_to_dict(row)
    add_keyword_2i_to_object(obj, row[5])
    obj.store()
```

现在我们可以基于一篇博客的关键字来查询:

```py
bucket = client.bucket('posts')

def fetch_posts_by_keyword(keyword):
    for key in bucket.get_index('keywords_bin', keyword):
        return bucket.get(key)
```

该函数会返回所有包含特定关键字的博客列表.