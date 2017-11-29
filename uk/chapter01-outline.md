Глава #1. Начерки
=================

Давайте створимо простий веб-сервіс для управління нотатками (notes), щоб ознайомитися з деякими технологіями, які використовуються в цій книзі і, звичайно, зміцнити розуміння концепцій побудови веб-сервісів.

Ми будемо зберігати замітки в SQLite базі даних і надавати доступ до нотаток через HTTP з використанням notes API. Є деякі варіанти технологій для створення сервісів, такі як `rails-api`, `sinatra`, `grape` або їх комбінації. Для всіх сервісів в цій книзі ми будемо використовувати `sinatra`, як правило, це справа смаку, `sinatra` є коротким і добре вписується у вимоги.

## <a name="infrastructure"></a>Інфраструктура

Будь ласка, створіть теку `notes`, в якій буде зберігатися файли коду нашого сервісу. Ми повинні встановити три ruby гема для внутрішнього обслуговування сервісу, а саме: `sinatra`, `sqlite3` і `activerecord` і два гема для тестування: `rspec` і `rack-test`.
Створіть файл з ім'ям `Gemfile` в теці `notes` з наступними вмістом:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'sqlite3'
gem 'activerecord'
gem 'rspec'
gem 'rack-test'
```

Потім в терміналі перейдіть в теку `notes` і виконайте `bundle install` (ruby, rubygems і гем `bundler` мають бути встановлені до цього). Це встановить всі вищеперелічені геми, а також створить файл `Gemfile.lock` з версіями використаних гемів. Ви повинні побачити щось подібне.

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

Установча частина завершена. Тепер ми можемо приступити до розробки нашого сервісу. Будь ласка, створіть файл `service.rb` в теці `notes` з підключенням трьох необхідних гемів.

```ruby
require "sqlite3"
require "active_record"
require "sinatra/main"
```

Зверніть увагу, що гем називається `activerecord` але ми підключаємо `active_record`. А також тільки частина sinatra, але це не дуже важливо.

## <a name="create-model"></a>Створення моделі

Потім нам потрібно створити таблицю notes в базі даних. Додайте наступний код в `service.rb`.

```ruby
class CreateNotes < ActiveRecord::Migration
  def change
    create_table :notes do |t|
      t.string :content, null: false, default: 'Empty'
    end
  end
end
```

Таблиця називається `notes` (не дивно) і має тільки одне поле - content типу String зі значенням за замовчуванням "Empty" ("Пусто").

Ми повинні створити відповідний клас ORM для управління записами таблиці notes. Це просто - успадковуємо клас від `ActiveRecord::Base` і називаємо його як таблицю `notes` тільки в однині (клас зможе визначити ім'я таблиці від свого імені).

```ruby
class Note < ActiveRecord::Base
end
```

Встановлення з'єднання.

```ruby
ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: 'db/notes.sqlite3')
```

ворення таблиці `notes`, якщо вона ще не існує - виконання міграції.

```ruby
CreateNotes.new.change unless ActiveRecord::Base.connection.table_exists? :notes
```

Це був код для створення інфраструктури.

## <a name="routes-for-crud"></a>Маршрути (роути, шляхи) для CRUD

Будь-яка дія, яку можна очікувати від вашої програми може бути виконана за допомогою однієї з чотирьох операцій: створення (`C`raate) сутності, отримання (`R`ead) сутності, оновлення (`U`pdate) сутності і видалення (`D`elete) сутності (`CRUD`). Ці дії пов'язані з чотирма (або близько чотирьох) HTTP заголовків: POST, GET, PUT (може бути PATCH), DELETE. URL ресурсу так само повинен бути якось пов'язаний з ім'ям ресурсу - в нашому випадку "notes". Це геніальна простота принципів `REST`.

Ми також додаємо префікс "api" до URL адрес, судячи по якому користувачам відразу видно, що ця URL-адреса є частиною якогось API. Крім того, префікс "v1", який може бути корисний, якщо ви плануєте підтримувати кілька версій API в майбутньому. Закінчення URL-адреси є позначенням формату, який підтримує сервіс. Нехай наш сервіс буде підтримувати простий текстовий формат ("text/plain") і URL-адреси будуть закінчуватися на ".txt".

Ми будемо додавати код для цих чотирьох `CRUD` операцій в тому ж порядку один за іншим. Перший - це створення (`C`reate).

```ruby
post "/api/v1/notes.txt" do
  content_type :txt
  note = Note.create(content: params[:content])
  status 201
  "##{note.id} #{note.content}"
