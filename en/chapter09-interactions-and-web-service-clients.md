Chapter #9. Interactions and Web Service Clients
================================================

Zip codes service is almost ready to go to production. It can be used by any application that can perform HTTP request. This application thus becoming a Web service client. JavaScript applications running in Web browser naturally fits in this architecture. Ruby application also can act like a Web service client. Any other Web service can use other web service internally. Zip codes service already uses users service for authentication.

In this chapter we will create ruby client for Zip codes service. But first we will do small changes in Zip codes service.

## <a name="rest-conventions"></a>REST Conventions

We have built Zip code service by REST conventions, we use HTTP verbs to define CRUD method and use URL to define resource. But we use different URLs for update and retrieve resource - on update Zip code is searched by ID (route `"/api/v1/zip_codes/:id.json"`), on retrieve - by zip attribute (route `"/api/v1/zip_codes/:zip.json"`). It was done for learning purposes and now we should follow the conventions, we will change route for retrieving single zip code in file `app/controllers/zip_codes_controller.rb`.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  authorize! :create, zip_code
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes.to_json
end

# Route was changed to the same like in Update and Delete actions
get "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  zip_code = ZipCode.find(params[:id])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  param :zip_code, Hash, required: true
  load_and_authorize! ZipCode
  @zip_code.update_attributes!(params[:zip_code]) if params[:zip_code].any?
  @zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  load_and_authorize! ZipCode
  @zip_code.destroy!
end
```

And fix acceptance tests for retrieve (get) action in file `spec/acceptance/zip_codes_spec.rb`

```ruby
require "spec_helper"

resource 'ZipCode' do

  # tests for create (post) action

  get "/api/v1/zip_codes/:id.json" do
    parameter :id, "Record ID", required: true

    let(:zip_code) { create(:zip_code) }

    context "Public User" do
      example "Read Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end

      example "Read Zip Code that does not exist", document: false do
        do_request(id: 8889)

        expect(status).to eq 404
        expect(response_body).to eq '{"message":"Record not found"}'
      end

      example "Read Zip Code provide invalid format zip id", document: false do
        do_request(id: 's1234')
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 400
        expect(json_response[:message]).to eq 'Invalid Parameter: id'
        expect(json_response[:errors][:id]).to eq "'s1234' is not a valid Integer"
      end
    end

    context 'Regular User (authenticated user with type "RegularUser")', document: false do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"RegularUser"}}') }

      example "Read Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end
    end

    context "Admin User", document: false do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"AdminUser"}}') }

      example "Read Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end
    end
  end

  # tests for update (put) and delete actions

end
```

## <a name="curl"></a>Curl

Let's quick check with `curl` that Zip codes service works as expected. `Curl` is also Web service client. Please run users service on port 4545 (run `ruby service.rb -p 4545` in terminal from users service dir) and Zip codes service on port 4567 (run `ruby application.rb` form `zip_codes` dir). We are ready for checking

Get Zip codes list

    $ curl "http://localhost:4567/api/v1/zip_codes.json" \
      -X GET

    [{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming","created_at":"2015-02-15T09:02:25.383Z","updated_at":"2015-02-15T09:02:25.383Z"}},{"zip_code":{"id":403,"zip":"98189-4795","street_name":"Lucie Falls"...

Get single Zip code

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X GET

    {"zip_code":{"id":401,"zip":"63109","street_name":"Candido Loop","building_number":"897","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T10:48:59.680Z"}}

Update few attributes (`street_name` and `building_number`) of Zip code

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X PUT \
      -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
      -H "Content-Type: application/json" \
      -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T13:19:52.762Z"}}

Everything works like expected, let's create Zip codes service `ruby` client.

Add this line to `zip_codes/application.rb` if you want test error handling in development mode

```ruby
disable :show_exceptions # in production it is false, so you probably do not need it
```

## <a name="web-service-client"></a>Web Service Client

We will create client in `irb` session using gem `activeresource`. Install gem `activeresource`

    $ gem install activeresource

Start `irb` session

    $ irb

```ruby
require "active_resource"

class ZipCode < ActiveResource::Base
  self.format = :json
  self.include_root_in_json = true
  self.site = "http://localhost:4567"
  self.prefix = "/api/v1/"
end
```

This is all we needed - work is done, now we can check in same `irb` session

Get Zip code from `http://localhost:4567/api/v1/zip_codes/401.json`

```ruby
zip_code = ZipCode.find(401)
zip_code.zip # => "63109"
zip_code.street_name # => "Candido Loop"
zip_code.building_number # => "897"
zip_code.city # => "New Hoyt"
zip_code.state # => "Utah"
```

Get Zip codes list from `http://localhost:4567/api/v1/zip_codes.json`

```ruby
zip_codes = ZipCode.find(:all)
zip_codes = ZipCode.find(:all, params: { state_eq: "Massachusetts" })
zip_codes = ZipCode.find(:all, params: { zip_start: "119" })
```

Request list, but return only one

```ruby
zip_codes = ZipCode.find(:first, params: { zip_start: "119" })
```

Update Zip code (by admin).

```ruby
ZipCode.headers["Authorization"] = "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
zip_code.city = "New Johnathan"
zip_code.save
```

Note adding "Authorization" HTTP header.

Delete Zip code

```ruby
zip_code = ZipCode.find(101186)
zip_code.destroy
```

Create Zip Code

```ruby
zip_code = ZipCode.create(zip: "82470-2132", street_name: "Micheal Street",
  building_number: "911", city: "South Madalyn", state: "Louisiana")

zip_code.id # => 101187
```

## <a name="cors"></a>CORS Support

If the application is running in a web browser must have access to a web service, we need to allow Cross-Origin Resource Sharing (CORS). Otherwise, only the applications that are running on the same URL as the web service will be able to access it.

We can use `Rack Middleware` for this.

Add gem `rack-cors` into `Gemfile` and run `bundle install`.

```ruby
gem 'rack-cors', :require => 'rack/cors'
```

Add next configuration code into file `application.rb`

```ruby
use Rack::Cors do
  allow do
    origins '*'
    resource '/*', headers: :any, methods: [:post, :put, :get, :delete, :options]
  end
end
```

After this site with any URL can access the web-service.

## <a name="summary"></a>Summary

We have created a client for Zip codes services using [activeresource](https://github.com/rails/activeresource).
