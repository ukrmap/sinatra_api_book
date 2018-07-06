Chapter #6. Validation and HTTP status codes
============================================

Processing of incorrect data input is important aspect of the functioning of the web service as well as error messages for the client. [HTTP status codes](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) - this is a common simple concept to notify the client about the status of processing the HTTP request. Error message in JSON complements HTTP-response and clarifies error for end users.

Here is a list of HTTP status codes, which we'll use:

* 200, 201 - Success: OK and Created (201 - for `POST` requests)
* 404      - Not Found
* 400, 422 - Bad Request, Unprocessable Entity
* 401, 403 - Unauthorized, Forbidden
* 500      - Internal Server Error

400 and 422 is similar, we will use 400 to notify about invalid request parameters in general (invalid type or value out of range) and 422 for validation errors.

401 and 403 also similar. We will use 401 if user sends incorrect or expired token for authentication. And 403 to notify authenticated user or public user about authorization error.

## <a name="500-internal-server-error"></a>500 - Internal Server Error

With current web-service implementation if unexpected error occurs service responds with 500 error with no response body. Developing JSON-service, we must provide notice to the client with an appropriate error message in the format of JSON (in file `application.rb`).

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

use Rack::PostBodyContentTypeParser
before { content_type :json }
ActiveRecord::Base.include_root_in_json = true

# Respond with error message at unexpected exception (HTTP 500 status code will be added by sinatra).
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

## <a name="404-not-found"></a>404 - Not Found

If client provides ID of Zip-code that does not exist in the database, ActiveRecord `find` method raises exception `ActiveRecord::RecordNotFound` error and web-service returns 500 HTTP status code. This is not good. Service should respond with 404 status code and appropriate error message.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

use Rack::PostBodyContentTypeParser
before { content_type :json }
ActiveRecord::Base.include_root_in_json = true

# Catch properly ActiveRecord::RecordNotFound
error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }

# Respond with error message at unexpected exception (HTTP 500 status code will be added by sinatra).
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

## <a name="422-unprocessable-entity"></a>422 - Unprocessable Entity

If we will try to save model with invalid data (e.g. wrong format zip) ActiveRecord will raise `ActiveRecord::RecordInvalid` error. In this case service should respond with 422 HTTP status code and with specific validation error messages.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

use Rack::PostBodyContentTypeParser
before { content_type :json }
ActiveRecord::Base.include_root_in_json = true

# Catch properly ActiveRecord::RecordNotFound
error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }

# Catch validation errors
error(ActiveRecord::RecordInvalid) do
  [422, { message: "Validation errors occurred",
          errors:  env['sinatra.error'].record.errors.messages }.to_json ]
end

# Respond with error message at unexpected exception (HTTP 500 status code will be added by sinatra).
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

## <a name="tests"></a>Tests

We can test error handling in `acceptance` tests. File `spec/acceptance/zip_codes_spec.rb`

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

    example "Create Zip Code with invalid params", document: false do
      do_request(zip_code: { zip: "1234" })

      expect(status).to eq 422
      expect(response_body).to eq '{"message":"Validation errors occurred","errors":{"zip":["is invalid"]}}'
      expect(new_zip_code).to be_nil
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

    example "Read Zip Code that does not exist", document: false do
      do_request(zip: '12345-6789')

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
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

    example "Update Zip Code that does not exist", document: false do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
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

    example "Delete Zip Code that does not exist", document: false do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end
  end
