Глава #10. Профилирование
=========================

Вообще говоря, при создании веб-сервиса, вы должны убедиться, что приложение действительно относительно независим от других веб-ресурсов и содержит инкапсулированную логику. Удаленные HTTP запросы являются причиной снижения производительности, а также усложняют внутреннее тестирование приложения. Постарайтесь свести к минимуму количество удаленных HTTP-запросов при обработке запроса к веб-сервису. Например не загружайте удаленного пользователя для конкретного действия, если веб-сервис работает одинаково для зарегистрированных и не зарегистрированных пользователей.

Одним из способов разобраться в том какие части приложения являются наиболее медленными является профилирование.

## <a name="ruby-built-in-profiler"></a>Встроенный `ruby` Profiler

Вы можете использовать встроенный `ruby` Profiler с опцией `-r profile` из командной строки:

    $ ruby -r profile application.rb

Или вы можете сделать это вручную [из кода приложения](http://stackoverflow.com/questions/4347466/whats-the-best-way-to-profile-a-sinatra-application).

## <a name="ruby-prof"></a>ruby-prof

Мы будем профилировать сервис Zip-кодов. Добавьте гем `ruby-prof` в `Gemfile`:

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

И запустите

    $ bundle install

Мы будем профилировать действие `update` веб-сервиса. Добавьте этот код в файл `app/controllers/zip_codes_controller.rb`:

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

Добавим задержку 15 секунд в `users` веб-сервиса. Вот обновленный файл `service.rb`:

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

Запустите `users` веб-сервис на порту 4545:

    $ ruby service.rb -p 4545

Затем запустите веб-сервис Zip-кодов (по умолчанию на порту 4567):

    $ ruby application.rb

Редактирование одного из существующих Zip-кодов из другого окна терминала (или вкладки):

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
       -X PUT \
       -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
       -H "Content-Type: application/json" \
       -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-03-23T11:43:25.475Z"}}

Посмотрите в окно терминала, где запущен сервис Zip-кодов:

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

Эту информацию немного трудно читать, но вы можете видеть, что удаленный вызов HTTP проводили в течение 15 секунд.

## <a name="new-relic"></a>New Relic

Еще одним хорошим инструментом профилирования является `New Relic`. Вы должны иметь зарегистрированный аккаунт на сайте `newrelic.com`, чтобы использовать его. После установки вы можете просматривать статистику работы приложения.

![Newrelic graph](../static/images/newrelic_rpm_graph.png)

Время выполнения разбито на секции: `Middleware`, `Ruby`, `Database`, `Web external`.

Установить `New Relic` не сложно: добавьте гем `newrelic_rpm` в `Gemfile` в секцию `production`, выполните `bundle install` и добавьте конфигурационный файл `config/newrelic.yml` с учетными данными.

## <a name="partial-denormalization"></a>Частичная денормализация в реляционной базе данных

Если приложение использует реляционную базу данных иногда можно добиться повышения производительности за счет сокращения количества SQL запросов на один HTTP запрос или уменьшения количества `JOIN`-ов в SQL запросах.

Покажем это на примере. Представьте себе, приложение, которое имеет такие модели: `Group`, `Post`, `Comment` и таблицы базы данных, стоящие за ними: `groups`,` posts`, `comments`. Группа содержит много сообщений и каждое сообщение может иметь много комментариев. Таблица базы данных `posts` содержит столбец `group_id`, который является внешним ключом по отношению к таблице `groups`. Таблица базы данных `comments` содержит столбец `post_id`, который является внешним ключом по отношению к таблице `posts`. Посмотрите на эту UML диаграмму.

![Normilized structure](../static/images/normilized.png)

Приложение содержит операцию для редактирования комментария и только пользователь, который является менеджером группы имеет возможность редактировать комментарий, связанный с сообщением в этой группе. Таким образом, каждый раз, приложение должно находить соответствующее сообщение к комментарию и проверять его `group_id`.

Мы можем сделать частичную денормализация, чтобы предотвратить выполнение дополнительного запроса - добавить колонку `group_id` к таблице  `comments`.

![Partiallly denormilized structure](../static/images/partiallly_denormilized.png)

Таким образом, мы можем уменьшить количество запросов к базе данных.

Атрибут `group_id` является неизменным для` Post`. Мы можем переопределить метод `#post_id=` в классе `Comment`.

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

Также хорошей практикой можно считать установку внешних ключей (`post_id` и `group_id`) как "не NULL" в базе данных, если модель проверяет присутствие ассоциации.

## <a name="summary"></a>Резюме

Мы использовали `gem` [ruby-prof](https://github.com/ruby-prof/ruby-prof) для профилирования приложения на `sinatra`. Мы коротко обсудили `New Relic` и рассмотрели технику частичной нормализации в реляционных базах данных для повышения производительности приложений.
