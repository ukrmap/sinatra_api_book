Chapter #8. Search API
======================

Currently Zip-codes service returns one record at а time (per request). We will extend API to allow retrieve several Zip-codes at once and filter collection by search parameters.

We need create route for retrieving collections of Zip-codes. We will allow users to query Zip-codes by all model attributes and also by beginning of Zip code (search Zip-codes that start from few requested characters).

## <a name="create-route"></a>Create route

First we will create route that returns all Zip-codes. Add this to `app/controllers/zip_codes_controller.rb`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.all
  zip_codes.to_json
end
```

We need to test it: create `rspec` test or check it manually. First we need to populate table `zip_codes` in the database with some test data. `sinatra-activerecord` supplies `rake` task for this - `db:seed`. You can see all available `rake` tasks: run `rake -T` in terminal (from folder `zip_codes`). Create file `db/seeds.rb` with instructions for creating 40 Zip-codes.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 40)
```

We are using same factory that we use in tests.

Run

    $ rake db:seed

And start service

    $ ruby application.rb

Then navigate to another terminal window and run

    $ curl localhost:4567/api/v1/zip_codes.json

    [{"zip_code":{"id":401,"zip":"71663","street_name":"Larue Lake","building_number":"2433","city":"Thompsonfurt","state":"Kansas","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-15T09:02:25.374Z"}},{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming",
    ...

We see quite a few (all of them) Zip-codes in the response. We can add default limit to avoid this. Update code in `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.limit(20)
  zip_codes.to_json
end
```

We can allow client to pass `params[:limit]` for configuring records quantity in the response but will not do this here for simplicity.

## <a name="database-indexes"></a>Search parameters

We will use gem `ransack` to allow search records by attributes. Add it it into `Gemfile`. I had `uninitialized constant Rails::Railtie (NameError)` error if `ransack` was present before `rspec_api_documentation`. Add `ransack` after `rspec_api_documentation` to avoid this problem.

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra', require: 'sinatra/main'
gem 'rack-contrib', git: 'https://github.com/rack/rack-contrib'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'
gem 'sinatra-param'
gem 'faraday'
gem 'sinatra-can'

group :development, :test do
  gem 'thin'
  gem 'pry-debugger'
  gem 'rspec_api_documentation'
end

gem 'ransack'

group :test do
  gem 'rspec'
  gem 'shoulda'
  gem 'factory_girl'
  gem 'database_cleaner'
  gem 'rack-test'
  gem 'faker'
  gem 'fakeweb'
end
```

Run `bundle install`

Then update code in `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params).result.limit(20)
  zip_codes.to_json
end
```

Now `ransack` allows to make search, e.g.

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"

    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

You can search Zip-codes with parameters like `<attribute>_eq` or `<attribute>_start` for any database table attribute (column name). Documentation for [basic searching](https://github.com/activerecord-hackery/ransack/wiki/Basic-Searching).

We need to allow only desired searching parameters and hide for user all power of `ransack`. Querying `zip_start = 406` transforms to SQL `zip LIKE '406%'` (or `ILIKE`), querying `zip_cont = 406` transforms to SQL `zip LIKE '%406%'` (or `ILIKE`). In first example btree index on column `zip` can be used by PostgreSQL, in second - cann't, and this will produce slow query.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes.to_json
end
```

Now all other search parameters will be ignored.

## <a name="database-indexes"></a>Database indexes

We must add the database indexes for the table `zip_codes` for quick search. Need to add indexes on each individual column as we allow search for every column separately. And this would be enough for most cases due to PostgreSQL ability to combine indexes.

Create migration (run in terminal from folder `zip_codes`)

    $ rake db:create_migration NAME=add_indexes_to_zip_codes

And update created migration (new file in folder `db/migraions`)

```ruby
class AddIndexesToZipCodes < ActiveRecord::Migration
  def up
    add_index :zip_codes, :street_name
    add_index :zip_codes, :building_number
    add_index :zip_codes, :city
    add_index :zip_codes, :state
  end

  def down
    remove_index :zip_codes, :street_name
    remove_index :zip_codes, :building_number
    remove_index :zip_codes, :city
    remove_index :zip_codes, :state
  end
end
```

Run migration in environment `development` (by default) and environment `test`

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

## <a name="analysis-of-sql-query"></a>Analysis of SQL-query

Add this code in `app/controllers/zip_codes_controller.rb` to view SQL queries in the terminal.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  puts "====  SQL  ===="
  puts zip_codes.to_sql
  zip_codes.to_json
end
```

Then restart the service (with `ruby application.rb`) and request it again two times.

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"
    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Logged two SQL queries:

    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."city" = 'Thompsonfurt' AND "zip_codes"."state" = 'Kansas') LIMIT 20;
    
    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."zip" ILIKE '40664%') LIMIT 20;

We will analyse both queries but first let's generate a lot Zip-codes to see difference in actual execution time of SQL queries. We can do this in file `seeds.rb` or in console (`script/console`). This will take some time.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 100_000)
```

Now we have about 100000 Zip-codes in database (maybe you had few before). Open `psql` (PostgreSQL interactive terminal).

    $ psql zipcodes_development

Analyse first query with `EXPLAIN ANALYSE` that gives us query plan and performs query and returns execution time.

    EXPLAIN ANALYSE SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."city" = 'Thompsonfurt' AND "zip_codes"."state" = 'Kansas')
    LIMIT 20;
                                        QUERY PLAN
    -------------------------------------------------------------------------------------
     Limit  (cost=4.43..12.22 rows=1 width=67) (actual time=0.108..0.109 rows=1 loops=1)
       ->  Bitmap Heap Scan on zip_codes  (cost=4.43..12.22 rows=1 width=67) (actual time=0.104..0.104 rows=1 loops=1)
             Recheck Cond: ((city)::text = 'Thompsonfurt'::text)
             Filter: ((state)::text = 'Kansas'::text)
             ->  Bitmap Index Scan on index_zip_codes_on_city  (cost=0.00..4.43 rows=2 width=0) (actual time=0.074..0.074 rows=1 loops=1)
                   Index Cond: ((city)::text = 'Thompsonfurt'::text)
     Total runtime: 0.247 ms
    (7 rows)

