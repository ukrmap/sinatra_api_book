Глава #8. API для пошуку
========================

В даний час сервіс Zip-кодів повертає один запис за раз (за запит). Ми будемо розширювати API, щоб дозволити отримання кількох Zip-кодів відразу і фільтрувати колекцію за допомогою параметрів пошуку.

Ми повинні створити URL-шлях для отримання колекції Zip-кодів. Ми дозволимо користувачам фільтрувати Zip-коди по всіх атрибутами моделі, а також шукати по початку атрибуту zip (пошук кодів, які починаються з кількох конкретних символів).

## <a name="create-route"></a>Створення шляху (URL)

По-перше, ми створимо URL-обробник, який повертає всі Zip-коди. Додайте наступний код в `app/controllers/zip_codes_controller.rb`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.all
  zip_codes.to_json
end
```

Ми повинні перевірити його: створити `rspec` тест або перевірити його вручну. По-перше, ми повинні заповнити таблицю `zip_codes` в базі даних деякими тестовими даними. `sinatra-activerecord` містить `rake` задачу для цього - `db:seed`. Ви можете побачити всі доступні `rake` задачі виконавши `rake -T` в терміналі (з теки `zip_codes`). Створіть файл `db/seeds.rb` з інструкціями для створення 40 Zip-кодів.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 40)
```

Ми використовуємо ту ж фабрику (factory), яку ми використовуємо в тестах.

Виконайте

    $ rake db:seed

І запустіть сервіс

    $ ruby application.rb