end
```

## <a name="400-bad-request"></a>400 - Bad Request

Sometimes we need to ensure correctness of parameter types provided by client. We can prevent interaction with database on incorrect input data and immediately return error message to client. We will use gem [sinatra-param](https://github.com/mattt/sinatra-param) for parameter validation and type coercion in `sinatra`.

Please add line `gem 'sinatra-param'` into `Gemfile`

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra', require: 'sinatra/main'
gem 'rack-contrib', git: 'https://github.com/rack/rack-contrib'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'
# Parameter Validation & Type Coercion for Sinatra
gem 'sinatra-param'

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

And run `bundle install`

If client provides parameter `zip_code` of wrong type (not `Hash`) web-service returns 500 error (e.g. ``NoMethodError: undefined method `stringify_keys' for "STRING":String`` for type `String`). We can prevent this behaviour with next declarative code in file `app/controllers/zip_codes_controller.rb`.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true # ensure params[:zip_code] is a Hash
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
  param :zip_code, Hash, required: true # ensure params[:zip_code] is a Hash
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

We can go further and validate with regular expression parameter `zip` in `get` route. This prevents database query if parameter has invalid format. And web-service returns HTTP status code 400 instead of 404, as well as a more appropriate error message (`"Parameter must match format"` instead `"Record not found"`, which provides that a record may have been deleted).

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes/:zip.json" do
  param :zip, String, format: /\A\d{5}(?:-\d{4})?\Z/ # route logic stops here if zip has wrong format
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

I have limitation for integer type in database. If client provides ID more than `2147483647` (which in the binary system equals `111111111111111111111111111111`) `activerecord` returns `PostgreSQL` error and web-service responds with 500 HTTP error. It is possible to prevent this with following parameter filtering.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes/:zip.json" do
  param :zip, String, format: /\A\d{5}(?:-\d{4})?\Z/
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647 # 0b111111111111111111111111111111
  param :zip_code, Hash, required: true
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647 # 0b111111111111111111111111111111
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

Updated tests (file `spec/acceptance/zip_codes_spec.rb`)

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

    example "Create Zip Code with invalid params", document: false do
      do_request(zip_code: { zip: "1234" })

      expect(status).to eq 422
      expect(response_body).to eq '{"message":"Validation errors occurred","errors":{"zip":["is invalid"]}}'
      expect(new_zip_code).to be_nil
    end

    example "Create Zip Code provide not Hash zip_code params", document: false do
      do_request(zip_code: "STRING")

      expect(status).to eq 422
    end

    example "Create Zip Code do not provide zip_code params", document: false do
      do_request

      expect(status).to eq 400
      expect(response_body).to eq '{"message":"Invalid Parameter: zip_code","errors":{"zip_code":"Parameter is required"}}'
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

    example "Read Zip Code that does not exist", document: false do
      do_request(zip: '12345-6789')

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Read Zip Code provide invalid format zip", document: false do
      do_request(zip: '1234')
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: zip'
      expect(json_response[:errors][:zip]).to eq 'Parameter must match format (?-mix:\A\d{5}(?:-\d{4})?\Z)'
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

    example "Update Zip Code that does not exist", document: false do
      do_request(id: 800, zip_code: valid_attributes)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Update Zip Code provide to big ID number", document: false do
      do_request(id: 3000000000, zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: id'
      expect(json_response[:errors][:id]).to eq 'Parameter cannot be greater than 2147483647'
    end

    example "Update Zip Code provide not Hash zip_code params", document: false do
      do_request(id: zip_code.id, zip_code: "STRING")

      expect(status).to eq 200
    end

    example "Update Zip Code do not provide zip_code params", document: false do
      do_request(id: zip_code.id)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: zip_code'
      expect(json_response[:errors][:zip_code]).to eq 'Parameter is required'
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

    example "Delete Zip Code that does not exist", document: false do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Delete Zip Code provide to big ID number", document: false do
      do_request(id: 3000000000)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: id'
      expect(json_response[:errors][:id]).to eq 'Parameter cannot be greater than 2147483647'
    end
  end
end
```

## <a name="summary"></a>Summary

We have envisaged error handling and appropriate responses with HTTP status codes with error messages in JSON format (we have used error handling ability of `sinatra`). We used gem [sinatra-param](https://github.com/mattt/sinatra-param) for parameter validation and type coercion in `sinatra`.
