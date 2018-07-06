Глава #2. Управление базой данных и общая структура приложения
==============================================================

В этой главе мы создадим сервис для простой игры Tic Tac Toe (крестики-нолики), мы будем уделять больше внимания структуре приложения и задачам управления базой данныx: для создания базы данных, для создания миграции, выполнение и отмены миграции.

Мы будем хранить игровые записи в реляционной базе данных (а именно PostgreSQL, но вы можете использовать и другие, такие как SQLite или MySQL). Запись игры сохраняет свою доску (поле игры), после создания доска пуста. Веб-сервис позволяет игроку делать ход на клетке доски (поля), после чего сервис делает собственный ход и возвращает обновленное представление игры в текстовом формате.

Вот представление пустого игрового поля:

       |   |
    -----------
       |   |
    -----------
       |   |

А здесь, как это может выглядеть после первого хода:

       |   |
    -----------
     O | X |
    -----------
       |   |

## <a name="service-interface"></a>Интерфейс веб-сервиса

Игрок должен быть в состоянии создать игру отправив `POST` запрос на URL `"games.txt"`.

    $ curl -X POST "localhost:4567/api/v1/games.txt"
    Game #1
    Status: In Progress

       |   |
    -----------
       |   |
    -----------
       |   |

Также игрок должен быть в состоянии сделать ход, отправив `PUT` запрос на URL конкретной игры (с ID) с номером ячейки, где игрок желает поставить крестик (символ "X").

    $ curl -X PUT "localhost:4567/api/v1/games/1.txt?game%5Bmove%5D=4"
    Game #1
    Status: In Progress

       |   |
    -----------
     O | X |
    -----------
       |   |

Сервис делает свой ход - компьютер ставит "0" на пустую ячейку (если игра не заканчивается после хода игрока), и уведомляет о состоянии игры: "В процессе", "Выиграно", "Проиграно", "Ничья". Обратите внимание, что игрок передает параметры игры, в данном случае это GET параметры (часть URL), но вообще это должны быть параметры POST (переданные в теле запроса HTTP) - длина URL ограничивается в зависимости от веб-сервера, так что в целом мы должны использовать POST. Пока же будем использовать GET для некоторой простоты.

Параметры игры - это хранилище ключей и значений, которое содержит только один ключ - "move" (ход), значение - это номер ячейки, в которой игрок хочет поставить крестик (символ "X"). Ячейка должна быть пустой для того, чтоб была возможность сделать в неё ход. Ячейки пронумерованы от 0 до 9 (это не правила игры, а наше представление о поле игры, для того, чтобы иметь возможность сделать ход):

     0 | 1 | 2
    -----------
     3 | 4 | 5
    -----------
     6 | 7 | 8

Если игра не закончена игрок может сделать новый ход.

Игра завершена если она выиграна или проиграна, или нет больше пустых ячеек на поле. Игра считается выигранной (игроком), если три крестика размещены на одной линии (горизонтальной, вертикальной или диагональной). Игра считается проигранной (игроком), если три нолика находятся на одной линии.

## <a name="games-model-interface"></a>Интерфейс модели игры

Наша игровая модель должна выглядеть себя следующим образом:

```ruby
# created new game with empty board.
game = Game.create

# Game has it's own unique ID
game.id

# making a move (computer makes countermove and saves record into database)
game.update_attributes(move: 4)

# Game status: "In Progress", "Won", "Lost", "Drow"
gmae.status

# Array of board cells: each value equals one of strings "X", "O" or ""
game.cells
```

## <a name="create-skeleton-of-the-service"></a>Создание структуры сервиса

Создайте пожалуйста папку `noughts_and_crosses` и файл `Gemfile` в ней со списком необходимых гемов:

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'

group :development, :test do
  gem 'thin'
  gem 'pry-debugger'
end

group :test do
  gem 'rspec'
  gem 'shoulda'
  gem 'factory_girl'
  gem 'database_cleaner'
  gem 'rack-test'
end
```

В терминале, перейдите в папку `noughts_and_crosses` и выполните `bundle install`. Мы разделили гемы на группы: некоторые из них нужны только в `test` режиме, некоторые только в` test` и `development`.

Обратите внимание на гем `sinatra-activerecord`, он автоматически устанавливает соединение с базой данных с помощью конфигурационного файла `config/database.yml` и добавляет `rake` задачи для управления базой данных.

Теперь создайте файл `application.rb` в папке `noughts_and_crosses`, это будет основной файл сервиса.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/models/*.rb")].each do |f|
  autoload File.basename(f, '.rb').classify.to_sym, f
end
Dir[File.join(settings.root, "app/controllers/*.rb")].each { |f| require f }

before do
  content_type :txt
end

error(ActiveRecord::RecordNotFound) { [404, "There is no Game with provided id"] }
error(ActiveRecord::RecordInvalid) { [422, env['sinatra.error'].record.errors.full_messages.join("\n")] }
error { "An internal server error occurred. Please try again later." }
```

