Глава #10. Профілювання
=======================

Взагалі кажучи, при створенні веб-сервісу, ви повинні переконатися, що додаток дійсно відносно незалежний від інших веб-ресурсів і містить інкапсульовану логіку. Дистанційні HTTP запити є причиною зниження продуктивності, а також ускладнюють внутрішнє тестування програми. Постарайтеся звести до мінімуму кількість віддалених HTTP-запитів при обробці запиту до веб-сервісу. Наприклад не завантажуйте віддаленого користувача для конкретної дії, якщо веб-сервіс працює однаково для зареєстрованих і не зареєстрованих користувачів.

Одним із способів розібратися в тому які частини додатки є найбільш повільними є профілювання.

## <a name="ruby-built-in-profiler">Вбудований `ruby` Profiler

Ви можете використовувати вбудований `ruby` Profiler з опціей` -r profile` з командного рядка:

    $ ruby -r profile application.rb

Або ви можете зробити це вручну [з коду програми](http://stackoverflow.com/questions/4347466/whats-the-best-way-to-profile-a-sinatra-application).

## <a name="ruby-prof"></a>ruby-prof

Ми будемо профілювати сервіс Zip-кодів. Додайте гем `ruby-prof` в` Gemfile`:

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
gem 'ruby-prof'

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

І запустіть

    $ bundle install

Ми будемо профілювати дію `update` веб-сервісу. Додайте цей код у файл `app / controllers / zip_codes_controller.rb`:

```ruby
put "/api/v1/zip_codes/:id.json" do
  # Profile the code
  RubyProf.start unless RubyProf.running?

  param :id, Integer, max: 2147483647
  param :zip_code, Hash, required: true
  load_and_authorize! ZipCode
  @zip_code.update_attributes!(params[:zip_code]) if params[:zip_code].any?
  response = @zip_code.to_json

  result = RubyProf.stop
  # Print a flat profile to text
  printer = RubyProf::FlatPrinter.new(result)
  printer.print(STDOUT)

  response
end
```

Додамо затримку 15 секунд в `users` веб-сервісу. Ось оновлений файл `service.rb`:

```ruby
require "sinatra/main"

get "/api/v1/users/me.json" do
  content_type :json
  sleep 15 # timeout delay

  case request.env['HTTP_AUTHORIZATION']
  when nil then [403, '{"message":"Access Forbidden"}']
  when "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
    then '{"user":{"type":"AdminUser"}}'
  when "OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"
    then '{"user":{"type":"RegularUser"}}'
  else [401, '{"message":"Invalid or expired token"}']
  end
end
```

Запустіть `users` веб-сервіс на порту 4545:

    $ ruby service.rb -p 4545

Потім запустіть веб-сервіс Zip-кодів (за замовчуванням на порту 4567):

    $ ruby application.rb

Редагування одного з існуючих Zip-кодів з іншого вікна терміналу (або вкладки):

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
       -X PUT \
       -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
       -H "Content-Type: application/json" \
       -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-03-23T11:43:25.475Z"}}

Подивіться у вікно терміналу, де запущений сервіс Zip-кодів:

    $ %self      total      self      wait     child     calls  name
       6.54     15.003     0.991    14.012     0.000        1   <Class::IO>#select 
       0.33      0.050     0.050     0.000     0.000       90   Array#join 
       0.13      0.020     0.020     0.000     0.000       14   PG::Connection#async_exec 
       0.05      0.008     0.008     0.000     0.000       99   Module#module_eval 
       0.03      0.005     0.005     0.000     0.000        1   PG::Connection#initialize 
       0.03      0.005     0.004     0.000     0.001        1   ActiveRecord::QueryMethods#where! 
       0.02      0.002     0.002     0.000     0.000        3   ActiveModel::Validations#errors 
       0.02      0.079     0.002     0.000     0.077      215  *Array#each 

       ...

       0.00     15.007     0.000     0.000    15.007        1   Faraday::Connection#get

Цю інформацію трохи важко читати, але ви можете бачити, що віддалений виклик HTTP виконувався протягом 15 секунд.

## <a name="new-relic"></a>New Relic

Ще одним хорошим інструментом профілювання є `New Relic`. Ви повинні мати зареєстрований аккаунт на сайті `newrelic.com`, щоб використовувати його. Після встановлення ви можете переглядати статистику роботи програми.

![Newrelic graph](../static/images/newrelic_rpm_graph.png)

Час виконання розбито на секції: `Middleware`,` Ruby`, `Database`,` Web external`.

Встановити `New Relic` не складно: додайте гем `newrelic_rpm` в `Gemfile` в секцію `production`, виконайте `bundle install` і додайте конфігураційний файл `config/newrelic.yml` з обліковими даними.

## <a name="partial-denormalization"></a>Часткова денормализация в реляційній базі даних

Якщо додаток використовує реляційну базу даних іноді можна домогтися підвищення продуктивності за рахунок скорочення кількості SQL запитів на один HTTP запит або зменшення кількості `JOIN`-ів в SQL запитах.

Покажемо це на прикладі. Уявіть собі, додаток, який має такі моделі: `Group`, `Post`, `Comment` і таблиці бази даних, які стоять за ними: `groups`, `posts`,` comments`. Група містить багато повідомлень і кожне повідомлення може мати багато коментарів. Таблиця бази даних `posts` містить стовпець` group_id`, який є зовнішнім ключем по відношенню до таблиці `groups`. Таблиця бази даних `comments` містить стовпець` post_id`, який є зовнішнім ключем по відношенню до таблиці `posts`. Подивіться на цю UML діаграму.

![Normilized structure](../static/images/normilized.png)

Додаток містить операцію для редагування коментаря і тільки користувач, який є менеджером групи має можливість редагувати коментар, пов'язаний з повідомленням у цій групі. Таким чином, кожен раз, додаток повинен знаходити відповідне повідомлення до коментаря і перевіряти його `group_id`.

Ми можемо зробити часткову денормализация, щоб запобігти виконанню додаткового запиту - додати колонку `group_id` до таблиці `comments`.

![Partiallly denormilized structure](../static/images/partiallly_denormilized.png)

Таким чином, ми можемо зменшити кількість запитів до бази даних.

Атрибут `group_id` є незмінним для` Post`. Ми можемо перевизначити метод `#post_id=` в класі `Comment`.

```ruby
class Comment
  belongs_to :post

  validates :post, presence: true

  def post_id=(value)
    write_attribute(:post_id, value)
    write_attribute(:group_id, post.try(:group_id))
    value
  end
end
```

Також хорошою практикою можна вважати установку зовнішніх ключів (`post_id` і `group_id`) як "не NULL" в базі даних, якщо модель перевіряє присутність асоціації.

## <a name="summary"></a>Резюме

Ми використовували `gem` [ruby-prof](https://github.com/ruby-prof/ruby-prof) для профілювання додатка на `sinatra`. Ми коротко обговорили `New Relic` і розглянули техніку часткової нормалізації в реляційних базах даних для підвищення продуктивності додатків.
