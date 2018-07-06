Chapter #5. JSON Web Services
=============================

Web services can use different formats for data serialization. The most common are XML and JSON. JSON - simple format, which is also perfect for JavaScript-applications that run in a web browser. One limitation of using JSON is the lack of built-in ability to specify data types (such as `String`, `Integer`, `Array`). The client must independently determine the data type. For most cases it is obvious and is handled by JSON parser.

Sample data in JSON format:

    {"name":"Lyric Adams","account":208.6,"orders":[456,803,1204],"address":"871 Tommie Roads","city":"Halchester","birthday":"1986-05-07","registeration_date":"2015-02-23"}

The same data is indented:

    {
      "name": "Lyric Adams",
      "account": 208.6,
      "orders": [456,803,1204],
      "address": "871 Tommie Roads",
      "city": "Halchester",
      "birthday": "1986-05-07",
      "registeration_date": "2015-02-23"
    }

The data presented here about the customer in the form of key-value store. JSON parser is able to determine the types of certain attributes: `"Lyric Adams"` - string, `208.6` - float number, `[456,803,1204]` - array of integers. Attributes `"birthday"` and `"registeration_date"` contain string values. Data types for date and time represented as a string in JSON.

There are at least two ways to convert all of the strings with the date into date data type: check all values that match date-time regular expression and convert them or convert only values of predefined list of keys (you must know in advance from somewhere all the keys that hold date-time values).

The next few chapters are devoted to the development of a web service to manage Zip-codes (postal codes of the United States). In this chapter we will create service that serves data in JSON format and handles JSON data from requests.

## <a name="create-zip-codes-service-structure"></a>Create structure of Zip-codes service

OK, create please folder `zip_codes` somewhere in your system. Create folders `app`, `config`, `db`, `doc`, `log`, `script`, `spec`. Create folders `models` and `controllers` inside folder `app` and create folders `acceptance`, `factories`, `models` inside folder `spec`. Create file `config/database.yml` with your database settings, here is my version:

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

Create file `script/console`

```ruby
#!/bin/bash

# parameter: RACK_ENV
bundle exec irb -r ./application.rb
```

And make it executable (for UNIX-like systems):

    $ chmod +x script/console

Create file `spec/spec_helper.rb`

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
  config.api_name = "Zip Codes API"
  config.format = :html
  config.curl_host = 'https://zipcodes.example.com'
  config.curl_headers_to_filter = %w(Host Cookie)
end
```

Note that we have updated configurations for gem `rspec_api_documentation`. Configuration option `curl_host` is important, if it is set documentation will contain `curl` example which is often useful for debugging web-services.

If you are using `git` add file `.gitignore` inside folder `zip_codes`.

    log/*.log
    doc/*

And also you can add empty file named `.keep` (or `.gitkeep`) inside folders `log`, `doc`, and `lib/tasks` (we did not create last folder).

Create file `Gemfile` inside folder `zip_codes`

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

And run

    $ bundle install

Create file `Rakefile`

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

Create file `application.rb` inside folder `zip_codes`

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }
```

You can also create file `.rspec` with configs for RSpec

    --color
    --require spec_helper

Line `--require spec_helper` allows automatically require file `spec_helper.rb` in all test files. So you can omit explicit `require "spec_helper"` in each spec file (we will be adding it anyway).

Look at created service structure:

![Basic gem structure](../static/images/zip_codes_structure.png)

## <a name="create-databases"></a>Create databases

If you have not yet created `development` and `test` databases run `rake db:create` (in terminal from folder `zip_codes`). This will create both databases.

## <a name="create-model-and-migration"></a>Create model and migration

We will create one model for Zip-codes with five attributes: zip, street name, building number, city, state. All attributes are strings. Attribute zip is mandatory (cannot be empty) and should be validated with next regular expression: `/\A\d{5}(?:-\d{4})?\Z/` (5 digits or 9 digits separated by "-" after the 5th digit).

Create please migration, run next `rake` task:

    $ rake db:create_migration NAME=create_zip_codes

Update migration code (file that was created in folder `db/migrate`):

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

Run migration for `development` and `test` databases

    $ rake db:migrate
    $ RACK_ENV=test rake db:migrate

Now we can create model - file `app/models/zip_code.rb`

