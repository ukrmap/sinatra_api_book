Chapter #1. Outline
===================

Let's build a simple Web service for managing notes to get more familiar with some technologies that are used is this book and of course to understand partial concepts of Web services.

We will keep notes in sqlite database and allow access and manage those notes via http using incipient just now new notes service. There are some choices of technologies for build service like `rails-api`, `sinatra`, `grape` or combination of this. For all services in this book we will use `sinatra`, generally this is matter of taste, `sinatra` is concise and fits well.

## <a name="infrastructure"></a>Infrastructure

Please create `notes` folder in which will store our service's code files. We need install three ruby gems for managing service internals, those are `sinatra`, `sqlite3` and `activerecord` and two gems for testing: `rspec` and `rack-test`.
Create file named `Gemfile` in `notes` folder with next content:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'sqlite3'
gem 'activerecord'
gem 'rspec'
gem 'rack-test'
```

Then in terminal navigate to `notes` folder and run `bundle install` (ruby, rubygems and gem `bundler` should be installed for this). This installs all of the above gems and also creates file `Gemfile.lock` with used versions of gems. You should see something similar.

    $ bundle install
    Fetching gem metadata from https://rubygems.org/.........
    Installing i18n (0.7.0)
    Installing json (1.8.2)
    Installing minitest (5.5.1)
    Using thread_safe (0.3.4)
    Installing tzinfo (1.2.2)
    Installing activesupport (4.2.0)
    Using builder (3.2.2)
    Installing activemodel (4.2.0)
    Installing arel (6.0.0)
    Installing activerecord (4.2.0)
    Using diff-lcs (1.2.5)
    Installing rack (1.6.0)
    Installing rack-protection (1.5.3)
    Installing rack-test (0.6.3)
    Installing rspec-support (3.1.2)
    Installing rspec-core (3.1.7)
    Installing rspec-expectations (3.1.2)
    Installing rspec-mocks (3.1.3)
    Installing rspec (3.1.0)
    Using tilt (1.4.1)
    Installing sinatra (1.4.5)
    Installing sqlite3 (1.3.10)
    Using bundler (1.5.3)
    Your bundle is complete!
    Use `bundle show [gemname]` to see where a bundled gem is installed.

Installations part is over. Now we can start developing our service. Please create file `service.rb` in `notes` folder and require three needed gems.

```ruby
require "sqlite3"
require "active_record"
require "sinatra/main"
```

Note that gem is called `activerecord` but we have just required `active_record`. And also only part of sinatra but this is not really important.

## <a name="create-model"></a>Create model

Then we need to create a database table to sore the notes. Add this please in `service.rb`.

```ruby
class CreateNotes < ActiveRecord::Migration
  def change
    create_table :notes do |t|
      t.string :content, null: false, default: 'Empty'
    end
  end
end
```

Table is called `notes` (not surprisingly) and has only one field - content of type text with default value "Empty".

We need to create corresponding ORM class for managing notes records. This is simple - just inherit it from `ActiveRecord::Base` and class will find `notes` table by its own name (downcase and pluralize word "Note").

```ruby
class Note < ActiveRecord::Base
end
```

Establish connection.

```ruby
ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: 'db/notes.sqlite3')
```

Create `notes` table if it is not exist - run migration.

```ruby
CreateNotes.new.change unless ActiveRecord::Base.connection.table_exists? :notes
```

That was set up infrastructure code.

## <a name="routes-for-crud"></a>Routes for CRUD

Any action that may be expected from your application can be performed with the one of the four operations: `C`raate an entity, `R`ead entity, `U`pdate entity and `D`elete entity (`CRUD`). This actions are coupled with four (or about four) HTTP verbs: POST, GET, PUT (can be PATCH), DELETE. URL of the entilty (or representation) is expected to be somehow associated with resource name which is "notes" in our case. This is genial simplicity of the `REST` principles.

We also adding prefix "api" to URLs that may be helpful for users to immediately see that this URLs are part of some API. Also "v1" prefix that can be useful if you plan to maintain several versions of API in future. URLs will end with format that service supports. Let's notes service will support text format and URLs will end with ".txt".

We will write code for this four `CRUD` operations is same order one by one. The first - is `C`reate.

```ruby
post "/api/v1/notes.txt" do
  content_type :txt
  note = Note.create(content: params[:content])
  status 201
  "##{note.id} #{note.content}"
end
```

`sinatra's` method `post` creates a handler for processing the HTTP request sent by the POST method to a provided URL - "/api/v1/notes.txt". `content_type :txt` adds HTTP header that notifies client about response format. After that record is created (saved in database), expected that user provides parameter named `content` (attribute will be set to "Empty" if user omits it). HTTP status code is 201 which means that a new resource being created. Service returns text representation of new record.

It is assumed that the service always returns data in text format, so it makes sense to move the string `content_type: txt` from code block belonging to a specific proute to the `before` filter that is used for all routes.

```ruby
before do
  content_type :txt
end
```

By analogy to the method `post` for creating record we are adding method `get` for retrieving (`R`ead) record(s). One for reading all records and one for reading specific record found by its ID.

```ruby
get "/api/v1/notes.txt" do
  Note.all.map { |note| "##{note.id} #{note.content}" }.join("\n")
end

get "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  "##{note.id} #{note.content}"
end
```

By default routes respond with success 200 HTTP status code, so explicit setting can be omitted.

This how our service should look at this moment (file `service.rb` at `notes` directory).

