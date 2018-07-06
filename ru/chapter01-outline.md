Глава #1. Наброски
==================

Давайте создадим простой веб-сервис для управления заметками (notes), чтобы ознакомиться с некоторыми технологиями, которые используются в этой книге и, конечно, укрепить понимание концепций построения веб-сервисов.

Мы будем хранить заметки в SQLite базе данных и предоставлять доступ к заметкам через HTTP с использованием notes API. Есть некоторые варианты технологий для создания сервисов, такие как `rails-api`,`sinatra`, `grape` или их комбинации. Для всех сервисов в этой книге мы будем использовать `sinatra`, как правило, это дело вкуса, `sinatra` лаконичный фреймворк и хорошо вписывается в требования.

## <a name="infrastructure"></a>Инфраструктура

Пожалуйста, создайте папку `notes`, в которой будет храниться файлы кода нашего сервиса. Мы должны установить три ruby гема для внутреннего обслуживания сервиса, а именно: `sinatra`, `sqlite3` и `activerecord` и два гема для тестирования: `rspec` и `rack-test`.
Создайте файл с именем `Gemfile` в папке `notes` со следующими содержимым:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'sqlite3'
gem 'activerecord'
gem 'rspec'
gem 'rack-test'
```

Затем в терминале перейдите в папку `notes` и выполните `bundle install` (ruby, rubygems и гем `bundler` должны быть установлены до этого). Это установит все вышеперечисленные гемы, а также создаст файл `Gemfile.lock` с версиями использованных гемов. Вы должны увидеть что-то подобное.

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

Установочная часть завершена. Теперь мы можем приступить к разработке нашего сервиса. Пожалуйста, создайте файл `service.rb` в папке `notes` с подключением трех необходимых гемов.

```ruby
require "sqlite3"
require "active_record"
require "sinatra/main"
```

Обратите внимание, что гем называется `activerecord` но мы подключаем `active_record`. А также только часть sinatra, но это не очень важно.

## <a name="create-model"></a>Создание модели

Затем нам нужно создать таблицу notes в базе данных. Добавьте следующий код в `service.rb`.

```ruby
class CreateNotes < ActiveRecord::Migration
  def change
    create_table :notes do |t|
      t.string :content, null: false, default: 'Empty'
    end
  end
end
```

Таблица называется `notes` (не удивительно) и имеет только одно поле - content типа string со значением по умолчанию "Empty" ("Пусто").

Мы должны создать соответствующий класс ORM для управления записями таблицы notes. Это просто - наследуем класс от `ActiveRecord::Base` и называем его как таблицу `notes` только в единственном числе (класс сможет определить имя таблицы от своего имени).

```ruby
class Note < ActiveRecord::Base
end
```

Установление соединения.

```ruby
ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: 'db/notes.sqlite3')
```

Создание таблицы `notes`, если она еще не существует - выполнение миграции.

```ruby
CreateNotes.new.change unless ActiveRecord::Base.connection.table_exists? :notes
```

Это был код для создания инфраструктуры.

## <a name="routes-for-crud"></a>Маршруты (роуты, пути) для CRUD

Любое действие, которое можно ожидать от вашего приложения может быть выполнено при помощи одной из четырех операций: создание (`C`raate) сущности, получение (`R`ead) сущности, обновление (`U`pdate) сущности и удаление (`D`elete) сущности (`CRUD`). Эти действия связаны с четырьмя (или около четырех) HTTP заголовков: POST, GET, PUT (может быть PATCH), DELETE. URL ресурса так же должен быть как-то связан с именем ресурса - в нашем случае "notes". Это гениальная простота принципов `REST`.

Мы также добавляем префикс "api" к URL адресам, судя по которому пользователям сразу видно, что этот URL-адреса являются частью какого-то API. Кроме того, префикс "v1", который может быть полезен, если вы планируете поддерживать несколько версий API в будущем. Окончание URL-адреса есть обозначением формата, который поддерживает сервис. Пусть наш сервис будет поддерживать простой текстовый формат ("text/plain") и URL-адреса будут оканчиваться на ".txt".

Мы будем добавлять код для этих четырех `CRUD` операций в том же порядке один за другим. Первый - это создание (`C`reate).

```ruby
post "/api/v1/notes.txt" do
  content_type :txt
  note = Note.create(content: params[:content])
  status 201
  "##{note.id} #{note.content}"