Давайте пройдемся по коду по маленьким частям

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"
```

Здесь мы загружаем все гемы для используемого режима (`environment`). Мы можем запустить сервис в различных средах, передавая параметр `RACK_ENV`. По умолчанию используется `development`

    RACK_ENV=production ruby application.rb

Далее, задаем корневую папку.

```ruby
set :root, File.dirname(__FILE__)
```

После этого мы можем ссылаться на корневую папку как `settings.root`.

Настройка `logger` для отслеживания доступа:

```ruby
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }
```

Мы должны создать папку `log` внутри папки `noughts_and_crosses`. Если вы используете `git` вы можете создать файл `.gitignore` в папке `noughts_and_crosses` со следующим содержимым:

    log/*.log

Это предотвращает попадание `log`-файлов в репозиторий. Также мы можем создать пустой файл `.gitkeep` или просто `.keep` внутри папки `log` чтобы гарантировать, что пустая папка `log` попадет в репозиторий.

Мы будем хранить файлы для моделей внутри папки `models` внутри папки `app` (внутри папки `noughts_and_crosses`). Мы используем `autoload`, чтобы подключить все эти файлы. Это означает, что файл на самом деле загружается в память только после первой попытки использовать класс. Также мы будем хранить все роуты (`routes`) в папке `app/controllers`. Все роуты, связанные с одной моделью будут находиться в одном файле. И в одном файле будут роуты, связанные только с одной моделью.

```ruby
Dir[File.join(settings.root, "app/models/*.rb")].each do |f|
  autoload File.basename(f, '.rb').classify.to_sym, f
end
Dir[File.join(settings.root, "app/controllers/*.rb")].each { |f| require f }
```

В этом веб-сервисе у нас будет только одна модель - `Game` и только один контроллер.

Тип содержимого ответов HTTP будет обычный текст. Далее, добавление HTTP заголовка "Content-Type: text/plain" с помощью метода `content_type`.

```ruby
before do
  content_type :txt
end
```

Добавление логики для обработки некоторых ошибок и возвращения соответствующего кода состояния HTTP: 404 - для `record not found`, и 422 - за ошибок валидации.

```ruby
error(ActiveRecord::RecordNotFound) { [404, "There is no Game with provided id"] }
error(ActiveRecord::RecordInvalid) { [422, env['sinatra.error'].record.errors.full_messages.join("\n")] }
error { "An internal server error occurred. Please try again later." }
```

Последняя строка кода обрабатывает все другие неожиданные ошибки и возвращает 500 код состояния HTTP.

Теперь, пожалуйста, создайте папки `app/models` и `app/controllers`. Создайте папку `config` внутри `noughts_and_crosses` с файлом `database.yml` в ней. Этот файл используется гемом ` sinatra-activerecord` по умолчанию. Вот мои настройки (измените `username`):

```yaml
development:
  adapter: postgresql
  encoding: unicode
  database: noughts_and_crosses_development
  username: alex

test:
  adapter: postgresql
  encoding: unicode
  database: noughts_and_crosses_test
  username: alex
```

Создайте папку `db` внутри корневой папки и папку `migrate` внутри папки `db`. Создайте папку `spec` для тестов, и в ней файл `spec_helper.rb`, папки `acceptance`,` factories`, `models`.

Вот `spec_helper.rb`

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
```

Вот вся структура веб-сервиса на этот момент:

![Basic gem structure](../static/images/noughts_and_crosses_structure.png)

## <a name="create-migration"></a>Создание миграции

Теперь нам нужно создать модель. Но сначала мы должны создать миграцию. Для этого мы используем `rake` задачу из гема `sinatra-activerecord`. Создайте, пожалуйста, файл `Rakefile` в папке `noughts_and_crosses`.

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'
```

Затем перейдите в папку `noughts_and_crosses` в терминале и создайте базу данных и файл миграции

    $ rake db:create
    $ rake db:create_migration NAME=create_games
    Loaded development environment
    db/migrate/20150129204548_create_games.rb

Затем измените созданный файл:

```ruby
class CreateGames < ActiveRecord::Migration
  def change
    create_table :games do |t|
      t.string :board, null: false, default: ',,,,,,,,'
      t.timestamps null: false
    end
  end
