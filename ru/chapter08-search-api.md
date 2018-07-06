Глава #8. API для поиска
========================

В настоящее время сервис Zip-кодов возвращает одну запись за раз (за запрос). Мы будем расширять API, чтобы разрешить получение нескольких Zip-кодов сразу и фильтровать коллекцию с помощью параметров поиска.

Мы должны создать URL-путь для получения коллекции Zip-кодов. Мы позволим пользователям фильтровать Zip-коды по всем атрибутами модели, а также искать по началу атрибута zip (поиск кодов, которые начинаются с нескольких конкретных символов).

## <a name="create-route"></a>Создание пути (URL)

Во-первых, мы создадим URL-обработчик, который возвращает все Zip-коды. Добавьте следующий код в `app/controllers/zip_codes_controller.rb`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.all
  zip_codes.to_json
end
```

Мы должны проверить его: создать `rspec` тест или проверить его вручную. Во-первых, мы должны заполнить таблицу `zip_codes` в базе данных некоторыми тестовыми данными. `sinatra-activerecord` содержит `rake` задачу для этого - `db:seed`. Вы можете увидеть все доступные `rake` задачи выполнив `rake -T` в терминале (из папки `zip_codes`). Создайте файл `db/seeds.rb` с инструкциями для создания 40 Zip-кодов.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 40)
```

Мы используем ту же фабрику (factory), которую мы используем в тестах.

Выполните

    $ rake db:seed

І запустите сервіс

    $ ruby application.rb

