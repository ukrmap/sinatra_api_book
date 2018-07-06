Глава #5. JSON веб-сервіси
==========================

Веб-сервіси можуть використовувати різні формати для серіалізациі даних. Найбільш поширеними є XML і JSON. JSON - простий формат, який до того ж прекрасно підходить для JavaScript-додатків, які працюють у веб-браузері. Одним з обмежень при використанні JSON є відсутність вбудованої можливості вказувати типи даних (наприклад, `String`,` Integer`, `Array`). Клієнт повинен самостійно визначати тип даних. У більшості випадків це очевидно і обробляється JSON парсером.

Приклад даних в JSON форматі:

    {"name":"Lyric Adams","account":208.6,"orders":[456,803,1204],"address":"871 Tommie Roads","city":"Halchester","birthday":"1986-05-07","registeration_date":"2015-02-23"}

Ті ж дані з відступами:

    {
      "name": "Lyric Adams",
      "account": 208.6,
      "orders": [456,803,1204],
      "address": "871 Tommie Roads",
      "city": "Halchester",
      "birthday": "1986-05-07",
      "registeration_date": "2015-02-23"
    }

Тут представлені дані про клієнта у вигляді сховища ключ-значення. Аналізатор (парсер) JSON здатний визначити типи деяких атрибутів: `"Lyric Adams"` - рядок (`String`), `208.6` - число з плаваючою точкою, `[456,803,1204]` - масив цілих чисел. Атрибути `"birthday"` і `"registeration_date"` містять строкові значення. Типи даних для дати і часу представлені у вигляді рядків у форматі JSON.

Є принаймні два способи для того, щоб перетворити всі рядки з датою в тип даних для дати: перевірити всі значення, які відповідають регулярному виразу для дати і часу і перетворити їх, або конвертувати тільки значення із заданого списку ключів (ви повинні знати заздалегідь, звідкись, всі ключі, які містять значення дати і часу).

Кілька наступних глав присвячені розробці веб-сервісу для управління Zip-кодами (поштові індекси США). У цій главі ми створимо сервіс, який надає дані у форматі JSON і обробляє дані у форматі JSON із запитів.

## <a name="create-zip-codes-service-structure"></a>Створення структури сервісу Zip-кодів

Створіть, будь ласка, теку `zip_codes` десь у вашій системі. Створіть теки `app`,` config`, `db`,` doc`, `log`,` script`, `spec`. Створіть теку `models` і` controllers` всередині теки `app`, а також створіть теки` acceptance`, `factories`,` models` всередині теки `spec`. Створіть файл `config / database.yml` з вашими настройками бази даних, ось моя версія:

```yaml
development:
  adapter: postgresql
  encoding: unicode
  database: zipcodes_development
  username: alex

test:
  adapter: postgresql
  encoding: unicode
  database: zipcodes_test
  username: alex
```

Створіть файл `script/console`

```ruby
#!/bin/bash

# parameter: RACK_ENV
bundle exec irb -r ./application.rb
```

І зробіть його виконуваним (для UNIX-подібних систем):

    $ chmod +x script/console

Створіть файл `spec/spec_helper.rb`

```ruby
ENV['RACK_ENV'] = 'test'
require File.expand_path("../../application", __FILE__)

FactoryGirl.find_definitions

RSpec.configure do |config|
  config.include Rack::Test::Methods
  config.include FactoryGirl::Syntax::Methods
  config.default_formatter = 'doc' if config.files_to_run.one?

  def app
    Sinatra::Application
  end

  config.before(:suite) do
    DatabaseCleaner.clean_with :truncation
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end
end

require "rspec_api_documentation/dsl"

RspecApiDocumentation.configure do |config|
  config.docs_dir = Pathname.new(Sinatra::Application.root).join("doc")
  config.app = Sinatra::Application
  config.api_name = "Zip-Codes API"
  config.format = :html
  config.curl_host = 'https://zipcodes.example.com'
  config.curl_headers_to_filter = %w(Host Cookie)
end
```

Зверніть увагу, що ми оновили конфігурацію для гема `rspec_api_documentation`. Параметр налаштування `curl_host` важливий, якщо він встановлений документація буде содержать` curl` приклад, який часто використовується для налагодження веб-сервісів.