end
```

Метод `post` гема `sinatra` створює обробник HTTP запиту посланого методом POST в наданий URL - "/api/v1/notes.txt". `content_type: txt` додає HTTP заголовок, який повідомляє клієнта про формат відповіді. Після цього створюється запис (збереження в базі даних), очікується, що користувач надає параметр з ім'ям `content` (інакше значення атрибута буде встановлено в значення за замовчуванням - "Empty"). Код стану HTTP 201, що означає, що новий ресурс створюється. Сервіс повертає текстове представлення нового запису.

Передбачається, що сервіс завжди повертає дані в текстовому форматі, так що має сенс перенести рядок `content_type: txt` з блоку коду, що належить конкретному шляху в `before` фільтр, який використовується для всіх шляхів.

```ruby
before do
  content_type :txt
end
```

За аналогією з методом `post` для створення запису ми додаємо метод `get` для отримання (`R`ead) запису(ів). Один для читання всіх записів і один для читання конкретного запису знайденого по ID.

```ruby
get "/api/v1/notes.txt" do
  Note.all.map { |note| "##{note.id} #{note.content}" }.join("\n")
end

get "/api/v1/notes/:id.txt" do
  note = Note.find(params[:id])
  "##{note.id} #{note.content}"
end
```

За замовчуванням sinatra повертає HTTP код стану 200, так що явне завдання `status 200` може бути опущено.

Ось як наш сервіс повинен виглядати на цей момент (файл `service.rb` в теці `notes`).

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

## <a name="test-it-manually"></a>Тестування вручну

Ми можемо запустити наш сервіс

    $ ruby service.rb

І перевірити, що він працює (або почати використовувати його) за допомогою `curl` інструменту командного рядка. Створення нової замітки з параметром `content`.

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=First%20Note"
    #1 First Note

    $ curl -X POST "localhost:4567/api/v1/notes.txt?content=Second%20Note"
    #2 Second Note

Ми використовували `%20` замість пробілу в URL.

Отримати всі замітки

    $ curl -X GET "localhost:4567/api/v1/notes.txt"
    #1 First Note
    #2 Second Note

Отримати конкретну замітку по ID

    $ curl -X GET "localhost:4567/api/v1/notes/1.txt"
    #1 First Note

Ви також можете використовувати ваш веб-браузер для отримання записів.

Для повноти функціональності ми все ще повинні створити операції редагування (`U`pdate) і видалення (`D`elete). Методи `put` і `delete` відповідно.

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

Ми можемо перевірити, що обидва з них працюють. Вам, можливо, буде потрібно перезапустити сервер (зупинити, натиснувши `Ctrl` + `C` і знову запустити виконавши `ruby service.rb`).

Редагування існуючої замітки

    $ curl -X PUT "localhost:4567/api/v1/notes/1.txt?content=New%20Content"
    #1 New Content

Видалення замітки

    $ curl -X DELETE "localhost:4567/api/v1/notes/1.txt"

Все працює. Це добре. Нам слід також написати тести. Іноді програмісти роблять це, перш ніж писати основний код програми. Написати один тест, який не проходить через відсутність функціональності, а потім додати цю функціональність, тим самим забезпечивши проходження тесту. Цей цикл називається "від червоного до зеленого" ("red-green"). Але ми додаємо всі тести відразу.

## <a name="automated-tests"></a>Автоматизовані тести

Створіть, будь ласка, теку `spec` всередині теки `notes` і файл `service_spec.rb` в ній наступного вмісту:

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

На один тест припадає більше однієї перевірки. Насправді хтось може вважати це поганою практикою, але я так роблю. Крім `rspec` ми використовуємо `rack-test` для тестування `rack`-додатків (яким і є `sinatra`). Перейдіть в теку `notes` в терміналі і виконайте `rspec --color --format = doc`.

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

Так що все працює і протестовано. Базове розуміння того, чим є HTTP сервіс або API у вас вже повинно бути. Ви, напевно, знали цей матеріал раніше. Але можливо вам потрібен свіжий погляд. Взагалі кажучи "повторення - мати навчання". Отже, давайте створимо цей же сервіс ще раз! Я жартую. Дурний жарт, я знаю. Добре, давайте зробимо ще один додаток, який може розглядатися як HTTP API, але вже в наступній главі.

## <a name="summary"></a>Резюме

Ми створили перший веб-сервіс в цій книзі. Ми використовували `sinatra` для створення самого сервісу і `rspec` з `rack-test` для створення автоматизованих тестів. Якщо ви плануєте використовувати `sinatra` я рекомендую вам прочитати книгу [Sinatra: Up and Running](http://shop.oreilly.com/product/0636920019664.do) і відвідати сайт [www.sinatrarb.com](http://www.sinatrarb.com/).
