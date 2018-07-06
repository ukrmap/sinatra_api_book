Кіраўнік # 5. JSON вэб-сэрвісы
==============================

Вэб-сэрвісы могуць выкарыстоўваць розныя фарматы для серыялізацыі дадзеных. Найбольш распаўсюджанымі з'яўляюцца XML і JSON. JSON - просты фармат, які да таго ж выдатна падыходзіць для JavaScript-прыкладанняў, якія працуюць у вэб-браўзэры. Адным з абмежаванняў пры выкарыстанні JSON з'яўляецца адсутнасць убудаванай магчымасці ўказваць тыпы дадзеных (напрыклад, `String`, `Integer`, `Array`). Кліент павінен самастойна вызначаць тып дадзеных. У большасці выпадкаў гэта відавочна і апрацоўваецца JSON парсерам.

Прыклад дадзеных у JSON фармаце:

    {"name":"Lyric Adams","account":208.6,"orders":[456,803,1204],"address":"871 Tommie Roads","city":"Halchester","birthday":"1986-05-07","registeration_date":"2015-02-23"}

Тыя ж дадзеныя в е водступамі:

    {
      "name": "Lyric Adams",
      "account": 208.6,
      "orders": [456,803,1204],
      "address": "871 Tommie Roads",
      "city": "Halchester",
      "birthday": "1986-05-07",
      "registeration_date": "2015-02-23"
    }

Тут представленны дадзеныя пра кліента Ў выглядзе сховішчы ключ-значэнне. Аналізатар (парсер) JSON здольны вызначыць тыпы некаторых атрыбутаў: `"Lyric Adams"` - радок, `208.6` - лік в е якаючы якая плавае Кропкай, `[456,803,1204]` - масіў цэлых лікаў. Атрыбуты `"birthday"` і `"registeration_date"` ўтрымліваюць радковыя значэння. Тыпы дадзеных для даты і гадзіне прадстаўлены Ў выглядзе радкоў у фармацыі JSON.

Ёсць па меншай меры два спосабу для Таго, Каб пераўтварыць ўсе радкі в е дата ў тып дадзеных для даты: праверыць усе значэння, Якія адпавядаюць рэгулярнаму выказаць для даты і гадзіне і пераўтварыць ІХ, Або канвертаваць Толькі значэння в е зададзенага спісу ключоў (вы павінны ведаць загадзя , аднекуль, усе ключы, Якія ўтрымліваюць значэння даты і гадзіне).

Некалькі наступного кіраўнікоў прысвечаны распрацоўцы вэб-сэрвісу для кіравання Zip-кодамі (паштовыя індэксы ЗША). У гэтя частцы мы створ сэрвіс, які Падае дадзеныя Ў фармацыі JSON і апрацоўвае дадзеныя Ў фармацыі JSON в е запытаў.

## <a name="create-zip-codes-service-structure"></a>Стварэнне структуры сэрвісу Zip-кодаў

Стварыце, калі ласка, тэчку `zip_codes` дзесьці ў вашай сістэме. Стварыце тэчкі `app`, `config`, `db`, `doc`, `log`, `script`, `spec`. Стварыце тэчку `models` і `controllers` ўнутры тэчкі `app`, а таксама стварыце тэчкі `acceptance`, `factories`, `models` ўнутры тэчкі `spec`. Стварыце файл `config/database.yml` з вашымі наладамі базы дадзеных, вось мая версія:

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

Стварыце файл `script/console`

```ruby
#!/bin/bash

# parameter: RACK_ENV
bundle exec irb -r ./application.rb
```

І зрабіце яго выкананым (для UNIX-падобных сістэм):

    $ chmod +x script/console

Стварыце файл `spec/spec_helper.rb`

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

Звярніце ўвагу, што мы абнавілі канфігурацыю для гема `rspec_api_documentation`. Параметр налады `curl_host` важны, калі ён усталяваны дакументацыя будзе ўтрымліваць `curl` прыклад, які часта выкарыстоўваецца для адладкі вэб-сэрвісаў.

