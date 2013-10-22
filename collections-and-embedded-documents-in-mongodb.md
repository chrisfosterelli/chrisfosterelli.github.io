Collections and Embedded Documents in MongoDB

When someone is approaching [MongoDB](http://www.mongodb.org/) from the SQL world, a very common confusion regarding database structure is when to use [embedded documents](http://docs.mongodb.org/manual/core/data-modeling-introduction/#embedded-data), and when to create an entirely new [collection](http://docs.mongodb.org/manual/reference/glossary/#term-collection). This distinction is very important because, although MongoDB is schemaless in nature, whether or not an element of your database is structured as embedded documents or a separate collection will change your code a fair amount. Making this change later on can represent a fair amount of work, so it helps to get this right the first time.
<!-- Content Breaker -->

There is no "right answer" to this question, as it depends entirely on the situation at hand. The natural tendency of people coming from the SQL world is to stick everything in separate collections, but often this is very unnecessary and will cause serious performance impacts. However, mistakenly placing something within another document may lead to pain further down the road.

A set of rules I have found useful is to ask yourself the following questions:

1. __Does the embedded document relate to one or more other collections?__
2. __Will you most often need the embedded document *without* the parent document?__
3. __Will you most often need the parent document *without* the embedded document?__

If the answer to two or more of these is _yes_, you likely will want a separate collection. If the answer to only one of these is _yes_, a separate collection should still be considered, but likely not needed. 

## Examples

#### Comments on a blog

You would like to create a system where people may submit comments on blog posts. The problem is that you are unsure if you should store the `comment` on the `post` document, or create a separate collection named `comments`.

__Does the embedded document relate to one or more other collections?__

_No_. A `comment` is typically related to only the `post` that it is commented on. There may be some situations where this is not true, such as if you provided `comment` author accounts for editing. However, even this is not a very convincing reason by itself to separate the `comment` into a separate collection.

__Will you most often need the embedded document *without* the parent document?__

Again, the answer is _no_. You likely will not often need to load a `comment` without also needing the context of the `post`.

__Will you most often need the parent document *without* the embedded document?__

In the majority of cases, the answer here is _no_. Most of the time you use this object, someone will be viewing a blog entry. You will want to both display the `post` and the comments at once, so it makes sense to fetch those together.

Overall, comments for a blog is a very good candidate for embedded documents.

#### Students in a class

You have a school management system, and you would like to enable students to enrol in a particular class. You are unsure if you should store the `student` objects on the `class`, or create a separate collection named `students`.

__Does the embedded document relate to one or more other collections?__

Typically, we can assume _yes_. A `student` will likely relate to other things, such as an `assignment` or `school` object. Also, a very important note is that each embedded document will likely relate to multiple documents in the `classes` collection, which is a very strong hint you need a separate collection.

__Will you most often need the embedded document *without* the parent document?__

The answer here will often be _yes_. If you want any sort of student information panel or want to have students enrolled in different classes, then you will often want the `student` document without needing the context of each `class`.

__Will you most often need the parent document *without* the embedded document?__

Probably _no_ for this one. It depends on what operation we are doing most often with the `class`, but I imagine that when we fetch a `class` we would likely need at least one `student` as well.

Overall, students in a class are probably better suited for a separate collection. It's important to keep in mind the scope of the problem you are solving with the data, and the operations that will be done most commonly. That said, a `student` is a very relational piece of data and better fits a separate collection.

## Other Resources

* [MongoDB: Embedded Documents vs Multiple Collections](http://openmymind.net/2012/1/30/MongoDB-Embedded-Documents-vs-Multiple-Collections/)
* [MongoDB relationships: embed or reference?](http://stackoverflow.com/questions/5373198/mongodb-relationships-embed-or-reference)
* [Data Modeling Considerations for MongoDB Applications](http://docs.mongodb.org/manual/core/data-modeling/)