end
```

Перейдите к терминалу снова и выполните миграцию

    $ rake db:migrate
    Loaded development environment
    == 20150129204548 CreateGames: migrating ======================================
    -- create_table(:games)
       -> 0.0105s
    == 20150129204548 CreateGames: migrated (0.0107s) =============================

Мы будем хранить игровое поле в виде строки (разделенные запятой символы "X", "O" или ""), мы также можем хранить его в любом другом строковом формате или в массиве, у вас всегда есть выбор.

## <a name="create-model"></a>Создание модели

Реализация игры на самом деле не очень важна, потому что мы больше ориентированы на поведение верхнего уровня. В любом случае ниже приведена моя версия реализации класса `Game` (файл `app/models/game.rb`).

```ruby
class Game < ActiveRecord::Base
  before_update :make_a_move

  validates_format_of :board, with: /\A(?:[XO]?,){8}[XO]?\Z/
  validates :move, presence: true, on: :update
  validates :move, inclusion: { in: [*0..8], message: 'is out of the board',
    allow_nil: true }, on: :update
  validate :ensure_geme_not_finished, on: :update
  validate :ensure_move_allowed, on: :update

  attr_accessible :move
  attr_reader :move

  def move=(index)
    @move = index.to_i if index.present?
  end

  def won?
    lines.include? "XXX"
  end

  def lost?
    lines.include? "OOO"
  end

  def finished?
    won? or lost? or cells.none?(&:blank?)
  end

  def status
    return 'In Progress' unless finished?
    won? ? 'Won' : (lost? ? 'Lost' : 'Draw')
  end

  def cells
    @cells ||= board.split(',', 9)
  end

private
  def part(*indexes)
    cells.values_at(*indexes).join
  end

  def lines
    [part(0,1,2), part(3,4,5), part(6,7,8), part(0,3,6),
      part(1,4,7), part(2,5,8), part(0,4,8), part(2,4,6)]
  end

  def ensure_geme_not_finished
    errors.add(:base, "Game is finished.") if finished?
  end

  def ensure_move_allowed
    errors.add(:move, "not allowed, cell is not empty.") if move && cells[move] != ''
  end

  def make_a_move
    cells[move] = 'X'
    unless won?
      empty_indexes = [*0..8].select { |ind| cells[ind] == '' }
      cells[empty_indexes.sample] = 'O'
    end
    self.board = cells.join(',')
  end
end
```

И тесты модели в файле `spec/models/game_spec.rb`

```ruby
require "spec_helper"

