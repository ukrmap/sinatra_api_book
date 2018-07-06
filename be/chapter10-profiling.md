Кіраўнік #10. Прафіляванне
==========================

Наогул кажучы, пры стварэнні вэб-сэрвісу, вы павінны пераканацца, што прыкладанне сапраўды адносна незалежны ад іншых вэб-рэсурсаў і змяшчае інкапсуляванага логіку. Выдаленыя HTTP запыты з'яўляюцца прычынай зніжэння прадукцыйнасці, а таксама ўскладняюць ўнутранае тэставанне прыкладання. Паспрабуйце звесці да мінімуму колькасць аддаленых HTTP-запытаў пры апрацоўцы запыту да вэб-сэрвісу. Напрыклад ня загружайце аддаленага карыстальніка для канкрэтнага дзеяння, калі вэб-сэрвіс працуе аднолькава для зарэгістраваных і не зарэгістраваныя карыстальнікаў.

Адным са спосабаў разабрацца ў тым якія часткі прыкладання з'яўляюцца найбольш павольнымі з'яўляецца прафіляванне.

## <a name="ruby-built-in-profiler"></a>Убудаваны `ruby` Profiler

Вы можаце выкарыстоўваць ўбудаваны `ruby` Profiler з опцыяй `-r profile` з каманднага радка:

    $ ruby -r profile application.rb

Ці вы можаце зрабіць гэта ўручную [з кода прыкладання](http://stackoverflow.com/questions/4347466/whats-the-best-way-to-profile-a-sinatra-application).

## <a name="ruby-prof"></a>ruby-prof

Мы будзем прафіляванага сэрвіс Zip-кодаў. Дадайце гем `ruby-prof` ў `Gemfile`:

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

І запусціце

    $ bundle install

Мы будзем прафіляванага дзеянне `update` вэб-сэрвісу. Дадайце гэты код у файл `app/controllers/zip_codes_controller.rb`:

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

Дадамо затрымку 15 секунд у `users` вэб-сэрвісу. Вось абноўлены файл `service.rb`:

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

Запусціце `users` вэб-сэрвіс на порце 4545:

    $ ruby service.rb -p 4545

Затым запусціце вэб-сэрвіс Zip-кодаў (па змаўчанні на порце 4567):

    $ ruby application.rb

Рэдагаванне аднаго з існуючых Zip-кодаў З другога акна тэрмінала (або ўкладкі):

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
       -X PUT \
       -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
       -H "Content-Type: application/json" \
       -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-03-23T11:43:25.475Z"}}

Паглядзіце ў акно тэрмінала, дзе запушчаны сэрвіс Zip-кодаў:

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

Гэтую інфармацыю трохі цяжка чытаць, але вы можаце бачыць, што аддалены выклік HTTP праводзілі на працягу 15 секунд.

## <a name="new-relic"></a>New Relic

Яшчэ адным добрым інструментам прафілявання з'яўляецца `New Relic`. Вы павінны мець зарэгістраваны рахунак на сайце `newrelic.com`, каб выкарыстоўваць яго. Пасля ўстаноўкі вы можаце праглядаць статыстыку працы прыкладання.

![Newrelic graph](../static/images/newrelic_rpm_graph.png)

Час выканання пабіта на секцыі: `Middleware`, `Ruby`, `Database`, `Web external`.

Ўсталяваць `New Relic` не сложно: дадайце гем `newrelic_rpm` ў `Gemfile` ў секцыю` production`, выканайце `bundle install` і дадайце канфігурацыйны файл `config/newrelic.yml` з уліковымі дадзенымі.

## <a name="partial-denormalization"></a>Частковая денормализация ў рэляцыйнай базе дадзеных

Калі дадатак выкарыстоўвае рэляцыйную базу дадзеных часам можна дамагчыся павышэння прадукцыйнасці за кошт скарачэння колькасці SQL запытаў на адзін HTTP запыт або змяншэння колькасці `JOIN`-ов ў SQL запытах.

Пакажам гэта на прыкладзе. Уявіце сабе, дадатак, якое мае такія мадэлі: `Group`, `Post`, `Comment` і табліцы базы дадзеных, якія стаяць за імі: `groups`, `posts`,` comments`. Група змяшчае шмат паведамленняў і кожнае паведамленне можа мець шмат каментароў. Табліца базы дадзеных `posts` змяшчае слупок `group_id`, які з'яўляецца знешніх ключом у адносінах да табліцы `groups`. Табліца базы дадзеных `comments` змяшчае слупок `post_id`, які з'яўляецца знешніх ключом у адносінах да табліцы `posts`. Паглядзіце на гэтую UML дыяграму.

![Normilized structure](../static/images/normilized.png)

Дадатак ўтрымлівае аперацыю для рэдагавання каментара і толькі карыстальнік, які з'яўляецца мэнэджэрам групы мае магчымасць рэдагаваць каментар, звязаны з паведамленнем у гэтай групе. Такім чынам, кожны раз, прыкладанне павінна знаходзіць адпаведнае паведамленне да камэнтару і правяраць яго `group_id`.

Мы можам зрабіць частковую денормализация, каб прадухіліць выкананне дадатковага запыту - дадаць калонку `group_id` да табліцы `comments`.

![Partiallly denormilized structure](../static/images/partiallly_denormilized.png)

Такім чынам, мы можам зменшыць колькасць запытаў да базе дадзеных.

Атрыбут `group_id` з'яўляецца нязменным для `Post`. Мы можам перавызначыць метад `#post_id =` ў класе `Comment`.

```ruby
class Comment
  belongs_to :post

  validates_presence_of :post

  def post_id=(value)
    write_attribute(:post_id, value)
    write_attribute(:group_id, post.try(:group_id))
    value
  end
end
```

Таксама добрай практыкай можна лічыць ўстаноўку знешніх ключоў (`post_id` і `group_id`) як "ня NULL" ў базе дадзеных, калі мадэль правярае прысутнасць асацыяцыі.

## <a name="summary"></a>Рэзюмэ

Мы выкарыстоўвалі `gem` [ruby-prof](https://github.com/ruby-prof/ruby-prof) для прафілявання прыкладання на `sinatra`. Мы коратка абмеркавалі `New Relic` і разгледзелі тэхніку частковай нармалізацыі ў рэляцыйных базах дадзеных для павышэння прадукцыйнасці прыкладанняў.
