Кіраўнік #8. API для пошуку
========================

У цяперашні час сэрвіс Zip-кодаў вяртае адну запіс за раз (за запыт). Мы будзем пашыраць API, каб дазволіць атрыманне некалькіх Zip-кодаў адразу і фільтраваць калекцыю з дапамогай параметраў пошуку.

Мы павінны стварыць URL-шлях для атрымання калекцыі Zip-кодаў. Мы дазволім карыстальнікам фільтраваць Zip-коды па ўсіх атрыбутамі мадэлі, а таксама шукаць па пачатку атрыбуту zip (пошук кодаў, якія пачынаюцца з некалькіх канкрэтных знакаў).

## <a name="create-route"></a>Стварэнне шляху (URL)

Па-першае, мы створым URL-апрацоўшчык, які вяртае ўсе Zip-коды. Дадайце наступны код у `app/controllers/zip_codes_controller.rb`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.all
  zip_codes.to_json
end
```

Мы павінны праверыць яго: стварыць `rspec` тэст або праверыць яго ўручную. Па-першае, мы павінны запоўніць табліцу `zip_codes` ў базе дадзеных некаторымі тэставымі дадзенымі. `Sinatra-activerecord` ўтрымлівае `rake` задачу для гэтага - `db:seed`. Вы можаце ўбачыць усе даступныя `rake` задачы выканаўшы `rake -T` ў тэрмінале (з папкі `zip_codes`). Стварыце файл `db/seeds.rb` з інструкцыямі для стварэння 40 Zip-кодаў.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 40)
```

Мы выкарыстоўваем тую ж фабрыку (factory), якую мы выкарыстоўваем у тэстах.

Выканайце

    $ rake db:seed

І запусціце сервіс

    $ ruby application.rb

