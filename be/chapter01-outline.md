Кіраўнік # 1. Накіды
====================

Давайце створым просты вэб-сэрвіс для кіравання нататкамі (notes), каб азнаёміцца ​​з некаторымі тэхналогіямі, якія выкарыстоўваюцца ў гэтай кнізе і, вядома, умацаваць разуменне канцэпцый пабудовы вэб-сэрвісаў.

Мы будзем захоўваць нататкі ў SQLite базе дадзеных і прадастаўляць доступ да нататак праз HTTP з выкарыстаннем notes API. Ёсць некаторыя варыянты тэхналогій для стварэння сэрвісаў, такія як `rails-api`,` sinatra`, `grape` або іх камбінацыі. Для ўсіх сэрвісаў у гэтай кнізе мы будзем выкарыстоўваць `sinatra`, як правіла, гэта справа густу,` sinatra` лаканічны фреймворк і добра ўпісваецца ў патрабаванні.

## <a name="infrastructure"></a>Інфраструктура

Калі ласка, стварыце тэчку `notes`, у якой будзе захоўвацца файлы кода нашага сэрвісу. Мы павінны ўсталяваць тры ruby гема для ўнутранага абслугоўвання сэрвісу, а менавіта: `sinatra`,` sqlite3` і `activerecord` і два гема для тэставання: `rspec` і `rack-test`.
Стварыце файл з імем `Gemfile` ў тэчцы `notes` з наступнымі змесцівам:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'sqlite3'
gem 'activerecord'
gem 'rspec'
gem 'rack-test'
```

Затым у тэрмінале перайдзіце ў тэчку `notes` і выканайце `bundle install` (ruby, rubygems і гем `bundler` павінны быць устаноўлены да гэтага). Гэта ўсталюе ўсе вышэйпералічаныя гемы, а таксама створыць файл `Gemfile.lock` з версіямі выкарыстаных гемов. Вы павінны ўбачыць нешта падобнае.

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

Усталявальная частка завершана. Цяпер мы можам прыступіць да распрацоўкі нашага сэрвісу. Калі ласка, стварыце файл `service.rb` ў тэчцы `notes` з падключэннем трох неабходных гемов.

```ruby
require "sqlite3"
require "active_record"
require "sinatra/main"
```

Звярніце ўвагу, што гем называецца `activerecord` але мы падлучальны `active_record`. А таксама толькі частка sinatra, але гэта не вельмі важна.

## <a name="create-model"></a>Стварэнне мадэлі

Затым нам трэба стварыць табліцу notes ў базе дадзеных. Дадайце наступны код у `service.rb`.

```ruby
class CreateNotes < ActiveRecord::Migration
  def change
    create_table :notes do |t|
      t.string :content, null: false, default: 'Empty'
    end
  end
end
```

Табліца называецца `notes` (не дзіўна) і мае толькі адно поле - content тыпу string з значэннем па змаўчанні "Empty" ("Пуста ").

Мы павінны стварыць адпаведны клас ORM для кіравання запісамі табліцы notes. Гэта проста - успадкоўваючы клас ад `ActiveRecord::Base` і называем яго як табліцу `notes` толькі ў адзіным ліку (клас зможа вызначыць імя табліцы ад свайго імя).

```ruby
class Note < ActiveRecord::Base
end
```

Ўсталяванне злучэння.

```ruby
ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: 'db/notes.sqlite3')
```

Стварэнне табліцы `notes`, калі яна яшчэ не існуе - выкананне міграцыі.

```ruby
CreateNotes.new.change unless ActiveRecord::Base.connection.table_exists? :notes
```

Гэта быў код для стварэння інфраструктуры.

## <a name="routes-for-crud"></a>Маршруты (роуты, пути) для CRUD

Любое дзеянне, якое можна чакаць ад вашага прыкладання можа быць выканана пры дапамозе адной з чатырох аперацый: стварэнне (`C`raate) сутнасці, атрыманне (`R`ead) сутнасці, абнаўленне (`U`pdate) сутнасці і выдаленне (`D`elete) сутнасці (`CRUD`). Гэтыя дзеянні звязаныя з чатырма (або каля чатырох) HTTP загалоўкаў: POST, GET, PUT (можа быць PATCH), DELETE. URL рэсурсу гэтак жа павінен быць неяк звязаны з імем рэсурсу - у нашым выпадку "notes". Гэта геніяльная прастата прынцыпаў `REST`.

Мы таксама дадаем прэфікс "api" да URL адрасах, мяркуючы па якім карыстальнікам адразу відаць, што гэты URL-адрасы з'яўляюцца часткай нейкага API. Акрамя таго, прэфікс "v1", які можа быць карысны, калі вы плануеце падтрымліваць некалькі версій API ў будучыні. Заканчэнне URL-адрасы ёсць пазначэннем фармату, які падтрымлівае сэрвіс. Няхай наш сэрвіс будзе падтрымліваць просты тэкставы фармат ("text / plain") і URL-адрасы будуць сканчацца на ".txt".

Мы будзем дадаваць код для гэтых чатырох `CRUD` аперацый у тым жа парадку адзін за адным. Першы - гэта стварэнне (`C`reate).

```ruby
post "/api/v1/notes.txt" do
  content_type :txt
  note = Note.create(content: params[:content])
  status 201
  "##{note.id} #{note.content}"