describe Game do
  describe "validations" do
    it { is_expected.not_to allow_value('').for(:board) }
    it { is_expected.to allow_value(',,,,,,,,').for(:board) }
    it { is_expected.to allow_value(',,X,X,,,O,,').for(:board) }
    it { is_expected.to allow_value('O,,X,X,,,O,,').for(:board) }
    it { is_expected.not_to allow_value(',,x,,,,,,').for(:board) }
    it { is_expected.not_to allow_value(',O,,X,X,,,O,,').for(:board) }
    it { is_expected.not_to allow_value('O,,X,X,,,O,').for(:board) }

    it { should validate_inclusion_of(:move).in_array([*0..8]).on(:update) }

    it "can not update finished game" do
      game = create(:game, board: 'X,,O,O,X,,,,X')
      expect { game.update_attributes!(move: '5') }.to raise_error
      expect(game.errors.full_messages).to include "Game is finished."
    end

    it "can not make a move at busy cell" do
      game = create(:game, board: ',,O,,X,,,,')
      expect { game.update_attributes!(move: '4') }.to raise_error
      expect(game.errors.full_messages).to include "Move not allowed, cell is not empty."
    end

    it "can make a move at free cell if geme is not finished" do
      game = create(:game, board: ',,X,O,X,,O,,')
      expect { game.update_attributes!(move: '5') }.not_to raise_error
    end

    it "records player move" do
      game = create(:game, board: 'O,,X,O,X,,,,')
      game.update_attributes!(move: '5')
      expect(game.board.count('X')).to eq 3
    end

    it "makes and records computer move after player move if game not won" do
      game = create(:game, board: 'O,,X,O,X,,,,')
      game.update_attributes!(move: '5')
      expect(game.board.count('O')).to eq 3
    end

    it "does not make computer move after player move if game won" do
      game = create(:game, board: 'O,,X,O,X,,,,')
      game.update_attributes!(move: '6')
      expect(game.board.count('O')).to eq 2
    end
  end

  describe 'assignament' do
    it { is_expected.not_to allow_mass_assignment_of(:board) }
    it { is_expected.not_to allow_mass_assignment_of(:cells) }
    it { is_expected.to allow_mass_assignment_of(:move) }
  end

  describe "creation" do
    specify "new game populated with empty board before create" do
      expect(subject.board).to eq ",,,,,,,,"
    end
  end

  describe "#won?" do
    it "is true if at least one of the board lines is filled with crosses" do
      expect(build(:game, board: 'X,,O,O,X,,,,X')).to be_won
      expect(build(:game, board: ',,O,O,,,X,X,X')).to be_won
    end

    it "is false none of the board lines is filled with crosses" do
      expect(build(:game, board: ',,,,,,,,')).not_to be_won
      expect(build(:game, board: 'X,O,X,O,X,X,O,,O')).not_to be_won
    end
  end

  describe "#lost?" do
    it "is true if at least one of the board lines is filled with noughts" do
      expect(build(:game, board: ',O,X,X,O,,,O,X')).to be_lost
      expect(build(:game, board: 'X,O,O,,O,X,O,X,X')).to be_lost
    end

    it "is false none of the board lines is filled with noughts" do
      expect(build(:game, board: ',,,,,,,,')).not_to be_lost
      expect(build(:game, board: 'X,O,X,X,O,,O,X,')).not_to be_lost
    end
  end

  describe "#finished?" do
    it "is true if at least one of the board lines is filled with three noughts or with three crosses (won or lost)" do
      expect(build(:game, board: 'X,,O,O,X,,,,X')).to be_finished
      expect(build(:game, board: ',,O,O,,,X,X,X')).to be_finished
      expect(build(:game, board: ',O,X,X,O,,,O,X')).to be_finished
      expect(build(:game, board: 'X,O,O,,O,X,O,X,X')).to be_finished
    end

    it "is false none of the board lines is filled with three noughts or with three crosses (neither won or lost)" do
      expect(build(:game, board: ',,,,,,,,')).not_to be_finished
      expect(build(:game, board: 'X,O,X,O,X,X,O,,O')).not_to be_finished
      expect(build(:game, board: 'X,O,X,X,O,,O,X,')).not_to be_finished
    end
  end
end
```

Также фабрика для тестов, файл `spec/factories/game.rb`

```ruby
FactoryGirl.define do
  factory :game do
  end
end
```

Необходимо запустить миграцию в `test` режиме (`test` environment)

    $ RACK_ENV=test rake db:migrate
    Loaded test environment
    == 20150129204548 CreateGames: migrating ======================================
    -- create_table(:games)
       -> 0.0087s
    == 20150129204548 CreateGames: migrated (0.0090s) =============================

Теперь вы можете запустить тесты модели.

    $ rspec

## <a name="create-controller-and-acceptance-tests"></a>Создание контроллера и `acceptance` тестов

Наконец мы создаем роуты для игры! Создайте пожалуйста файл `app/controllers/games_controller.rb` со следующим содержимым.

```ruby
post "/api/v1/games.txt" do
  @game = Game.create
  status 201
  erb :game
end

get "/api/v1/games/:id.txt" do
  @game = Game.find(params[:id])
  erb :game
end

put "/api/v1/games/:id.txt" do
  @game = Game.find(params[:id])
  @game.update_attributes!(params[:game])
  erb :game
end

delete "/api/v1/games/:id.txt" do
  @game = Game.find(params[:id])
  @game.destroy
end

template :game do
  (<<-GAME).gsub(/^ {4}/, '')
    <% cells = @game.cells.map { |c| c == '' ? ' ' : c } %>
    Game #<%= @game.id %>
    Status: <%= @game.status %>

     <%= cells.values_at(0,1,2).join(' | ') %>
    -----------
     <%= cells.values_at(3,4,5).join(' | ') %>
    -----------
     <%= cells.values_at(6,7,8).join(' | ') %>
  GAME
