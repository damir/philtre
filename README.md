# Philtre

It's the [Sequel](http://sequel.jeremyevans.net) equivalent for Ransack, Metasearch, Searchlogic. If
this doesn't make you fall in love, I don't know what will :-p

See philtre-rails for rails integration.

## Installation

Add this line to your application's Gemfile:

    gem 'philtre'

Or for all the rails integration goodies

    gem 'philtre-rails'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install philtre

## Basic Usage

Parse the predicates on the end of field names, and modify a Sequel::Dataset
to retrieve matching rows.

So, using a fairly standard rails-style parameter hash:

``` ruby
  filter_parameters = {
    birth_year: ['2012', '2011'],
    title: 'bar',
    order: ['title', 'name_asc', 'birth_year_desc'],
  }

  # This would normally be a real Sequel::Dataset
  personages_dataset = Sequel.mock[:personages]

  philtre = Philtre.new( filter_parameters ).apply( personages_dataset ).sql
```

should result in (formatting added here for clarity)

``` SQL
  SELECT *
  FROM "personages"
  WHERE
    (("birth_year" IN ('2012', '2011'))
    AND
    ("title" = 'bar'))
  ORDER BY ("title" ASC, "name" ASC, "date" DESC)
```

## Predicates

```{title: 'sir'}``` is fine when you want to match on string equality. But
there are all kinds of other things you need to do. For example

```{title_like: 'sir', age_gt: 10}``` is for a where clause ```title ~* 'sir' and age >  10```

There are a range of predefined predicates, mostly borrowed from the other search gems:

```
  gt
  gte, gteq
  lt
  lte, lteq
  eq
  not_eq
  matches, like
  not_blank
  like_all
  like_any
```

## Custom Predicates

There are two ways:

1) You can also define your own by creating a Filter with a block:

``` ruby
  philtre = Philtre.new filter_parameters do
    def tagged_by_id(tag_ids)
      Tag.db[:projects_tags]
        .select(:personage_id)
        .filter(tag_id: tag_ids, :project_id => :personage__id )
        .exists
    end

    def really_fancy(tag_ids)
      # do some really fancy SQL here
    end

    # etc...
  end
```

Now you can pass the filter_parameter hash ```{tagged_by_id: 45}```.

The result of a predicate block should be a ```Sequel::SQL::Expression``` (ie
one of Sequel's hash expressions in the simplest case) which will work instead
of its named placeholder. That is, if the placeholder is inside a SELECT
clause it worked work to give in an ORDER BY.

2) You could also inherit from ```Philtre::Filter``` and override
```#predicates```. And optionally override ```Philtre.new``` (which is just a
factory method on ```module Philtre```) to return the instance of your class.

## Advanced usage

There is also the ```Philtre::Grinder``` class which can insert placeholders into
your ```Sequel::Dataset``` definition, and then substitute those once it has the
parameter hash. Effectively this makes it a SQL macro engine.

Why so complicated? Well, it's really handy when you need to use aggregate
queries, and apply different values in the parameter hash to where clauses
inside and the outside of the aggregation. For example, give me a list of all
stores in a particular region who share of total sales was more than some
percentage. Yes, you can also use window functions to deal with that
particular query.

``` ruby
  # This would normally be a real Sequel::Dataset
  stores_dataset = Sequel.mock[:stores]

  # parameterise it with placeholders
  parameterised_dataset = stores_dataset.filter( :region.lieu, :sales_gt.lieu, :manager.lieu )

  filter_parameters = {
    region: 'The Bundus',
    sales_gt: 10,
    order: ['store_name', 'sales_desc'],
  }

  # generate the SQL you need
  parameterised_dataset.grind( Philtre.new( filter_parameters ) ).sql
```

will result in

``` SQL
  SELECT *
  FROM stores
  WHERE ((region = 'The Bundus') AND (sales > 10))
```

Notice that the manager part of the where clause is absent because
filter_parameters didn't have a manager key.

Look at the sql generated by parameterised_dataset and you'll see the placeholders
marked by SQL comments, so you can debug the Giant SQL Statement more easily. You
might also want to find a command-line SQL pretty printer (eg ``fsqlf```) and use it to produce
readable SQL instead of a very long hard-to-read string.

If you don't like the monkey-patching of Symbol with #lieu, you can use
several other ways to generate the placeholders. ```Philtre::PlaceHolder.new```
is canonical in that all the other possibilities use it.

## Highly Advanced Usage

Sometimes method chaining gets ugly. So you can say

``` ruby
  store_id_range = 20..90
  parameterised_dataset = stores_dataset.rolled do
    where :region.lieu, :sales_gt.lieu, :manager.lieu
    where store_id: store_id_range
    select_append db[:products].join(:stores, :store_id => :id ).select(:product_name)
  end
```

Notice that values outside the block are accessible inside, _without_
the need for a block parameter. This uses Ripar under the cover and indirects
the binding lookup, so may result in errors that you won't expect.

## Specs

Nothing fancy. Just:

    $ rspec spec

## Contributing

1. Fork it ( http://github.com/djellemah/philtre/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
