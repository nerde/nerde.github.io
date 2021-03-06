---
layout: post
title: "A Better LEFT JOIN with ActiveRecord"
category: posts
comments: true
---

Every once in a while when I'm working with Rails, I face a little
problem like this: given two models, `Book` and `Category`, book belonging to a
category, list all books ordered by its category's name, showing it
alongside with the it's name.

That's easy enough, right?

{% highlight ruby %}
ca1 = Category.create! name: 'Category A'
ca2 = Category.create! name: 'Category B'
Book.create! name: 'Book A', category: ca1
Book.create! name: 'Book B', category: ca1
Book.create! name: 'Book C', category: ca2

category_name = Category.arel_table[:name]
Book.eager_load(:category).order(category_name).each do |book|
  puts "#{book.category.name} - #{book.name}"
end
#=> Category A - Book A
#=> Category A - Book B
#=> Category B - Book C
{% endhighlight %}

The code above generates the SQL below:

{% highlight sql %}
SELECT "books"."id" AS t0_r0, "books"."name" AS t0_r1,
"books"."category_id" AS t0_r2, "books"."created_at" AS t0_r3,
"books"."updated_at" AS t0_r4, "categories"."id" AS t1_r0,
"categories"."name" AS t1_r1, "categories"."created_at" AS t1_r2,
"categories"."updated_at" AS t1_r3 FROM "books"
LEFT OUTER JOIN "categories" ON "categories"."id" = "books"."category_id"
ORDER BY "categories"."name";
{% endhighlight %}

I don't like this SQL. If I have a lot of books and categories, its
performance will not be as good as it could. Let's make it better.

{% highlight ruby %}
category_name = Category.arel_table[:name]
Book.joins(:category).preload(:category).order(category_name) \
  .each do |book|
    puts "#{book.category.name} - #{book.name}"
  end
#=> Category A - Book A
#=> Category A - Book B
#=> Category B - Book C
{% endhighlight %}

The SQL now will be:

{% highlight sql %}
SELECT "books".* FROM "books"
INNER JOIN "categories" ON "categories"."id" = "books"."category_id"
ORDER BY "categories"."name";

SELECT "categories".* FROM "categories" WHERE "categories"."id" IN (1, 2);
{% endhighlight %}

Despite the fact that two queries need to be run for this approach, it is, in
fact, better than the former. It's faster, when you have a large amount of
data.

Very well. Now, let's create a fourth book:

{% highlight ruby %}
Book.create! name: 'Book D'
category_name = Category.arel_table[:name]
Book.joins(:category).preload(:category).order(category_name) \
  .each do |book|
    puts "#{book.category.name} - #{book.name}"
  end
#=> Category A - Book A
#=> Category A - Book B
#=> Category B - Book C
{% endhighlight %}

Did you notice something wrong? The fourth book is not shown on the
list. That's because it has no category. The `INNER JOIN`
generated by ActiveRecord excludes this record from the query.

## A Known Solution

The solution here, as you might know, is to use `LEFT JOIN` instead of
`INNER JOIN`. The first algorithm shown here uses it `LEFT JOIN`, but it's
slower.

Digging on Google brought me to this solution:

{% highlight ruby %}
category_name = Category.arel_table[:name]
Book.joins('LEFT JOIN categories ON books.category_id = categories.id') \
  .preload(:category).order(category_name).each do |book|
    puts "#{book.category.try(:name)} - #{book.name}"
  end
#=> Category A - Book A
#=> Category A - Book B
#=> Category B - Book C
#=>  - Book D
{% endhighlight %}

The pros:

* It works;
* it's faster than the `eager_load`.

But it looks terrible, don't you think? The cons:

* It's not database agnostic;
* I have to explicitly define in raw SQL how these two tables are called and
how they are related to each other;
* It's not DRY, since I already defined their names and relashionship in the
model.

## Arel to the Rescue!

I found another solution:

{% highlight ruby %}
categories = Category.arel_table
books = Book.arel_table

books_categories = books.join(categories, Arel::Nodes::OuterJoin) \
  .on(books[:category_id].eq(categories[:id])).join_sources

category_name = Category.arel_table[:name]
Book.joins(books_categories).preload(:category).order(category_name) \
  .each do |book|
    puts "#{book.category.try(:name)} - #{book.name}"
  end
#=> Category A - Book A
#=> Category A - Book B
#=> Category B - Book C
#=>  - Book D
{% endhighlight %}

The pros:

* It's database agnostic;
* I do not need to know the tables' names.

The cons:

* It's too much code to do too little;
* I still have to specify how the relashionship is done.

Based on the Arel solution, I created a [gem](https://github.com/nerde/left_join)
that adds this functionality to ActiveRecord:

{% highlight ruby %}
  gem 'left_join'
{% endhighlight %}

Now, the version using the gem:

{% highlight ruby %}
category_name = Category.arel_table[:name]
Book.left_join(:category).preload(:category).order(category_name) \
  .each do |book|
    puts "#{book.category.try(:name)} - #{book.name}"
  end
#=> Category A - Book A
#=> Category A - Book B
#=> Category B - Book C
#=>  - Book D
{% endhighlight %}

Much cleaner, right? The advantages:

* It's faster than `eager_load`;
* It's database agnostic;
* I don't need to write any raw SQL;
* I don't need to repeat information about tables or relationships;
* It works like `joins` does, so it's possible to `LEFT JOIN` a chain of
associations.

That's all for now. Thanks for reading!