```ruby
require "sqlite3"
require "active_record"
require "sinatra/main"

class CreateNotes < ActiveRecord::Migration
  def change
    create_table :notes do |t|
      t.string :content, null: false, default: 'Empty'
    end
  end
end

class Note < ActiveRecord::Base
end

ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: 'db/notes.sqlite3')
CreateNotes.new.change unless ActiveRecord::Base.connection.table_exists? :notes

before do
  content_type :txt
end

post "/api/v1/notes.txt" do
  note = Note.create(content: params[:content])
  status 201
  "##{note.id} #{note.content}"
end

get "/api/v1/notes.txt" do
  Note.all.map { |note| "##{note.id} #{note.content}" }.join("\n")
end

get "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  "##{note.id} #{note.content}"
end
```

## <a name="test-it-manually"></a>Test it manually

We can run our service

    $ ruby service.rb

And test that it works (or start use it) with `curl` command line tool. Create new notes with content parameter.

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=First%20Note"
    #1 First Note

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=Second%20Note"
    #2 Second Note

We used `%20` for URL encoded space char (blank).

Retrieve all notes

    $ curl -X GET "localhost:4567/api/v1/notes.txt"
    #1 First Note
    #2 Second Note

Retrieve specific note by it's ID

    $ curl -X GET "localhost:4567/api/v1/notes/1.txt"
    #1 First Note

You can also use your browser for retrieving records.

For completeness of functionality we still need to create `U`pdate and `D`elete operations. Methods `put` and `delete` respectively.

```ruby
put "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  note.update_attributes!(content: params[:content])
  "##{note.id} #{note.content}"
end

delete "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  note.destroy
end
```

We can test that both of them work. You may need to restart the server (stopped by pressing `Ctrl` + `C` and run `ruby service.rb` again).

Update existing note

    $ curl -X PUT "localhost:4567/api/v1/notes/1.txt?content=New%20Content"
    #1 New Content

Delete note

    $ curl -X DELETE "localhost:4567/api/v1/notes/1.txt"

Everything works. That is great. We should also write tests. Sometimes programmers do this before writing main application code - write one test that falls due to missing functionality and then add this functionality to make test passing. This is called "red-green" cycle. But we are adding all tests at once.

## <a name="automated-tests"></a>Automated tests

Create please directory called `spec` inside notes directory and file `service_spec.rb` in it with next content:

```ruby
require_relative "../service"
require "rack/test"

RSpec.configure do |config|
  config.include Rack::Test::Methods
  config.after(:each) { Note.delete_all }

  def app
    Sinatra::Application
  end
end

describe "Notes Service" do
  describe "POST /api/v1/notes.txt" do
    let(:note) { Note.last }

    it "craetes new note" do
      post "/api/v1/notes.txt", content: "My New Note"
      expect(last_response.status).to eq 201
      expect(last_response.body).to eq "##{note.id} My New Note"
    end
  end

  describe "GET /api/v1/notes.txt" do
    before { Note.create([{ id: 1, content: "First Note" }, { id: 2, content: "Second Note" }]) }

    it "retrieves all notes" do
      get "/api/v1/notes.txt"
      expect(last_response).to be_ok
      expect(last_response.body).to eq "#1 First Note\n#2 Second Note"
    end
  end

  describe "GET /api/v1/notes/:id.txt" do
    before { Note.create([{ id: 4, content: "My Note" }]) }

    it "retrieves specific note" do
      get "/api/v1/notes/4.txt"
      expect(last_response).to be_ok
      expect(last_response.body).to eq "#4 My Note"
    end
  end

  describe "PUT /api/v1/notes/:id.txt" do
    before { Note.create([{ id: 4, content: "My Note" }]) }

    it "updates note and returns updated content" do
      put "/api/v1/notes/4.txt", content: "New Content"
      expect(last_response).to be_ok
      expect(last_response.body).to eq "#4 New Content"
      expect(Note.last.content).to eq "New Content"
    end
  end

  describe "DELETE /api/v1/notes/:id.txt" do
    before { Note.create([{ id: 4, content: "My Note" }]) }

    it "deletes note" do
      delete "/api/v1/notes/4.txt"
      expect(last_response).to be_ok
      expect(Note.last).to be_nil
    end
  end
end
```

There are more than one expectation per one test. In fact, someone may think it is a bad practice, but I do so. Besides `rspec` we are using `rack-test` for testing `rack` based applications (which `sinatra` is). Navigate to folder `notes` in the terminal and run `rspec --color --format=doc`.

    rspec --color --format=doc

    Notes Service
      POST /api/v1/notes.txt
        craetes new note
      GET /api/v1/notes.txt
        retrieves all notes
      GET /api/v1/notes/:id.txt
        retrieves specific note
      PUT /api/v1/notes/:id.txt
        updates note and returns updated content
      DELETE /api/v1/notes/:id.txt
        deletes note

    Finished in 0.11771 seconds (files took 0.80795 seconds to load)
    5 examples, 0 failures

So everything is working and tested. A basic understanding of what is an HTTP service or API you already should have. You probably knew this stuff before. But maybe you need a fresh look. Generally speaking, "the repetition - the mother of learning". So let's build this same service again! I am kidding. Stupid joke, I know. Ok let's build another one application that can be regarded as HTTP API, but in the next chapter.

## <a name="summary"></a>Summary

We have created the first Web service in this book. We used `sinatra` for building service and `rspec` with `rack-test` to create automated tests. If you plan to use `sinatra` I encourage you to read book [Sinatra: Up and Running](http://shop.oreilly.com/product/0636920019664.do) and visit site [www.sinatrarb.com](http://www.sinatrarb.com/).