Затым перайдзіце ў іншую ўкладку (або акно) тэрмінала і выканайце

    $ curl localhost:4567/api/v1/zip_codes.json

    [{"zip_code":{"id":401,"zip":"71663","street_name":"Larue Lake","building_number":"2433","city":"Thompsonfurt","state":"Kansas","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-15T09:02:25.374Z"}},{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming",
    ...

Мы бачым даволі шмат Zip-кодаў у HTTP-адказе. Мы можам дадаць абмежаванне па змаўчанні, каб пазбегнуць гэтага. Абнавіце код у `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.limit(20)
  zip_codes.to_json
end
```

Мы можам дазволіць кліенту перадаваць `params[:limit]` для налады колькасці запісаў у адказе, але не будзем рабіць гэта для прастаты.

## <a name="database-indexes"></a>Параметры пошуку

Мы будзем выкарыстоўваць гем `ransack`, для пошуку запісаў па атрыбутам. Дадайце яго ў `Gemfile`. Я атрымаў памылку `uninitialized constant Rails::Railtie (NameError)`, калі `ransack` быў у `Gemfile` перад `rspec_api_documentation`. Дадайце `ransack` пасля `rspec_api_documentation`, каб пазбегнуць гэтую праблему.

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

Выканайце `bundle install`

І абновіце код у `app/controllers/zip_codes_controller.rb`.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params).result.limit(20)
  zip_codes.to_json
end
```

Цяпер `ransack` дазваляе выконваць пошук, напрыклад

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"

    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Вы можаце фільтраваць Zip-коды па параметрах тыпу `<атрыбут>_eq` або `<атрыбут>_start` для любога атрыбуту (імя слупка) табліцы базы дадзеных. Дакументацыя для [базавага пошуку](https://github.com/activerecord-hackery/ransack/wiki/Basic-Searching).

Мы павінны дазволіць толькі патрэбныя параметры пошуку і схаваць для карыстальнікаў ўсю моц `ransack`. Запыт `zip_start = 406` ператвараецца ў SQL` zip LIKE '406%'` (або `ILIKE`), запыт `zip_cont = 406` ператвараецца ў SQL `zip LIKE '% 406%'` (або `ILIKE`). У першым выпадку `BTREE` індэкс калонкі `zip` (індэкс пошуку па бінарнага дрэве) можа быць выкарыстаны PostgreSQL, у другім выпадку - не можа, і гэта прыводзіць да павольнага запыце.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes.to_json
end
```

Зараз усе іншыя параметры пошуку будуць ігнаравацца.

## <a name="database-indexes"></a>Індэксы базы дадзеных

Мы павінны дадаць індэксы базы дадзеных для табліцы `zip_codes` для хуткага пошуку. Трэба дадаць індэксы на кожны слупок, паколькі мы дазваляем пошук для кожнага слупка асобна. І гэтага будзе дастаткова для большасці выпадкаў, так як PostgreSQL можа аб'ядноўваць індэксы для пошуку.

Стварэнне міграцыі (выканайце ў тэрмінале з папкі `zip_codes`)

    $ rake db:create_migration NAME=add_indexes_to_zip_codes

І адрэдагуйце створаную міграцыю (новы файл у тэчцы `db/migrations`)

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

Выканайце міграцыю ў асяроддзі `development` (па змаўчанні) і асяроддзі `test`

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

## <a name="analysis-of-sql-query"></a>Аналіз SQL-запыту

Дадайце гэты код у `app/controllers/zip_codes_controller.rb` для прагляду SQL запытаў у тэрмінале.

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  puts "====  SQL  ===="
  puts zip_codes.to_sql
  zip_codes.to_json
end
```

Затым перазагрузіце сэрвіс (выканаўшы `ruby application.rb`) і звярніцеся да сэрвісу зноў два разы.

    $ curl "localhost:4567/api/v1/zip_codes.json?city_eq=Thompsonfurt&state_eq=Kansas"
    $ curl "localhost:4567/api/v1/zip_codes.json?zip_start=40664"

Атрымана два SQL запыту:

    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."city" = 'Thompsonfurt' AND "zip_codes"."state" = 'Kansas') LIMIT 20;

    SELECT  "zip_codes".* FROM "zip_codes"
    WHERE ("zip_codes"."zip" ILIKE '40664%') LIMIT 20;

Мы прааналізуем абодва запыту, але спачатку давайце створым пабольш запісаў, каб убачыць розніцу ў фактычным часу выканання SQL запытаў. Мы можам зрабіць гэта ў файле `seeds.rb` або ў кансолі (`script/console`). Гэта зойме некаторы час.

```ruby
require "factory_girl"
require "faker"
require File.join(settings.root, 'spec', 'factories', 'zip_codes.rb')

FactoryGirl.create_list(:zip_code, 100_000)
```

Цяпер у нас ёсць каля 100 000 запісаў у базе дадзеных (магчыма, вы дадалі некалькі раней). Адкрыйце `psql` (інтэрактыўны тэрмінал PostgreSQL).

    $ psql zipcodes_development

Аналіз першага запыту з дапамогай `EXPLAIN ANALYSE`, што дае нам план запыту і выконвае запыт вяртаючы фактычны час выканання.

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

`QUERY PLAN` цяжка чытаць, але мы можам бачыць час 0,247 мс, што даволі хутка, радок` Bitmap Index Scan on index_zip_codes_on_city` азначае, што быў выкарыстаны індэкс на слупку `city` - гэта добра.

Зараз давайце выканаем `EXPLAIN ANALYSE` для другога SQL запыту

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

І зноў быў выкарыстаны індэкс базы дадзеных. Шчыра кажучы, я не чакаў гэтага, я думаў, што PostgreSQL не можа выкарыстоўваць індэкс Bitmap (бінарнае дрэва) для ўмовы `ILIKE` (пошук без уліку рэгістра). У агульным выпадку база дадзеных не можа выкарыстоўваць бінарны індэкс для пошуку без уліку рэгістра, але ў гэтым выпадку радок, якая змяшчае толькі лічбы роўная самой сабе ў верхнім і ніжнім рэгістры.

Вы можаце ўбачыць розніцу паміж `LIKE` і` ILIKE` калі пошук ажыццяўляецца па радку, якая адрозніваецца ў верхнім і ніжнім рэгістры. Паспрабуйце параўнаць наступныя два запыту:

    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" LIKE 'foo%') LIMIT 20;
    EXPLAIN ANALYSE SELECT "zip_codes".* FROM "zip_codes" WHERE ("zip_codes"."zip" ILIKE 'foo%') LIMIT 20;

У першым выпадку індэкс выкарыстоўваецца, у другім - не (`QUERY PLAN` змяшчае радок" Seq Scan on zip_codes "замест" Index Scan using index_zip_codes_on_zip on zip_codes" і запыт значна павольней).

Гіпатэтычна, калі вам патрэбен толькі пошук з улікам рэгістра, мы можам замяніць `ILIKE` на `LIKE`

```ruby
get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes = zip_codes.where("zip LIKE ", "#{params[:zip_start]}%") if params[:zip_start]
  zip_codes.to_json
end
```

Звярніце ўвагу, што бінарны індэкс не можа быць выкарыстаны для пошуку па ўмове `zip LIKE '% 123%'`, у гэтым выпадку мы павінны мець індэкс для паўнатэкставага пошуку (full search index). Бінарны індэкс можа быць выкарыстаны толькі для пошуку, дзе мы дакладна ведаем некалькі знакаў з самага пачатку. Пошук пераўтворыць ўмова ў `((zip)::text >= '123'::text) AND ((zip)::text < '123'::text)`.

Адзначым таксама, што пытанне аб магчымым ужыванні індэкса можа залежаць ад ўстаноўкі базы дадзеных (не толькі версіі), напрыклад, калі база дадзеных не выкарыстоўвае стандартную "C" лакаль, і мы хочам выкарыстоўваць BTREE індэкс для `LIKE` запытаў мы павінны стварыць індэкс вызначыўшы клас аператара, см. [класы аператараў і операторного сямействаў](http://www.postgresql.org/docs/9.4/static/indexes-opclass.html).

```ruby
execute "CREATE INDEX index_zip_codes_on_zip ON zip_codes (postcode varchar_pattern_ops);"
```

## <a name="summary"></a>Рэзюмэ

Мы стварылі URL для атрымання спісу Zip-кодаў і выкарыстоўвалі гем [ransack](https://github.com/activerecord-hackery/ransack) для фільтрацыі запісаў. Мы стварылі індэксы базы дадзеных для аптымізацыі SQL-запытаў.