```ruby
class ZipCode < ActiveRecord::Base
  validates :zip, presence: true
  validates_format_of :zip, with: /\A\d{5}(?:-\d{4})?\Z/

  attr_accessible :zip, :street_name, :building_number, :city, :state
end
```

And tests for model - file `spec/models/zip_code_spec.rb`

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

You can run tests with `rspec` command.

Create please file `spec/factories/zip_codes.rb` with factory for creating Zip-codes in tests. We will use it shortly in `acceptance` tests.

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

We have used gem [faker](https://github.com/stympy/faker) to generate the various attributes of the model.

## <a name="top-level-interface-planning"></a>Top-level interface planning

Web service should return JSON representation of Zip-code and be able to handle JSON encoded parameters for creating/updating Zip-code.

Retrieve Zip-code: request and response.

    $ curl "https://localhost:4567/api/v1/zip_codes/53796.json" -X GET

    {"zip_code":{"id":2,"zip":"53796","street_name":"Johnston Forest",
    "building_number":"463","city":"Mosciskiville","state":"Connecticut",
    "created_at":"2015-02-09T15:20:42.474Z","updated_at":"2015-02-09T15:20:42.474Z"}}

Create Zip-code: request and response.

    $ curl "https://localhost:4567/api/v1/zip_codes.json" \
    $ -X POST \
    $ -H "Content-Type: application/json" \
    $ -d '{"zip_code":{"zip":"31460-3046","street_name":"Cartwright Dale", \
    $ "building_number":"77779","city":"Ovaside","state":"South Dakota"}}'

    {"zip_code":{"id":1,"zip":"31460-3046","street_name":"Cartwright Dale",
    "building_number":"77779","city":"Ovaside","state":"South Dakota",
    "created_at":"2015-02-09T15:20:42.440Z","updated_at":"2015-02-09T15:20:42.440Z"}}

Functional for JSON-serialization is included in gem `activerecord`. For a more configurable options, you can use [active_model_serializers](https://github.com/rails-api/active_model_serializers) or [JBuilder](https://github.com/rails/jbuilder).

For parse JSON from the body of `POST` or `PUT` HTTP request we may use `Rack::PostBodyContentTypeParser` `middleware` from [rack-contrib](https://github.com/rack/rack-contrib).

Service should also provide HTTP header `Content-Type: application/json` in each response.

## <a name="create-controller"></a>Create controller

Add `rack-contrib` in `Gemfile`. We will use newest version from github.

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

Amend `application.rb` to add support for JSON request bodies with `Rack::PostBodyContentTypeParser` `middleware`, specify HTTP response header `"Content-Type: application/json"`, and add configuration option for method `ActiveRecord::Base#to_json`.

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

# Adds Content-Type: application/json HTTP header in response
before { content_type :json }

# Configure method ActiveRecord::Base#to_json to add root node at top level,
# e.g. {"zip_code":{"zip": ... }}
ActiveRecord::Base.include_root_in_json = true
```

`Rack::PostBodyContentTypeParser` converts JSON that transferred in HTTP request body into hash `params`.

Create please file `app/controllers/zip_codes_controller.rb` with four CRUD-actions for managing Zip-codes.

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

And `acceptance` tests in file `spec/acceptance/zip_codes_spec.rb`

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

That is it! (for a while). We created web service that works but does not include the processing of incorrect input. We will do this in next chapter.

## <a name="summary"></a>Summary

We used method `to_json` of `activerecord` for JSON-serialization and `middleware` from [rack-contrib](https://github.com/rack/rack-contrib) for deserialization (parsing) JSON params from POST/PUT HTTP requests.

We also created `acceptance` tests. Almost each test checks correctness of specific JSON-attributes and we use `JSON.parse` in every test. You probably should have helper-method for this, here is useful article: [Rails API Testing Best Practices](http://matthewlehner.net/rails-api-testing-guidelines/) (despite of "Rails" in title, article contains instructions that can be used in every `rspec` + `rack-test` tests).

Some gems for JSON testing:

* [json_spec](https://github.com/collectiveidea/json_spec)
* [airborne](https://github.com/brooklynDev/airborne)
* [json_expressions](https://github.com/chancancode/json_expressions)
* [match_json](https://github.com/WhitePayments/match_json)

To keep clear and maintainable tests, I can suggest check only main attributes in examples with single record and check only IDs for examples with collection of records.