Потім перейдіть в іншу вкладку (або вікно) терміналу і виконайте

    $ curl localhost:4567/api/v1/zip_codes.json

    [{"zip_code":{"id":401,"zip":"71663","street_name":"Larue Lake","building_number":"2433","city":"Thompsonfurt","state":"Kansas","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-15T09:02:25.374Z"}},{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming",
    ...

Ми бачимо досить багато Zip-кодів в HTTP-відповіді. Ми можемо додати обмеження за замовчуванням, щоб уникнути цього. Змініть код в `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.limit(20)
  zip_codes.to_json
end
```

Ми можемо дозволити клієнту передавати `params[:limit]` для настройки кількості записів у відповіді, але не будемо робити це для простоти.

## <a name="database-indexes"></a>Параметри пошуку

Ми будемо використовувати гем `ransack`, для пошуку записів по атрибутам. Додайте його в `Gemfile`. Я отримав помилку `uninitialized constant Rails::Railtie (NameError)`, коли `ransack` був у `Gemfile` перед `rspec_api_documentation`. Додайте `ransack` після `rspec_api_documentation`, щоб уникнути цю проблему.

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

Виконайте `bundle install`

І поновіть код в `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params).result.limit(20)
  zip_codes.to_json
end
```

Тепер `ransack` дозволяє виконувати пошук, наприклад

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"

    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Ви можете фільтрувати Zip-коди за параметрами типу `<атрибут>_eq` або `<атрибут>_start` для будь-якого атрибута (ім'я стовпця) таблиці бази даних. Документація для [базового пошуку](https://github.com/activerecord-hackery/ransack/wiki/Basic-Searching).

Ми повинні дозволити тільки потрібні параметри пошуку і приховати для користувачів всю міць `ransack`. Запит `zip_start = 406` перетворюється в SQL `zip LIKE '406%'` (або `ILIKE`), запит `zip_cont = 406` перетворюється в SQL `zip LIKE '%406%'` (або `ILIKE`). У першому випадку `BTREE` індекс колонки `zip` (індекс пошуку за бінарним деревом) може бути використаний PostgreSQL, у другому випадку - не може, і це призводить до повільного запиту.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes.to_json
end
```

Тепер всі інші параметри пошуку будуть ігноруватися.

## <a name="database-indexes"></a>Індекси бази даних

Ми повинні додати індекси бази даних для таблиці `zip_codes` для швидкого пошуку. Потрібно додати індекси на кожен стовпець, оскільки ми дозволяємо пошук для кожного стовпця окремо. І цього буде досить для більшості випадків, так як PostgreSQL може об'єднувати індекси для пошуку.

Створення міграції (виконайте в терміналі з теки `zip_codes`)

    $ rake db:create_migration NAME=add_indexes_to_zip_codes

І відредагуйте створену міграцію (новий файл в теці `db/migraions`)

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

Виконайте міграцію в оточенні `development` (за замовчуванням) і оточенні 'test`

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

## <a name="analysis-of-sql-query"></a>Аналіз SQL-запиту

Додайте цей код в `app/controllers/zip_codes_controller.rb` для перегляду SQL запитів в терміналі.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  puts "====  SQL  ===="
  puts zip_codes.to_sql
  zip_codes.to_json
end
```

Потім перезавантажте сервіс (виконавши `ruby application.rb`) і зверніться до сервісу знову два рази.

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"
    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Отримано два SQL запити:

    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."city" = 'Thompsonfurt' AND "zip_codes"."state" = 'Kansas') LIMIT 20;
    
    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."zip" ILIKE '40664%') LIMIT 20;

Ми проаналізуємо обидва запити, але спочатку давайте створимо побільше записів, щоб побачити різницю у фактичному часі виконання SQL запитів. Ми можемо зробити це у файлі `seeds.rb` або в консолі (`script/console`). Це займе деякий час.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 100_000)
```

Тепер у нас є близько 100 000 записів в базі даних (можливо, ви додали кілька раніше). Відкрийте `psql` (інтерактивний термінал PostgreSQL).

    $ psql zipcodes_development

Аналіз першого запиту за допомогою `EXPLAIN ANALYSE`, що дає нам план запиту і виконує запит повертаючи фактичний час виконання.

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

`QUERY PLAN` важко читати, але ми можемо бачити час 0,247 мс, що досить швидко, рядок `Bitmap Index Scan on index_zip_codes_on_city` означає, що був використаний індекс на стовпці `city` - це добре.

Тепер давайте виконаємо `EXPLAIN ANALYSE` для другого SQL запиту

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

І знову був використаний індекс бази даних. Чесно кажучи, я не очікував цього, я думав, що PostgreSQL не може використовувати індекс Bitmap (бінарне дерево) для умови `ILIKE` (пошук без урахування регістру). У загальному випадку база даних не може використовувати бінарний індекс для пошуку без урахування регістру, але в цьому випадку рядок, який містить тільки цифри дорівнює самій собі у верхньому і нижньому регістрі.

Ви можете побачити різницю між `LIKE` і `ILIKE` якщо пошук здійснюється по рядку, який відрізняється у верхньому і нижньому регістрі. Спробуйте порівняти наступні два запити:

    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" LIKE 'foo%') LIMIT 20;
    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" ILIKE 'foo%') LIMIT 20;

В первом случае индекс используется, во втором - нет (`QUERY PLAN` містить рядок "Seq Scan on zip_codes" замість "Index Scan using index_zip_codes_on_zip on zip_codes" і запит набагато повільніше).

Гіпотетично, якщо вам потрібен тільки пошук з урахуванням регістру, ми можемо замінити `ILIKE` на` LIKE`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes = zip_codes.where("zip LIKE ", "#{params[:zip_start]}%") if params[:zip_start]
  zip_codes.to_json
end
```

Зверніть увагу, що бінарний індекс не може бути використаний для пошуку за умовою `zip LIKE '%123%'`, в цьому випадку ми повинні мати індекс для повнотекстового пошуку (full search index). Бінарний індекс може бути використаний тільки для пошуку, де ми точно знаємо декілька символів з самого початку. Пошук перетворює умову в `((zip)::text >= '123'::text) AND ((zip)::text < '123'::text)`.

Відзначимо також, що питання про можливе застосування індексу може залежати від установки бази даних (не тільки версії), наприклад, якщо база даних не використовує стандартну "C" локаль, і ми хочемо використати BTREE індекс для `LIKE` запитів ми повинні створити індекс визначивши клас оператора, див. [класи операторів і операторних сімейств](http://www.postgresql.org /docs/9.4/static/indexes-opclass.html).

```ruby
execute "CREATE INDEX index_zip_codes_on_zip ON zip_codes (postcode varchar_pattern_ops);" 
```

## <a name="summary"></a>Резюме

Ми створили URL для отримання списку Zip-кодів і використовували гем [ransack](https://github.com/activerecord-hackery/ransack) для фільтрації записів. Ми створили індекси бази даних для оптимізації SQL-запитів.