Якщо ви використовуєте `git` додайте файл` .gitignore` всередину теки `zip_codes`.

    log/*.log
    doc/*

А також ви можете додати порожній файл з ім'ям `.keep` (або `.gitkeep`) всередину тек `log`, `doc` і `lib/tasks` (ми не створювали останню теку).

Створіть файл `Gemfile` всередині теки `zip_codes`

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra', require: 'sinatra/main'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'

group :development, :test do
  gem 'thin'
  gem 'pry-debugger'
  gem 'rspec_api_documentation'
end

group :test do
  gem 'rspec'
  gem 'shoulda'
  gem 'factory_girl'
  gem 'database_cleaner'
  gem 'rack-test'
  gem 'faker'
end
```

І виконайте

    $ bundle install

Створіть файл `Rakefile`

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

Створіть файл `application.rb` всередині теки `zip_codes`

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }
```

Ви також можете створити файл `.rspec` з налаштуваннями для RSpec

    --color
    --require spec_helper

Рядок `--require spec_helper` дозволяє автоматично підключити файл `spec_helper.rb` в усі тестові файли. Таким чином, ви можете не вказувати явно `require" spec_helper "` в кожному тестовому файлі (ми будемо це вказувати в будь-якому випадку).

Подивіться на структуру створеної програми:

![Basic gem structure](../static/images/zip_codes_structure.png)

## <a name="create-databases"></a>Створення баз даних

Якщо ви ще не створили `development` і `test` бази даних, виконайте `rake db:create` (в терміналі з теки `zip_codes`). Це створить обидві бази даних.

## <a name="create-model-and-migration"></a>Створення моделі і міграції

Ми створимо одну модель для Zip-кодів з п'ятьма атрибутами: ZIP, назва вулиці, номер будинку, місто, штат. Всі атрибути є рядками. Атрибут ZIP є обов'язковим (не може бути порожнім) і повинен відповідати (перевірятися) наступному регулярному виразу: `/\A\d{5}(?:-\d{4})?\Z/` (5 цифр або 9 цифр розділених символом "-" після 5-й цифри).

Створіть, будь ласка, міграцію, виконайте наступну `rake` задачу:

    $ rake db:create_migration NAME=create_zip_codes

Оновлення коду міграції (файл, який був створений в теці `db/migrate`):

```ruby
class CreateZipCodes < ActiveRecord::Migration
  def change
    create_table :zip_codes do |t|
      t.string :zip, null: false
      t.string :street_name
      t.string :building_number
      t.string :city
      t.string :state

      t.timestamps null: false
    end

    add_index :zip_codes, :zip
  end
end
```

Виконайте міграцію для `development` і `test` бази даних

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

Тепер ми можемо створити модель - файл `app/models/zip_code.rb`

```ruby
class ZipCode < ActiveRecord::Base
  validates :zip, presence: true
  validates_format_of :zip, with: /\A\d{5}(?:-\d{4})?\Z/

  attr_accessible :zip, :street_name, :building_number, :city, :state
end
```

І тести для моделі - файл `spec/models/zip_code_spec.rb`

```ruby
require "spec_helper"

describe ZipCode do
  describe "validations" do
    it { should validate_presence_of(:zip) }

    it { is_expected.to allow_value('12345').for(:zip) }
    it { is_expected.to allow_value('12345-1234').for(:zip) }
    it { is_expected.not_to allow_value('123ab').for(:zip) }
    it { is_expected.not_to allow_value('123456').for(:zip) }
    it { is_expected.not_to allow_value('12345-123').for(:zip) }
  end

  describe 'assignament' do
    it { is_expected.not_to allow_mass_assignment_of(:id) }
    it { is_expected.to allow_mass_assignment_of(:zip) }
    it { is_expected.to allow_mass_assignment_of(:street_name) }
    it { is_expected.to allow_mass_assignment_of(:building_number) }
    it { is_expected.to allow_mass_assignment_of(:city) }
    it { is_expected.to allow_mass_assignment_of(:state) }
  end
end
```

Ви можете запустити тести командою `rspec`.

Створіть, будь ласка, файл `spec/factories/zip_codes.rb` з фабрикою для створення Zip-кодів в тестах. Ми будемо використовувати її найближчим часом в `acceptance` тестах.

```ruby
FactoryGirl.define do
  factory :zip_code do
    zip { Faker::Address.zip }
    street_name { Faker::Address.street_name }
    building_number { Faker::Address.building_number }
    city { Faker::Address.city }
    state { Faker::Address.state }
  end
end
```

Ми використовували гем [faker] (https://github.com/stympy/faker) для генерації різних атрибутів моделі.

## <a name="top-level-interface-planning"></a>Планування інтерфейсу верхнього рівня

Веб-сервіс повинен повертати JSON уявлення Zip-коду і бути в змозі обробляти JSON закодовані параметри для створення/оновлення Zip-коду.

Отримання Zip-коду: запит і відповідь.

    $ curl "https://localhost:4567/api/v1/zip_codes/53796.json" -X GET

    {"zip_code":{"id":2,"zip":"53796","street_name":"Johnston Forest",
    "building_number":"463","city":"Mosciskiville","state":"Connecticut",
    "created_at":"2015-02-09T15:20:42.474Z","updated_at":"2015-02-09T15:20:42.474Z"}}

Створення Zip-коду: запит і відповідь.

    $ curl "https://localhost:4567/api/v1/zip_codes.json" \
    $ -X POST \
    $ -H "Content-Type: application/json" \
    $ -d '{"zip_code":{"zip":"31460-3046","street_name":"Cartwright Dale", \
    $ "building_number":"77779","city":"Ovaside","state":"South Dakota"}}'

    {"zip_code":{"id":1,"zip":"31460-3046","street_name":"Cartwright Dale",
    "building_number":"77779","city":"Ovaside","state":"South Dakota",
    "created_at":"2015-02-09T15:20:42.440Z","updated_at":"2015-02-09T15:20:42.440Z"}}

Функціонал для JSON-серіалізації включений в гем `activerecord`. Для більшої кількості можливостей налаштування, ви можете використовувати [active_model_serializers](https://github.com/rails-api/active_model_serializers) або [JBuilder](https://github.com/rails/jbuilder).

Для розбору JSON з тіла `POST` або` PUT` HTTP запиту, ми можемо використовувати `Rack::PostBodyContentTypeParser`` middleware` з [rack-contrib](https://github.com/rack/rack-contrib).

Веб-сервіс повинен також передавати HTTP заголовок `Content-Type: application/json` в кожній відповіді.

## <a name="create-controller"></a>Створення контролера

Додамо `rack-contrib` в `Gemfile`. Ми будемо використовувати новітню версію з GitHub.

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra', require: 'sinatra/main'
# use Rack::PostBodyContentTypeParser to add support for JSON request bodies
gem 'rack-contrib', git: 'https://github.com/rack/rack-contrib'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'

group :development, :test do
  gem 'thin'
  gem 'pry-debugger'
  gem 'rspec_api_documentation'
end

group :test do
  gem 'rspec'
  gem 'shoulda'
  gem 'factory_girl'
  gem 'database_cleaner'
  gem 'rack-test'
  gem 'faker'
end
```

Змініть `application.rb` для того, щоб додати підтримку для розбору JSON-тіла HTTP запиту за допомогою `Rack::PostBodyContentTypeParser` `middleware`, вкажіть заголовок HTTP відповіді`"Content-Type: application/json"`, а також додайте опцію конфігурації для методу `ActiveRecord::Base#to_json`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

# Support for JSON request bodies
use Rack::PostBodyContentTypeParser

# Adds "Content-Type: application/json" HTTP header in response
before { content_type :json }

# Configure method ActiveRecord::Base#to_json to add root node at top level,
# e.g. {"zip_code":{"zip": ... }}
ActiveRecord::Base.include_root_in_json = true
```

`Rack::PostBodyContentTypeParser` перетворює JSON, що передається в тілі запиту HTTP в хеш `params`.

Створіть, будь ласка, файл `app/controllers/zip_codes_controller.rb` з чотирма CRUD-операціями для управління Zip-кодами.

```ruby
post "/api/v1/zip_codes.json" do
  zip_code = ZipCode.new(params[:zip_code])
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes/:zip.json" do
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

І `acceptance` тести у файлі `spec/acceptance/zip_codes_spec.rb`

```ruby
require "spec_helper"

resource 'ZipCode' do
  post "/api/v1/zip_codes.json" do
    header "Content-Type", "application/json"

    parameter :zip, "Zip", scope: :zip_code, required: true
    parameter :street_name, "Street name", scope: :zip_code
    parameter :building_number, "Building number", scope: :zip_code
    parameter :city, "City", scope: :zip_code
    parameter :state, "State", scope: :zip_code
    let(:raw_post) { params.to_json }

    # let(:valid_attributes) do
    #   { zip: "35761-7714", street_name: "Lavada Creek",
    #       building_number: "88871", city: "New Herminaton", state: "Rhode Island" }
    # end
    let(:valid_attributes) { attributes_for(:zip_code) }
    let(:new_zip_code) { ZipCode.last }

    example "Create Zip Code" do
      do_request(zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 201
      expect(json_response[:zip_code].values_at(*valid_attributes.keys)).to eq valid_attributes.values
      expect(new_zip_code).to be_present
      expect(new_zip_code.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
    end
  end

  get "/api/v1/zip_codes/:zip.json" do
    parameter :zip, "Zip", scope: :zip_code, required: true

    let(:zip_code) { create(:zip_code) }

    example "Read Zip Code" do
      do_request(zip: zip_code.zip)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 200
      expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
        zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
    end
  end

  put "/api/v1/zip_codes/:id.json" do
    header "Content-Type", "application/json"

    parameter :id, "Record ID", required: true
    parameter :street_name, "Street name", scope: :zip_code
    parameter :building_number, "Building number", scope: :zip_code
    parameter :city, "City", scope: :zip_code
    parameter :state, "State", scope: :zip_code
    let(:raw_post) { params.to_json }

    let(:zip_code) { create(:zip_code) }
    let(:valid_attributes) { attributes_for(:zip_code) }

    example "Update Zip Code" do
      do_request(id: zip_code.id, zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 200
      expect(json_response[:zip_code].values_at(:zip, :street_name, :building_number, :city, :state)).to eq(
        valid_attributes.values_at(:zip, :street_name, :building_number, :city, :state))
      expect(zip_code.reload.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
    end
  end

  delete "/api/v1/zip_codes/:id.json" do
    parameter :id, "Record ID", required: true

    let(:zip_code) { create(:zip_code) }

    example "Delete Zip Code" do
      do_request(id: zip_code.id)

      expect(status).to eq 200
      expect(ZipCode.where(id: zip_code.id)).to be_empty
    end
  end
end
```

От і все! (на час). Ми створили веб-сервіс, який працює, але не включає в себе обробку невірного введення. Ми зробимо це в наступному розділі.

## <a name="summary"></a>Резюме

Ми використали метод `to_json` з `activerecord` для JSON-серіалізациі і `middleware` з [rack-contrib](https://github.com/rack/rack-contrib) для десеріалізациі (розбору) JSON параметрів з POST/PUT HTTP-запитів.

Ми також створили `acceptance` тести. Майже кожен тест перевіряє правильність конкретних JSON-атрибутів, і ми використовуємо `JSON.parse` в кожному тесті. Ви, напевно, повинні мати допоміжний-метод для цього, ось корисна стаття: [Rails API Testing Best Practices](http://matthewlehner.net/rails-api-testing-guidelines/) (незважаючи на "Rails" в назві, стаття містить інструкції, які можуть бути використані в будь-яких `rspec` + `rack-test` тестах).

Деякі геми для тестування JSON:

* [json_spec](https://github.com/collectiveidea/json_spec)
* [airborne](https://github.com/brooklynDev/airborne)
* [json_expressions](https://github.com/chancancode/json_expressions)
* [match_json](https://github.com/WhitePayments/match_json)

Щоб зберігати чіткі і підтримувані тести, я можу запропонувати перевіряти тільки основні атрибути в прикладах з одним записом і перевіряти тільки ідентифікатори (IDs) для прикладів з колекцією записів.
