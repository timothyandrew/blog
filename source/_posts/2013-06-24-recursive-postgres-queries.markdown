---
layout: post
title: "Recursive Postgres Queries"
date: 2013-06-24 08:30
comments: true
categories: [database, postgres, recursion]
---

### Introduction

At [Nilenso](http://nilenso.com/), I'm working on an ([open-source!](http://github.com/nilenso/ashoka-survey-web)) app to design and conduct surveys.

Here's a simple survey being designed:

[![Survey Builder Overview](/images/recursive-pg/survey-overview.png)](/images/recursive-pg/survey-overview.png)

Internally, this is represented as:

[![Survey Model Overview](/images/recursive-pg/internal-survey-overview.png)](/images/recursive-pg/internal-survey-overview.png)

A survey is made up of many *questions*. A number of questions can (optionally) be grouped together in a *category*. Our actual data model is a bit more complicated than this (sub-questions, especially), but let's assume that we're just working with questions and categories.

Here's how we preserve the ordering of questions and categories in this survey.

Each question and category has an `order_number` field. It's an integer that specifies the relative ordering of itself and its siblings.

For example, in this case,

[![Order Number Overview](/images/recursive-pg/order-number-overview.png)](/images/recursive-pg/order-number-overview.png)


the question `Bar` will have an `order_number` that is *less than* the order number of `Baz`.

This guarantees that the a category can fetch its sub-questions in the right order:

```ruby
# In category.rb

def sub_questions_in_order
  questions.order('order_number')
end
```
In fact, this is how we fetch the entire survey at this point. Each category calls `sub_questions_in_order` on each of its children, and so on down the entire tree.

This gives us a depth-first ordering of this tree:

[![Naive Ordering](/images/recursive-pg/naive-order.png)](/images/recursive-pg/naive-order.png)

For large surveys with 5+ levels of nesting, and more than a hundred questions, this is pretty slow.

<!-- more -->

### Recursive Queries

I've used gems like [`awesome_nested_set`](https://github.com/collectiveidea/awesome_nested_set) before, but as far as I could find, none of them supported fetching results across multiple models.

Then I stumbled on [a page](http://www.postgresql.org/docs/9.1/static/queries-with.html) describing PostgreSQL's support for recursive queries! That seemed perfect. 

Let's solve this particular problem using recursive queries. (My understanding of this is still very basic, so please don't take my word for any of this)

To define a recursive Postgres query, we need to define an initial query, which is called the non-recursive term.

In our case, that would be the top level questions and categories.
Top level elements don't have a parent category, so their `category_id` is `NULL`.

```sql
(
  SELECT id, content, order_number, type, category_id FROM questions
  WHERE questions.survey_id = 2 AND questions.category_id IS NULL
)
UNION
(
  SELECT id, content, order_number, type, category_id FROM categories
  WHERE categories.survey_id = 2 AND categories.category_id IS NULL
)
```

(That query, and all subsequent ones, assume that the survey we're querying has id = 2)

This gives us the top-level elements.

[![Top-Level Queries](/images/recursive-pg/top-level-elements-query.png)](/images/recursive-pg/top-level-elements-query.png)


Now on to the recursive term. According to the Postgres docs:

[![Postgres Steps](/images/recursive-pg/steps.png)](/images/recursive-pg/steps.png)

Our recursive term simply has to find all the direct children of the elements fetched by the non-recursive term.


```sql
WITH RECURSIVE first_level_elements AS (
  -- Non-recursive term
  (
    (
      SELECT id, content, order_number, category_id FROM questions
      WHERE questions.survey_id = 2 AND questions.category_id IS NULL
    UNION
      SELECT id, content, order_number, category_id FROM categories
      WHERE categories.survey_id = 2 AND categories.category_id IS NULL
    )
  )
  UNION
  -- Recursive Term
  SELECT q.id, q.content, q.order_number, q.category_id
  FROM first_level_elements fle, questions q
  WHERE q.survey_id = 2 AND q.category_id = fle.id
)
SELECT * from first_level_elements;
```

But wait. The recursive term is only fetching questions. What if the child of a first-level category is a category?
Postgres doesn't let us reference the non-recursive term more than once. So doing a `UNION` of categories and questions is out.
We have to be a little creative here:

```sql
WITH RECURSIVE first_level_elements AS (
  (
    (
      SELECT id, content, order_number, category_id FROM questions
      WHERE questions.survey_id = 2 AND questions.category_id IS NULL
    UNION
      SELECT id, content, order_number, category_id FROM categories
      WHERE categories.survey_id = 2 AND categories.category_id IS NULL
    )
  )
  UNION
  (
      SELECT e.id, e.content, e.order_number, e.category_id
      FROM
      (
        -- Fetch questions AND categories
        SELECT id, content, order_number, category_id FROM questions WHERE survey_id = 2
        UNION
        SELECT id, content, order_number, category_id FROM categories WHERE survey_id = 2
      ) e, first_level_elements fle
      WHERE e.category_id = fle.id
  )
)
SELECT * from first_level_elements;
```
We perform a `UNION` of categories and questions *before* joining it with the non-recursive term.

This yields all the survey elements:

[![Query without ordering](/images/recursive-pg/all-elements-without-order-query.png)](/images/recursive-pg/all-elements-without-order-query.png)

Unfortunately, it looks like the ordering is way off.

### Ordering a Recursive Query

The problem is that since we're effectively *appending* second-level elements to first-level elements, we're effectively performing a [*breadth-first search*](https://en.wikipedia.org/wiki/Breadth-first_search) instead of a [*depth-first search*](http://en.wikipedia.org/wiki/Depth-first_search).

How can we correct that?

Postgres has [arrays](http://www.postgresql.org/docs/9.1/static/arrays.html) that can be built during a query.

Let's build an array of order numbers for each element that we fetch. Let's call this the `path`. The `path` for an element is:

> the path of it's parent category (if one exists) + its own order_number

If we sort the final result by `path`, we've converted it into a depth-first search!

```sql
WITH RECURSIVE first_level_elements AS (
  (
    (
      SELECT id, content, category_id, array[id] AS path FROM questions
      WHERE questions.survey_id = 2 AND questions.category_id IS NULL
    UNION
      SELECT id, content, category_id, array[id] AS path FROM categories
      WHERE categories.survey_id = 2 AND categories.category_id IS NULL
    )
  )
  UNION
  (
      SELECT e.id, e.content, e.category_id, (fle.path || e.id)
      FROM
      (
        SELECT id, content, category_id, order_number FROM questions WHERE survey_id = 2
        UNION
        SELECT id, content, category_id, order_number FROM categories WHERE survey_id = 2
      ) e, first_level_elements fle
      WHERE e.category_id = fle.id
  )
)
SELECT * from first_level_elements ORDER BY path;
```

[![Query with duplicate](/images/recursive-pg/almost.png)](/images/recursive-pg/almost.png)

That's *almost* right. There are two entries for *What's your favourite song?*

This is happening because when we do an ID comparison to look for children:

```sql
WHERE e.category_id = fle.id
```

`fle` contains both questions and categories. But we want this to match only categories (because questions can't have children).

Let's hard-code a type into each of these queries, so that we don't try and check for children of a question:

```sql
WITH RECURSIVE first_level_elements AS (
  (
    (
      SELECT id, content, category_id, 'questions' as type, array[id] AS path FROM questions
      WHERE questions.survey_id = 2 AND questions.category_id IS NULL
    UNION
      SELECT id, content, category_id, 'categories' as type, array[id] AS path FROM categories
      WHERE categories.survey_id = 2 AND categories.category_id IS NULL
    )
  )
  UNION
  (
      SELECT e.id, e.content, e.category_id, e.type, (fle.path || e.id)
      FROM
      (
        SELECT id, content, category_id, 'questions' as type, order_number FROM questions WHERE survey_id = 2
        UNION
        SELECT id, content, category_id, 'categories' as type, order_number FROM categories WHERE survey_id = 2
      ) e, first_level_elements fle
      -- Look for children only if the type is 'categories'
      WHERE e.category_id = fle.id AND fle.type = 'categories'
  )
)
SELECT * from first_level_elements ORDER BY path;
```
[![Final Query](/images/recursive-pg/final.png)](/images/recursive-pg/final.png)

That seems about right. We're done here.

Let's see how much of a performance boost this gives us.

### Performance

Using this script (after creating a survey from the UI), I generated 10 sub-question chains; each 6 levels deep.

```ruby
survey = Survey.find(9)
10.times do
  category = FactoryGirl.create(:category, :survey => survey)
  6.times do
    category = FactoryGirl.create(:category, :category => category, :survey => survey)
  end
  FactoryGirl.create(:single_line_question, :category_id => category.id, :survey_id => survey.id)
end
```

Each chain looks like this:

[![Sub-question Chain](/images/recursive-pg/chain.png)](/images/recursive-pg/chain.png)

Let's see if recursive queries are faster.

```ruby
pry(main)> Benchmark.ms { 5.times { Survey.find(9).sub_questions_using_recursive_queries }}
=> 36.839999999999996

pry(main)> Benchmark.ms { 5.times { Survey.find(9).sub_questions_in_order } }
=> 1145.1309999999999
```

31x faster? Not bad.

[![Not bad](/images/recursive-pg/not-bad.jpg)](/images/recursive-pg/not-bad.jpg)