Калі вы выкарыстоўваеце `git` дадайце файл `.gitignore` ўнутр папкі `zip_codes`.

    log/*.log
    doc/*

А таксама вы можаце дадаць пусты файл з імем `.keep` (або` .gitkeep`) ўнутр тэчак `log`, `doc` і `lib/tasks` (мы не стваралі апошнюю тэчку).

Стварыце файл `Gemfile` ўнутры тэчкі `zip_codes`

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

І выканайце

    $ bundle install

Стварыце файл `Rakefile`

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

Стварыце файл `application.rb` ўнутры тэчкі `zip_codes`

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }
```

Вы таксама можаце стварыць файл `.rspec` з наладамі для RSpec

    --color
    --require spec_helper

Радок `--require spec_helper` дазваляе аўтаматычна падключыць файл `spec_helper.rb` ва ўсе тэставыя файлы. Такім чынам, вы можаце не ўказваць відавочна `require "spec_helper"` ў кожным тэставым файле (мы будзем дадаваць гэта ў любым выпадку).

Паглядзіце на структуру створанага прыкладання:

![Basic gem structure](../static/images/zip_codes_structure.png)

## <a name="create-databases"></a>Стварэнне баз дадзеных

Калі вы яшчэ не стварылі `development` і `test` базы дадзеных, выканайце `rake db: create` (у тэрмінале з папкі `zip_codes`). Гэта створыць абедзве базы дадзеных.

## <a name="create-model-and-migration"></a>Стварэнне мадэлі і міграцыі

Мы створым адну мадэль для Zip-кодаў з пяццю атрыбутамі: ZIP, назва вуліцы, нумар будынка, горад, штат. Усе атрыбуты з'яўляюцца радкамі. Атрыбут ZIP з'яўляецца абавязковым (не можа быць пустым) і павінен адпавядаць (правярацца) наступнага рэгулярнаму выразу: `/\A\d{5}(?:-\d{4})?\Z/` (5 лічбаў або 9 лічбаў падзеленых знакам "-" пасля 5-й лічбы).

Стварыце, калі ласка, міграцыю, запусціце наступную `rake` задачу:

    $ rake db:create_migration NAME=create_zip_codes

Абнаўленне кода міграцыі (файл, які быў створаны ў тэчцы `db / migrate`):

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

Выканайце міграцыю для `development` і `test` базы дадзеных

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

Цяпер мы можам стварыць мадэль - файл `app/models/zip_code.rb`

```ruby
class ZipCode < ActiveRecord::Base
  validates :zip, presence: true
  validates_format_of :zip, with: /\A\d{5}(?:-\d{4})?\Z/

  attr_accessible :zip, :street_name, :building_number, :city, :state
end
```

І тэсты для мадэлі - файл `spec/models/zip_code_spec.rb`

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

Вы можаце запусціць тэсты камандай `rspec`.

Стварыце, калі ласка, файл `spec/factories/zip_codes.rb` з завода для стварэння Zip-кодаў у тэстах. Мы будзем выкарыстоўваць яе ў бліжэйшы час у `acceptance` тэстах.

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

Мы выкарыстоўвалі гем [faker](https://github.com/stympy/faker) для генерацыі розных атрыбутаў мадэлі.

## <a name="top-level-interface-planning"></a>Планаванне інтэрфейсу верхняга ўзроўню

Вэб-сэрвіс павінен вяртаць JSON ўяўленне Zip-кода і быць у стане апрацоўваць JSON закадаваныя параметры для стварэння/абнаўлення Zip-кода.

Атрыманне Zip-кода: запыт і адказ.

    $ curl "https://localhost:4567/api/v1/zip_codes/53796.json" -X GET

    {"zip_code":{"id":2,"zip":"53796","street_name":"Johnston Forest",
    "building_number":"463","city":"Mosciskiville","state":"Connecticut",
    "created_at":"2015-02-09T15:20:42.474Z","updated_at":"2015-02-09T15:20:42.474Z"}}

Стварэнне Zip-кода: запыт і адказ.

    $ curl "https://localhost:4567/api/v1/zip_codes.json" \
    $ -X POST \
    $ -H "Content-Type: application/json" \
    $ -d '{"zip_code":{"zip":"31460-3046","street_name":"Cartwright Dale", \
    $ "building_number":"77779","city":"Ovaside","state":"South Dakota"}}'

    {"zip_code":{"id":1,"zip":"31460-3046","street_name":"Cartwright Dale",
    "building_number":"77779","city":"Ovaside","state":"South Dakota",
    "created_at":"2015-02-09T15:20:42.440Z","updated_at":"2015-02-09T15:20:42.440Z"}}

Функцыянал для JSON-серыялізацыі уключаны ў гем `activerecord`. Для большай колькасці магчымасцяў налады, вы можаце выкарыстоўваць [active_model_serializers](https://github.com/rails-api/active_model_serializers) або [JBuilder](https://github.com/rails/jbuilder).

Для разбору JSON з цела `POST` або` PUT` HTTP запыту, мы можам выкарыстоўваць `Rack::PostBodyContentTypeParser` ` middleware` з [rack-contrib](https://github.com/rack/rack-contrib).

Вэб-сэрвіс павінен таксама перадаваць HTTP загаловак `Content-Type: application/json` у кожным адказе.

## <a name="create-controller"></a>Стварэнне кантролера

Дадамо `rack-contrib` ў `Gemfile`. Мы будзем выкарыстоўваць найноўшую версію з GitHub.

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

Зменіце `application.rb` для таго, каб дадаць падтрымку для разбору JSON-цела HTTP запыту з дапамогай `Rack::PostBodyContentTypeParser` `middleware`, пакажыце загаловак HTTP адказу `"Content-Type: application/json"`, а таксама дадайце опцыю канфігурацыі для метаду `ActiveRecord::Base#to_json`.

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

`Rack::PostBodyContentTypeParser` пераўтворыць JSON, што перадаецца ў целе запыту HTTP ў хэш `params`.

Стварыце, калі ласка, файл `app/controllers/zip_codes_controller.rb` з чатырма CRUD-аперацыямі для кіравання Zip-кодамі.

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

І `acceptance` тэсты ў файле `spec/acceptance/zip_codes_spec.rb`

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

Вось і ўсё! (на час). Мы стварылі вэб-сэрвіс, які працуе, але не ўключае ў сябе апрацоўку няслушнага ўводу. Мы зробім гэта ў наступным раздзеле.

## <a name="summary"></a>Рэзюмэ

Мы выкарыстоўвалі метад `to_json` з `activerecord` для JSON-серыялізацыі і `middleware` з [rack-contrib](https://github.com/rack/rack-contrib) для десериализации (разбору) JSON параметраў з POST/PUT HTTP-запытаў.

Мы таксама стварылі `acceptance` тэсты. Амаль кожны тэст правярае правільнасць канкрэтных JSON-атрыбутаў, і мы выкарыстоўваем `JSON.parse` у кожным тэсце. Вы, напэўна, павінны мець дапаможны-метад для гэтага, вось карысная артыкул: [Rails API Testing Best Practices](http://matthewlehner.net/rails-api-testing-guidelines/) (нягледзячы на ​​"Rails" у назве, артыкул утрымлівае інструкцыі, якія могуць быць выкарыстаны ў кожным `rspec` + `rack-test` тэстах).

Некаторыя гемы для тэставання JSON:

* [json_spec](https://github.com/collectiveidea/json_spec)
* [airborne](https://github.com/brooklynDev/airborne)
* [json_expressions](https://github.com/chancancode/json_expressions)
* [match_json](https://github.com/WhitePayments/match_json)

Каб захоўваць выразныя і падтрымоўваныя тэсты, я магу прапанаваць правяраць толькі асноўныя атрыбуты ў прыкладах з адной запісам і правяраць толькі ідэнтыфікатары (IDs) для прыкладаў з калекцыяй запісаў.