`QUERY PLAN` is hard to read but we can see time 0.247 ms which is pretty fast, string `Bitmap Index Scan on index_zip_codes_on_city` means that index on column `city` was used - that is good.

Now let's `EXPLAIN ANALYSE` second SQL query

    EXPLAIN ANALYSE SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."zip" ILIKE 'a0664%') LIMIT 20;
                                        QUERY PLAN
    -------------------------------------------------------------------------------------
     Limit  (cost=0.42..8.44 rows=10 width=67) (actual time=0.128..0.135 rows=2 loops=1)
       ->  Index Scan using index_zip_codes_on_zip on zip_codes  (cost=0.42..8.44 rows=10 width=67) (actual time=0.126..0.133 rows=2 loops=1)
             Index Cond: (((zip)::text >= '40664'::text) AND ((zip)::text < '40665'::text))
             Filter: ((zip)::text ~~* '40664%'::text)
     Total runtime: 0.211 ms
    (5 rows)

And again database index was used. To be honest I didn't expect this, I thought that PostgreSQL cannot use Bitmap index (binary tree) for condition `ILIKE` (case insensitive search). Generally database cannot use binary index for case insensitive search, but in this case string which contains only numbers is equal to itself in upper and lower case.

You can see difference between `LIKE` and `LIKE` if search is performed on string that is different in upper and lower register. Try to compare next two queries:

    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" LIKE 'foo%') LIMIT 20;
    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" ILIKE 'foo%') LIMIT 20;

In first case index is used, in second is not (`QUERY PLAN` contains string "Seq Scan on zip_codes" instead "Index Scan using index_zip_codes_on_zip on zip_codes" and query is much slower).

Hypothetically if you need only case sensitive search we can replace `LIKE` with `LIKE`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes = zip_codes.where("zip LIKE ", "#{params[:zip_start]}%") if params[:zip_start]
  zip_codes.to_json
end
```

Note that binary tree index can not be used for search by condition `zip LIKE '%123%'`, for this case we need to have full search index. Binary index can be used only for search where we know exact few characters from the beginning. Search transforms condition to `((zip)::text >= '123'::text) AND ((zip)::text < '123'::text)`.

Note also that possible usage of index can depend on database installation (not only version), e.g. when the database does not use the standard "C" locale and we want to use btree index for `LIKE` queries we need create index with operator class, see [Operator Classes and Operator Families](http://www.postgresql.org/docs/9.4/static/indexes-opclass.html).

```ruby
execute "CREATE INDEX index_zip_codes_on_zip ON zip_codes (postcode varchar_pattern_ops);" 
```

## <a name="summary"></a>Summary

We created route for retrieving Zip-codes collections and used gem [ransack](https://github.com/activerecord-hackery/ransack) for filtering records. We created database indexes to optimise SQL queries.