end
```

Метад `post` гема `sinatra` стварае апрацоўшчык HTTP запыту пасланага метадам POST у прадастаўлены URL - "/api/v1/notes.txt". `Content_type: txt` дадае HTTP загаловак, які паведамляе кліента аб фармаце адказу. Пасля гэтага ствараецца запіс (захаванне ў базе даных), чакаецца, што карыстач падае параметр з імем `content` (інакш значэнне атрыбуту будзе ўстаноўлена ў значэнне па змаўчанні - радок "Empty"). Код стану HTTP 201, што азначае, што новы рэсурс ствараецца. Сэрвіс вяртае тэкставае прадстаўленне новай запісу.

Мяркуецца, што сэрвіс заўсёды вяртае дадзеныя ў тэкставым фармаце, так што мае сэнс перанесці радок `content_type: txt` з блока кода які належыць канкрэтнаму шляху ў `before` фільтр, які выкарыстоўваецца для ўсіх шляхоў.

```ruby
before do
  content_type :txt
end
```

Па аналогіі з метадам `post` для стварэння запісу мы дадаем метад `get` для атрымання (`R`ead) запісу(аў). Адзін для чытання ўсіх запісаў і адзін для чытання канкрэтнай запіс знойдзеную па ID.

```ruby
get "/api/v1/notes.txt" do
  Note.all.map { |note| "##{note.id} #{note.content}" }.join("\n")
end

get "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  "##{note.id} #{note.content}"
end
```

Па змаўчанні sinatra вяртае HTTP код стану 200, так што відавочнае заданне `status 200` можа быць апушчана.

Вось як наш сэрвіс павінен выглядаць на гэты момант (файл `service.rb` ў каталогу` notes`).

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

## <a name="test-it-manually"></a>Тестирование вручную

Мы можам запусціць наш сэрвіс

    $ ruby service.rb

І праверыць, што ён працуе (або пачаць выкарыстоўваць яго) з дапамогай `curl` інструмента каманднага радка. Стварэнне новай нататкі з параметрам `content`.

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=First%20Note"
    #1 First Note

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=Second%20Note"
    #2 Second Note

Мы выкарыстоўвалі `% 20` замест прабелу ў URL.

Атрымаць усе нататкі

    $ curl -X GET "localhost:4567/api/v1/notes.txt"
    #1 First Note
    #2 Second Note

Атрымаць канкрэтную нататку па ID

    $ curl -X GET "localhost:4567/api/v1/notes/1.txt"
    #1 First Note

Вы таксама можаце выкарыстоўваць ваш вэб-браўзэр для атрымання запісаў.

Для паўнаты функцыянальнасці мы ўсё яшчэ павінны стварыць аперацыі рэдагавання ( `U`pdate) і выдалення (` D`elete). Метады `put` і` delete` адпаведна.

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

Мы можам праверыць, што абодва з іх працуюць. Вам, магчыма, спатрэбіцца перазапусціць сервер (спыніць, націснуўшы `Ctrl` + `C` і зноў запусціць выканаўшы `ruby service.rb`).

Рэдагаванне існуючай нататкі

    $ curl -X PUT "localhost:4567/api/v1/notes/1.txt?content=New%20Content"
    #1 New Content

Выдаленне нататкі

    $ curl -X DELETE "localhost:4567/api/v1/notes/1.txt"

Усё працуе. Гэта выдатна. Нам варта таксама напісаць тэсты. Часам праграмісты робяць гэта, перш чым пісаць асноўнай код прыкладання. Напісаць адзін тэст, які не праходзіць з-за адсутнасці функцыянальнасці, а затым дадаць гэтую функцыянальнасць, тым самым забяспечыўшы праходжанне тэсту. Гэты цыкл называецца "ад чырвонага да зялёным" ("red-green"). Але мы дадаем ўсе тэсты адразу.

## <a name="automated-tests"></a>Автоматизированные тесты

Стварыце, калі ласка, каталог `spec` ўнутры каталога` notes` і файл `service_spec.rb` ў ім са наступным змесцівам:

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

На адзін тэст прыпадае больш адной праверкі. На самай справе хто-то можа лічыць гэта дрэнны практыкай, але я так раблю. Акрамя `rspec` мы выкарыстоўваем` rack-test` для тэставання `rack`-прыкладанняў (якім з'яўляецца `sinatra`). Перайдзіце ў тэчку `notes` ў тэрмінале і выканайце `rspec --color --format=doc`.

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

Так што ўсё працуе і пратэставана. Базавую разуменне таго, чым з'яўляецца HTTP сэрвіс ці API ў вас ужо павінна быць. Вы, напэўна, ведалі гэты артыкул раней. Але магчыма вам патрэбен свежы погляд. Наогул кажучы "паўтарэнне - маці вучэння". Такім чынам, давайце створым гэты ж сэрвіс яшчэ раз! Я жартую. Дурны жарт, я ведаю. Добра, давайце зробім яшчэ адно прыкладанне, якое можа разглядацца як HTTP API, але ўжо ў наступным раздзеле.

## <a name="summary"></a>Рэзюмэ

Мы стварылі першы вэб-сэрвіс у гэтай кнізе. Мы выкарыстоўвалі `sinatra` для стварэння самага сэрвісу і `rspec` з `rack-test` для стварэння аўтаматызаваных тэстаў. Калі вы плануеце выкарыстоўваць `sinatra` я рэкамендую вам прачытаць кнігу [Sinatra: Up and Running](http://shop.oreilly.com/product/0636920019664.do) і наведаць сайт [www.sinatrarb.com](http://www.sinatrarb.com/).