end
```

Метод `post` гема `sinatra` создает обработчик HTTP запроса посланного методом POST в предоставленный URL - "/api/v1/notes.txt". `content_type: txt` добавляет HTTP заголовок, который уведомляет клиента о формате ответа. После этого создается запись (сохранение в базе данных), ожидается, что пользователь предоставляет параметр с именем `content` (иначе значение атрибута будет установлено в значение по умолчанию - строка "Empty"). Код состояния HTTP 201, что означает, что новый ресурс создается. Сервис возвращает текстовое представление новой записи.

Предполагается, что сервис всегда возвращает данные в текстовом формате, так что имеет смысл перенести строку  `content_type :txt` из блока кода принадлежащего конкретному пути в `before` фильтр, который используется для всех путей.

```ruby
before do
  content_type :txt
end
```

По аналогии с методом `post` для создания записи мы добавляем метод `get` для получения (`R`ead) записи(ей). Один для чтения всех записей и один для чтения конкретной запись найденой по ID.

```ruby
get "/api/v1/notes.txt" do
  Note.all.map { |note| "##{note.id} #{note.content}" }.join("\n")
end

get "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  "##{note.id} #{note.content}"
end
```

По умолчанию sinatra возвращает HTTP код состояния 200, так что явное задание `status 200` может быть опущено.

Вот как наш сервис должен выглядеть на этот момент (файл `service.rb` в каталоге `notes`).

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

Мы можем запустить наш сервис

    $ ruby service.rb

И проверить, что он работает (или начать использовать его) с помощью `curl` инструмента командной строки. Создание новой заметки с параметром `content`.

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=First%20Note"
    #1 First Note

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=Second%20Note"
    #2 Second Note

Мы использовали `%20` вместо пробела в URL.

Получить все заметки

    $ curl -X GET "localhost:4567/api/v1/notes.txt"
    #1 First Note
    #2 Second Note

Получить конкретную заметку по ID

    $ curl -X GET "localhost:4567/api/v1/notes/1.txt"
    #1 First Note

Вы также можете использовать ваш веб-браузер для получения записей.

Для полноты функциональности мы все еще должны создать операции редактирования (`U`pdate) и удаления (`D`elete). Методы `put` и` delete` соответственно.

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

Мы можем проверить, что оба из них работают. Вам, возможно, потребуется перезапустить сервер (остановить, нажав `Ctrl` + `C` и снова запустить выполнив `ruby service.rb`).

Редактирование существующей заметки

    $ curl -X PUT "localhost:4567/api/v1/notes/1.txt?content=New%20Content"
    #1 New Content

Удаление заметки

    $ curl -X DELETE "localhost:4567/api/v1/notes/1.txt"

Все работает. Это здорово. Нам следует также написать тесты. Иногда программисты делают это, прежде чем писать основной код приложения. Написать один тест, который не проходит из-за отсутствия функциональности, а затем добавить эту функциональность, тем самым обеспечив прохождение теста. Этот цикл называется "от красного к зелёному" ("red-green"). Но мы добавляем все тесты сразу.

## <a name="automated-tests"></a>Автоматизированные тесты

Создайте, пожалуйста, каталог `spec` внутри каталога `notes` и файл `service_spec.rb` в нем со следующем содержимым:

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

На один тест приходится более одной проверки. На самом деле кто-то может считать это плохой практикой, но я так делаю. Кроме `rspec` мы используем `rack-test` для тестирования `rack`-приложений (которым является `sinatra`). Перейдите в папку `notes` в терминале и выполните `rspec --color --format=doc`.

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

Так что все работает и протестировано. Базовое понимание того, чем является HTTP сервис или API у вас уже должно быть. Вы, наверное, знали этот материал раньше. Но возможно вам нужен свежий взгляд. Вообще говоря "повторение - мать учения". Итак, давайте создадим этот же сервис еще раз! Я шучу. Глупая шутка, я знаю. Хорошо, давайте сделаем еще одно приложение, которое может рассматриваться как HTTP API, но уже в следующей главе.

## <a name="summary"></a>Резюме

Мы создали первый веб-сервис в этой книге. Мы использовали `sinatra` для создания самого сервиса и `rspec` с `rack-test` для создания автоматизированных тестов. Если вы планируете использовать `sinatra` я рекомендую вам прочитать книгу [Sinatra: Up and Running](http://shop.oreilly.com/product/0636920019664.do) и посетить сайт [www.sinatrarb.com](http://www.sinatrarb.com/).