end
```

Мы называем этот файл - контроллер, но это только куча роутов (и шаблон), связаных с API для управления игрой. Теперь мы готовы к запуску нашего сервиса - с помощью команды `ruby application.rb` из папки `noughts_and_crosses`.

    $ ruby application.rb

Кроме того, мы можем создать файл `config.ru`

```ruby
require_relative 'application.rb'
run Sinatra::Application
```

И запускать сервис следующей командой

    $ rackup -p 4567

Этот файл будет, вероятно, необходим для развертывания приложения в `production`. Расширение "ru" - это сокращение от "rack up". Вы можете убедиться, что веб-сервис работает должным образом, проверив его при помощи `curl`. Для того, чтобы обеспечить правильное функционирование веб-сервиса в будущем, весьма желательно иметь `acceptance` тесты.

Создайте файл `spec/acceptance/games_spec.rb` (или `spec/features/games_spec.rb`) со следующим содержимым:

```ruby
require "spec_helper"

describe "Games", type: :request do
  describe "POST /api/v1/games.txt" do
    let(:game) { Game.last }

    it "craetes game with empty board and responds with text representation of game" do
      post "/api/v1/games.txt"
      expect(last_response.status).to eq 201
      expect(last_response.body).to eq (<<-GAME).gsub(/^ {8}/, '')
        Game ##{game.id}
        Status: In Progress

           |   |
        -----------
           |   |
        -----------
           |   |
      GAME
    end
  end

  describe "GET /api/v1/games/:id.txt" do
    let!(:game) { create(:game, board: ",,X,O,X,,O,,") }

    it "responds with ok status and text representation of game if game exists" do
      get "/api/v1/games/#{game.id}.txt"
      expect(last_response).to be_ok
      expect(last_response.body).to eq (<<-GAME).gsub(/^ {8}/, '')
        Game ##{game.id}
        Status: In Progress

           |   | X
        -----------
         O | X |
        -----------
         O |   |
      GAME
    end

    it "responds with 404 status and error message if game does not exist" do
      get "/api/v1/games/234.txt"
      expect(last_response.status).to eq 404
      expect(last_response.body).to eq "There is no Game with provided id"
    end
  end

  describe "PUT /api/v1/games/:id.txt" do
    let!(:game) { create(:game, board: ",O,X,O,X,,,,") }

    it "allows player to make a move and responds with text representation of game" do
      put "/api/v1/games/#{game.id}.txt", game: { move: 6 }
      expect(last_response).to be_ok
      expect(last_response.body).to eq (<<-GAME).gsub(/^ {8}/, '')
        Game ##{game.id}
        Status: Won

           | O | X
        -----------
         O | X |
        -----------
         X |   |
      GAME
    end

    it "responds with 404 status and error message if game does not exist" do
      put "/api/v1/games/234.txt"
      expect(last_response.status).to eq 404
      expect(last_response.body).to eq "There is no Game with provided id"
    end

    it "responds with 422 status and error message if move not provided" do
      put "/api/v1/games/#{game.id}.txt"
      expect(last_response.status).to eq 422
      expect(last_response.body).to eq "Move can't be blank"
    end

    it "responds with 422 status and error message if move out of the board" do
      put "/api/v1/games/#{game.id}.txt", game: { move: -1 }
      expect(last_response.status).to eq 422
      expect(last_response.body).to eq "Move is out of the board"
    end

    it "responds with 422 status and error message when trying to make a move on a busy cell" do
      put "/api/v1/games/#{game.id}.txt", game: { move: 2 }
      expect(last_response.status).to eq 422
      expect(last_response.body).to eq "Move not allowed, cell is not empty."
    end
  end

  describe "DELETE /api/v1/games/:id.txt" do
    let!(:game) { create(:game, board: ",,X,O,X,,O,,") }

    it "responds with ok status and text representation of game if game exists" do
      delete "/api/v1/games/#{game.id}.txt"
      expect(last_response).to be_ok
      expect(Game.all).to be_empty
    end

    it "responds with 404 status and error message if game does not exist" do
      delete "/api/v1/games/234.txt"
      expect(last_response.status).to eq 404
      expect(last_response.body).to eq "There is no Game with provided id"
    end
  end