Затем перейдите в другую вкладку (или окно) терминала и выполните

    $ curl localhost:4567/api/v1/zip_codes.json

    [{"zip_code":{"id":401,"zip":"71663","street_name":"Larue Lake","building_number":"2433","city":"Thompsonfurt","state":"Kansas","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-15T09:02:25.374Z"}},{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming",
    ...

Мы видим довольно много Zip-кодов в HTTP-ответе. Мы можем добавить ограничение по умолчанию, чтобы избежать этого. Обновите код в `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.limit(20)
  zip_codes.to_json
end
```

Мы можем позволить клиенту передавать `params[:limit]` для настройки количества записей в ответе, но не будем делать это для простоты.

## <a name="database-indexes"></a>Параметры поиска

Мы будем использовать гем `ransack`, для поиска записей по атрибутам. Добавьте его в `Gemfile`. Я получил ошибку `uninitialized constant Rails::Railtie (NameError)`, когда `ransack` был в `Gemfile` перед `rspec_api_documentation`. Добавьте `ransack` после `rspec_api_documentation`, чтобы избежать эту проблему.

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

Выполните `bundle install`

И обновите код в `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params).result.limit(20)
  zip_codes.to_json
end
```

Теперь `ransack` позволяет выполнять поиск, например

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"

    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Вы можете фильтровать Zip-коды по параметрам типа `<атрибут>_eq` или `<атрибут>_start` для любого атрибута (имя столбца) таблицы базы данных. Документация для [базового поиска](https://github.com/activerecord-hackery/ransack/wiki/Basic-Searching).

Мы должны позволить только нужные параметры поиска и скрыть для пользователей всю мощь `ransack`. Запрос `zip_start = 406` превращается в SQL `zip LIKE '406%'` (или `ILIKE`), запрос `zip_cont = 406` превращается в SQL `zip LIKE '% 406%'` (или `ILIKE`). В первом случае `BTREE` индекс колонки `zip` (индекс поиска по бинарному дереву) может быть использован PostgreSQL, во втором случае - не может, и это приводит к медленному запросу.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes.to_json
end
```

Теперь все другие параметры поиска будут игнорироваться.

## <a name="database-indexes"></a>Индексы базы данных

Мы должны добавить индексы базы данных для таблицы `zip_codes` для быстрого поиска. Нужно добавить индексы на каждый столбец, поскольку мы позволяем поиск для каждого столбца отдельно. И этого будет достаточно для большинства случаев, так как PostgreSQL может объединять индексы для поиска.

Создание миграции (выполните в терминале из папки `zip_codes`)

    $ rake db:create_migration NAME=add_indexes_to_zip_codes

И отредактируйте созданную миграцию (новый файл в папке `db/migrations`)

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

Выполните миграцию в окружении `development` (по умолчанию) и окружении `test`

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

## <a name="analysis-of-sql-query"></a>Анализ SQL-запроса

Добавьте этот код в `app/controllers/zip_codes_controller.rb` для просмотра SQL запросов в терминале.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  puts "====  SQL  ===="
  puts zip_codes.to_sql
  zip_codes.to_json
end
```

Затем перезагрузите сервис (выполнив `ruby application.rb`) и обратитесь к сервису снова два раза.

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"
    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Получено два SQL запроса:

    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."city" = 'Thompsonfurt' AND "zip_codes"."state" = 'Kansas') LIMIT 20;

    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."zip" ILIKE '40664%') LIMIT 20;

Мы проанализируем оба запроса, но сначала давайте создадим побольше записей, чтобы увидеть разницу в фактическом времени выполнения SQL запросов. Мы можем сделать это в файле `seeds.rb` или в консоли (`script/console`). Это займет некоторое время.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 100_000)
```

Теперь у нас есть около 100 000 записей в базе данных (возможно, вы добавили несколько ранее). Откройте `psql` (интерактивный терминал PostgreSQL).

    $ psql zipcodes_development

Анализ первого запроса с помощью `EXPLAIN ANALYSE`, что дает нам план запроса и выполняет запрос возвращая фактическое время выполнения.

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

`QUERY PLAN` трудно читать, но мы можем видеть время 0,247 мс, что довольно быстро, строка `Bitmap Index Scan on index_zip_codes_on_city` означает, что был использован индекс на столбце `city` - это хорошо.

Теперь давайте выполним `EXPLAIN ANALYSE` для второго SQL запроса

    EXPLAIN ANALYSE SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."zip" ILIKE '40664%') LIMIT 20;
                                        QUERY PLAN
    -------------------------------------------------------------------------------------
     Limit  (cost=0.42..8.44 rows=10 width=67) (actual time=0.128..0.135 rows=2 loops=1)
       ->  Index Scan using index_zip_codes_on_zip on zip_codes  (cost=0.42..8.44 rows=10 width=67) (actual time=0.126..0.133 rows=2 loops=1)
             Index Cond: (((zip)::text >= '40664'::text) AND ((zip)::text < '40665'::text))
             Filter: ((zip)::text ~~* '40664%'::text)
     Total runtime: 0.211 ms
    (5 rows)

И снова был использован индекс базы данных. Честно говоря, я не ожидал этого, я думал, что PostgreSQL не может использовать индекс Bitmap (бинарное дерево) для условия `ILIKE` (поиск без учета регистра). В общем случае база данных не может использовать бинарный индекс для поиска без учета регистра, но в этом случае строка, которая содержит только цифры равна самой себе в верхнем и нижнем регистре.

Вы можете увидеть разницу между `LIKE` и `ILIKE` если поиск осуществляется по строке, которая отличается в верхнем и нижнем регистре. Попробуйте сравнить следующие два запроса:

    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" LIKE 'foo%') LIMIT 20;
    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" ILIKE 'foo%') LIMIT 20;

В первом случае индекс используется, во втором - нет (`QUERY PLAN` содержит строку "Seq Scan on zip_codes" вместо "Index Scan using index_zip_codes_on_zip on zip_codes" и запрос гораздо медленнее).

Гипотетически, если вам нужен только поиск с учётом регистра, мы можем заменить `ILIKE` на `LIKE`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes = zip_codes.where("zip LIKE ", "#{params[:zip_start]}%") if params[:zip_start]
  zip_codes.to_json
end
```

Обратите внимание, что бинарный индекс не может быть использован для поиска по условию `zip LIKE '%123%'`, в этом случае мы должны иметь индекс для полнотекстового поиска (full search index). Бинарный индекс может быть использован только для поиска, где мы точно знаем несколько символов с самого начала. Поиск преобразует условие в `((zip)::text >= '123'::text) AND ((zip)::text < '123'::text)`.

Отметим также, что вопрос о возможном применении индекса может зависеть от установки базы данных (не только версии), например, если база данных не использует стандартную "C" локаль, и мы хотим использовать BTREE индекс для `LIKE` запросов мы должны создать индекс определив класс оператора, см. [классы операторов и операторных семейств](http://www.postgresql.org /docs/9.4/static/indexes-opclass.html).

```ruby
execute "CREATE INDEX index_zip_codes_on_zip ON zip_codes (postcode varchar_pattern_ops);"
```

## <a name="summary"></a>Резюме

Мы создали URL для получения списка Zip-кодов и использовали гем [ransack](https://github.com/activerecord-hackery/ransack) для фильтрации записей. Мы создали индексы базы данных для оптимизации SQL-запросов.
