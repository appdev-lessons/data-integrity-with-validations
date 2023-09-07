# Must See Movies: Data integrity with validations

The project can be loaded here:

LTI{Load Must See Movies Validations assignment}(https://grades.firstdraft.com/launch)[S9ymPy6WCsn18gLbByVbZQ7k]{vfdtzJb5bLYqYwuqgeRKpc5d}(5)[Must See Movies Validations Project]

Our starting point code for this project, `msm-validations`, is one possible solution for `refactoring-msm-gui-1`. Therefore, the target is the same:

[https://msm-gui.matchthetarget.com/](https://msm-gui.matchthetarget.com/)

All of the specs from `refactoring-msm-gui-1` are there, and they should be passing. Just run `rake grade` once to get full credit. We won't introduce any more specs in the assignment, but you should follow along with the lesson and type commands into the `rails console` as you go.

Before you begin, read through the starting point code and compare it to your own solution to `refactoring-msm-gui-1`. Be sure to `rake sample_data` and `bin/dev` so that you can click through the application, verify that it's working, and read the server log.

Does any part of the code puzzle you?

Go ahead, I'll wait!

## Invalid records

As soon as we start saving data into our tables from external sources, be it from our users, CSVs, APIs, or wherever, we have to start worrying about whether that data is _valid_. Did they fill out all of the required fields? Did they choose a username that was already taken? Did they enter an age less than 0? Did they vote twice?

If _invalid_ records sneak into our database, and our code assumes our data is valid, then we're going to have all kinds of problems. Then, we have to start writing our code very defensively, with tons of `if`/`elsif`/`else`/`end` statements scattered everywhere to guard against invalid _data_ causing errors in otherwise functional code.

For example, in your Must See Movies Validations codespace (after `rake sample_data` and `bin/dev`), you could visit the details page of a movie; say, `/movies/2`. If you just ran `rake sample_data`, you would see the details page for The Godfather.

Now, back in your codespace, at the bash prompt start the `rails console` and enter this command:

```pry
pry(main)> Director.where({name: "Francis Ford Coppola"}).at(0).destroy
```

That will issue the SQL to delete the director of The Godfather from your database.

Back in the live app preview, visit the details page we've worked so hard on, `/movies/2`. `undefined method 'name' for nil:NilClass`?! What's up with that? It was fine a second ago!

Should you go change `app/views/movie_templates/show.html.erb` to resolve the issue? All too often, this is what students do. But you don't have a code problem; you have a _data_ problem.

Even worse, go back to `/movies`. The entire index page is broken, due to a handful of orphaned movies within the `.each` loop!

Fortunately, we have several tools at our disposal to solve the immediate problem at hand:

 - Use the debugging REPL embedded in the Better Errors page to delete the movies that are causing problems, or assign them valid `director_id`s and save.
 - Probably better: run `rake sample_data` again to reset everything.

## Defensive code

Sometimes, we really _do_ want to allow spotty data, and we just have to be prepared for it when writing our code. In this case, maybe we want to allow movies to be created without the creator knowing who the director is, and so they are allowed to leave `director_id` blank.

In that case, you have to write your code defensively, with `if`/`else`/`end` statements. For example, we could do something like this:

```erb
<% the_director = @the_movie.director %>

<% if the_director != nil %>
  <%= the_director.name %>
<% else %>
  Uh oh! We weren't able to find a director for this movie.
<% end %>
```

This will get very tiresome to repeat over and over.

## Data integrity with validations

Many times, we don't want to allow spotty data; we want our data to be uniform and consistent. In this case, it's best not to allow invalid data that doesn't meet our criteria to enter our database in the first place; then, we don't need to worry about writing lots of defensive conditionals downstream in our code.

If so, then ActiveRecord offers a tremendously useful feature: **validations**.

Validations are a way that we can specify constraints on what we allow in each column in a model. Crucially, if our validation rules aren't met, then **the .save method won't insert the record into the database**. It will _not_ transact with the database; and, it will return `false` (until now, it has always returned `true`).

Here's an example: to start to address the movies/director_id issue, let's say we don't want to allow any rows to enter our movies table with a blank `director_id` attribute.

Then, we can declare a validation in the `Movie` model with the `validates` method:

```ruby
# app/models/movie.rb

class Movie < ApplicationRecord
  validates(:director_id, presence: true)

  def director
  # ...
end
```

 - The first argument to `validates` is a `Symbol`; the name of the column we are declaring constraints for.
 - The second argument to `validates` is a `Hash`. (Recall [the lesson on different Ruby styles](https://learn.firstdraft.com/lessons/116-optional-syntaxes-in-ruby)). 
   - The keys in the `Hash` are the constraints we are applying. There is a fixed list of constraints built-in to ActiveRecord that we can choose from. By far, the three most common are `:presence`, `:uniqueness`, and `:numericality`.
 - The value associated to each key is either `true`, which means you want the simplest use of the validation; or, the value can be a whole other `Hash` containing configuration options, like minimum and maximum values for `:numericality`.
 
Let's give our simple validation a test. Open a `rails console` and create a new instance of a `Movie`:

```pry
[1] pry(main)> m = Movie.new
   (0.4ms)  SELECT sqlite_version(*)
=> #<Movie:0x00007f93ab387430
 id: nil,
 title: nil,
 year: nil,
 duration: nil,
 description: nil,
 image: nil,
 director_id: nil,
 created_at: nil,
 updated_at: nil>
```

Now, before assigning any attributes at all, try to do a `.save`:

```pry
[2] pry(main)> m.save
=> false
```

Ah ha! It returned `false`, and there's no SQL output like usual. If you look at `m`, you'll see that it still has no `id`, `created_at`, or `updated_at` — it didn't sneak into the database:

```pry
[3] pry(main)> m
=> #<Movie:0x00007f93ab387430
 id: nil,
 title: nil,
 year: nil,
 duration: nil,
 description: nil,
 image: nil,
 director_id: nil,
 created_at: nil,
 updated_at: nil>
```

### The Errors collection

If we want to, we can even find out why the object didn't save. Every ActiveRecord object has an attribute called `errors`, which contains an instance of `ActiveModel::Errors` — a collection of validation failures:

```pry
[4] pry(main)> m.errors
=> #<ActiveModel::Errors [#<ActiveModel::Error attribute=director_id, type=blank, options={}>]>
```

The `messages` attribute of this object contains a `Hash` where the keys are the names of columns where something went wrong (as `Symbol`s), and the values are `Array`s of `Strings` describing what went wrong:

```pry
[5] pry(main)> m.errors.messages
=> {:director_id=>["can't be blank"]}
```

There's even a handy method called `.full_messages` which will assemble friendly error messages for us and return an `Array` of `String`s:

```pry
[6] pry(main)> m.errors.full_messages
=> ["Director can't be blank"]
```

Neat! Later, we will loop through this `Array` and display the friendly messages to our users to help them re-fill out forms.

## Commonly used built-in validators

By far the most common data integrity checks we need are:

 1. To make sure a value is present before saving. This is especially useful in the case of foreign keys, like we saw.
 2. To make sure a value is unique before saving (i.e., no other rows have the same value in that column).

Let's add a uniqueness validation:

```ruby
# app/models/movie.rb

class Movie < ApplicationRecord
  validates(:director_id, presence: true)
  validates(:title, uniqueness: true)

  def director
  # ...
end
```

If you're experimenting in `rails console`, don't forget to `exit` and re-launch it; model files are not automatically reloaded by `rails console` when we update them. But once we restart `rails console`, we can test our new validation:

```pry
[11] pry(main)> m = Movie.new
[12] pry(main)> m.title = "zebra"
[13] pry(main)> m.director_id = 1
[14] pry(main)> m.save
=> true
[15] pry(main)> n = Movie.new
[16] pry(main)> n.title = "zebra"
[17] pry(main)> n.save
   (0.1ms)  begin transaction
  Movie Exists? (0.2ms)  SELECT 1 AS one FROM "movies" WHERE "movies"."title" = ? LIMIT ?  [["title", "zebra"], ["LIMIT", 1]]
   (0.0ms)  rollback transaction
=> false
[18] pry(main)> n.errors.full_messages
=> ["Director can't be blank", "Title has already been taken"]
```

Great!

## Further reading

You can read a lot more about validations at the official Rails Guides:

[https://guides.rubyonrails.org/active_record_validations.html](https://guides.rubyonrails.org/active_record_validations.html)

But don't forget that the official Rails Guides use a lot of optional Ruby syntaxes to write more concise code than we like to in AppDev. Review [the lesson on different Ruby styles](https://learn.firstdraft.com/lessons/116-optional-syntaxes-in-ruby) if the code examples seem confusing.

---