end
```

Затем вы можете запустить тесты

    $ rspec
    Loaded test environment
    ..................................

    Finished in 0.4139 seconds (files took 1.8 seconds to load)
    34 examples, 0 failures

Все хорошо, теперь можно идти в паб. Ой, подождите! Ещё две вещи...

## <a name="create-console"></a>Создать консоль

Мы можем создать `development` консоль для быстрого доступа к данным. Создайте, пожалуйста, папку `script` в корневой папке сервиса и файл `console` в ней:

```ruby
#!/bin/bash
bundle exec irb -r ./application.rb
```

Сделайте файл исполняемым (для Unix-подобных систем)

    $ chmod +x script/console

Мы можем использовать его в `development` среде по умолчанию

    $ script/console
    Loaded development environment
    irb >

Или в `test` или` production` режиме

    $ RACK_ENV=test script/console
    Loaded test environment
    irb >

Для выхода наберите `quit` и нажмите Enter (или` Ctrl` + `c`), это обычная консоль `irb` с загруженным файлом `application.rb`.

## <a name="add-custom-rake-task"></a>Добавление `rake` задачи

Хорошо, что делать, если нам нужно создать новую `rake` задачу? Например, чтобы мы могли бы быть в состоянии удалять все записи игр, которые старше, чем один день.

Для начала, создадим индекс базы данных для колонки `created_at` для лучшей производительности. Предполагая, что предыдущая миграция была уже выполнена на `production` сервере, создадим новую миграцию для добавления индекса.

    $ rake db:create_migration NAME=add_index_on_games_created_at
    Loaded development environment
    db/migrate/20150129215128_add_index_on_games_created_at.rb

Измените созданный файл миграции.

```ruby
class AddIndexOnGamesCreatedAt < ActiveRecord::Migration
  def change
    add_index :games, :created_at
  end
end
```

Выполните миграцию в `development` среде и `test` среде.

    $ rake db:migrate
    == 20150129215128 AddIndexOnGamesCreatedAt: migrating =========================
    -- add_index(:games, :created_at)
       -> 0.0051s
    == 20150129215128 AddIndexOnGamesCreatedAt: migrated (0.0054s) ================

    $ RACK_ENV=test rake db:migrate
    == 20150129215128 AddIndexOnGamesCreatedAt: migrating =========================
    -- add_index(:games, :created_at)
       -> 0.0052s
    == 20150129215128 AddIndexOnGamesCreatedAt: migrated (0.0054s) ================

Создайте папку `lib` в папке `noughts_and_crosses`, создайте папку `tasks` в папке `lib` и создайте файл `delete_old_games.rake` в папке` tasks` (`lib/tasks/delete_old_games.rake`).

```ruby
desc 'Delete all games that are older that one day'
task :delete_old_games do
  Game.where(Game.arel_table[:created_at].lt(1.day.ago)).delete_all
end
```

Добавьте одну строку в `Rakefile` для подключения всех задач из папки `lib/tasks`.

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'
Dir.glob('lib/tasks/**/*.rake').each { |r| load r }
```

И это все! Убедитесь, что новая `rake` задача доступна.

    $ rake -T
    Loaded development environment
    rake db:create              # Creates the database from DATABASE_URL or config/database.yml for the current RAILS_ENV (use db:create:all to create all databases in the...
    rake db:create_migration    # Create a migration (parameters: NAME, VERSION)
    rake db:drop                # Drops the database from DATABASE_URL or config/database.yml for the current RAILS_ENV (use db:drop:all to drop all databases in the config)
    rake db:fixtures:load       # Load fixtures into the current environment's database
    rake db:migrate             # Migrate the database (options: VERSION=x, VERBOSE=false, SCOPE=blog)
    rake db:migrate:status      # Display status of migrations
    rake db:rollback            # Rolls the schema back to the previous version (specify steps w/ STEP=n)
    rake db:schema:cache:clear  # Clear a db/schema_cache.dump file
    rake db:schema:cache:dump   # Create a db/schema_cache.dump file
    rake db:schema:dump         # Create a db/schema.rb file that is portable against any DB supported by AR
    rake db:schema:load         # Load a schema.rb file into the database
    rake db:seed                # Load the seed data from db/seeds.rb
    rake db:setup               # Create the database, load the schema, and initialize with the seed data (use db:reset to also drop the database first)
    rake db:structure:dump      # Dump the database structure to db/structure.sql
    rake db:structure:load      # Recreate the databases from the structure.sql file
    rake db:version             # Retrieves the current schema version number
    rake delete_old_games       # Delete all games that are older that one day

Запуск задачи

    $ rake delete_old_games

## <a name="summary"></a>Резюме

В этой главе были в целом рассмотрены многие аспекты структуры веб-сервиса. Мы использовали гем [sinatra-activerecord](https://github.com/janko-m/sinatra-activerecord), который добавляет в `sinatra` `rake` задачи для управления базой данных и автоматически устанавливает соединение с базой данных. Этот гем является расширением `sinatra`. Возможно вы захотите подробнее ознакомится с [расширениями sinatra](http://www.sinatrarb.com/extensions.html)
