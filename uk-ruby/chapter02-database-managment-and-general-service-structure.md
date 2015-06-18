Глава #2. Управління базою даних і загальна структура програми
==============================================================

У цій главі ми створимо сервіс для простої гри Tic Tac Toe (хрестики-нулики), ми будемо приділяти більше уваги структурі програми і задачам управління базою даних: для створення бази даних, для створення міграції, виконання та скасування міграції.

Ми будемо зберігати ігрові записи в реляційній базі даних (а саме PostgreSQL, але ви можете використовувати й інші, такі як SQLite або MySQL). Запис гри зберігає свою дошку (поле гри), після створення дошка порожня. Веб-сервіс дозволяє гравцеві робити хід на клітинці дошки (поля), після чого сервіс робить власний хід і повертає оновлене уявлення гри в текстовому форматі.

Ось уявлення порожнього ігрового поля:

       |   |
    -----------
       |   |
    -----------
       |   |

А тут, як це може виглядати після першого ходу:

       |   |
    -----------
     O | X |
    -----------
       |   |

## <a name="service-interface"></a>Інтерфейс веб-сервісу

Гравець повинен бути в змозі створити гру відправивши `POST` запит на URL `"games.txt"`.

    $ curl -X POST "localhost:4567/api/v1/games.txt"
    Game #1
    Status: In Progress

       |   |
    -----------
       |   |
    -----------
       |   |

Також гравець повинен бути в змозі зробити хід, відправивши `PUT` запит на URL конкретної гри (з ID) з номером комірки, де гравець бажає поставити хрестик (символ "X").

    $ curl -X PUT "localhost:4567/api/v1/games/1.txt?game%5Bmove%5D=4"
    Game #1
    Status: In Progress

       |   |
    -----------
     O | X |
    -----------
       |   |

Сервіс робить свій хід - комп'ютер ставить "О" на порожню комірку (якщо гра не закінчується після ходу гравця), і повідомляє про стан гри: "В процесі", "Виграно", "Програно", "Нічия". Зверніть увагу, що гравець передає параметри гри, в даному випадку це GET параметри (частина URL), але взагалі це мають бути параметри POST (передані в тілі запиту HTTP) - довжина URL обмежується залежно від веб-сервера, так що в цілому ми повинні використовувати POST. Поки ж будемо використовувати GET для деякої простоти.

Параметри ігри - це сховище ключів і значень, яке містить тільки один ключ - "move" (хід), значення - це номер комірки, в якій гравець хоче поставити хрестик (символ "X"). Комірка повинна бути порожньою для того, щоб була можливість зробити в неї хід. Комірки пронумеровані від 0 до 9 (це не правила гри, а наше уявлення про поле гри, для того, щоб мати можливість зробити хід):

     0 | 1 | 2
    -----------
     3 | 4 | 5
    -----------
     6 | 7 | 8

Якщо гра не закінчена гравець може зробити новий хід.

Гра завершена якщо вона виграна або програна, чи не існує більше порожніх клітинок на полі. Гра вважається виграною (гравцем), якщо три хрестики розміщені на одній лінії (горизонтальній, вертикальній або діагональній). Гра вважається програною (гравцем), якщо три нулика знаходяться на одній лінії.

## <a name="games-model-interface"></a>Інтерфейс моделі гри

Наша ігрова модель повинна виглядати себе таким чином:

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

## <a name="create-skeleton-of-the-service"></a>Створення структури сервісу

Створіть будь ласка теку `noughts_and_crosses` і файл `Gemfile` в ній зі списком необхідних гемів:

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

У терміналі, перейдіть в теку `noughts_and_crosses` і виконайте` bundle install`. Ми розділили геми на групи: деякі з них потрібні тільки в `test` режимі, деякі тільки В `test` та `development`.

Зверніть увагу на гем `sinatra-activerecord`, він автоматично встановлює з'єднання з базою даних за допомогою конфігураційного файлу `config/database.yml` і додає `rake` завдання для управління базою даних.

Тепер створіть файл `application.rb` в теці `noughts_and_crosses`, це буде основний файл сервісу.

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

Давайте пройдемося по коду по маленьким частинам

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"
```

Тут ми завантажуємо всі геми для використовуваного режиму (`environment`). Ми можемо запустити сервіс в різних середовищах, передаючи параметр `RACK_ENV`. За замовчуванням використовується `development`

    RACK_ENV=production ruby application.rb

Далі, задаємо кореневу теку.

```ruby
set :root, File.dirname(__FILE__)
```

Після цього ми можемо посилатися на кореневу теку як `settings.root`.

Налаштування `logger` для відстеження доступу:

```ruby
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }
```

Ми повинні створити теку `log` всередині теки `noughts_and_crosses`. Якщо ви використовуєте `git` ви можете створити файл `.gitignore` в теці `noughts_and_crosses` з наступним вмістом:

    log/*.log

Це запобігає потраплянню `log`-файлів в репозиторій. Також ми можемо створити порожній файл `.gitkeep` або просто` .keep` всередині теки `log` щоб гарантувати, що порожня папка`log` потрапить в репозиторій.

Ми будемо зберігати файли для моделей всередині теки `models` всередині теки `app` (всередині теки `noughts_and_crosses`). Ми використовуємо `autoload`, щоб підключити всі ці файли. Це означає, що файл насправді завантажується в пам'ять тільки після першої спроби використати клас. Також ми будемо зберігати всі роути (`routes`) в теці `app/controllers`. Всі роути, пов'язані з однією моделлю будуть знаходитися в одному файлі. І в одному файлі будуть роути, пов'язані тільки з однією моделлю.

```ruby
Dir[File.join(settings.root, "app/models/*.rb")].each do |f|
  autoload File.basename(f, '.rb').classify.to_sym, f
end
Dir[File.join(settings.root, "app/controllers/*.rb")].each { |f| require f }
```

У цьому веб-сервісі у нас буде тільки одна модель - `Game` і тільки один контролер.

Тип вмісту відповідей HTTP буде звичайний текст. Далі, додавання HTTP заголовка "Content-Type: text/plain" за допомогою методу `content_type`.

```ruby
before do
  content_type :txt
end
```

Додавання логіки для обробки деяких помилок і повернення відповідного коду стану HTTP: 404 - для `record not found`, і 422 - за помилок валідації.

```ruby
error(ActiveRecord::RecordNotFound) { [404, "There is no Game with provided id"] }
error(ActiveRecord::RecordInvalid) { [422, env['sinatra.error'].record.errors.full_messages.join("\n")] }
error { "An internal server error occurred. Please try again later." }
```

Останній рядок коду обробляє всі інші несподівані помилки і повертає 500 код стану HTTP.

Тепер, будь ласка, створіть теки `app/models` і `app/controllers`. Створіть теку `config` всередині `noughts_and_crosses` з файлом `database.yml` в ній. Цей файл використовується гемом `sinatra-activerecord` за замовчуванням. Ось мої налаштування (змініть `username`):

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

Створіть теку `db` всередині кореневої теки і теку `migrate` всередині теки `db`. Створіть теку `spec` для тестів, і в ній файл `spec_helper.rb`, теки `acceptance`, `factories`, `models`.

Ось `spec_helper.rb`

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

Ось вся структура веб-сервісу на цей момент:

![Basic gem structure](../static/images/noughts_and_crosses_structure.png)

## <a name="create-migration"></a>Создание миграции

Тепер нам потрібно створити модель. Але спочатку ми повинні створити міграцію. Для цього ми використаємо `rake` задачу з гему `sinatra-activerecord`. Створіть, будь ласка, файл `Rakefile` в теці `noughts_and_crosses`.

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'
```

Потім перейдіть в теку `noughts_and_crosses` в терміналі і створіть базу даних і файл міграції

    $ rake db:create
    $ rake db:create_migration NAME=create_games
    Loaded development environment
    db/migrate/20150129204548_create_games.rb

Потім змініть створений файл:

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

Перейдіть до терміналу знову і виконайте міграцію

    $ rake db:migrate
    Loaded development environment
    == 20150129204548 CreateGames: migrating ======================================
    -- create_table(:games)
       -> 0.0105s
    == 20150129204548 CreateGames: migrated (0.0107s) =============================

Ми будемо зберігати ігрове поле у вигляді рядка (розділені комою символи "X", "O" або ""), ми також можемо зберігати його в будь-якому іншому строковому форматі або в масиві, у вас завжди є вибір.

## <a name="create-model"></a>Створення моделі

Реалізація гри насправді не дуже важлива, тому що ми більше орієнтовані на поведінку верхнього рівня. У будь-якому випадку нижче приведена моя версія реалізації класу `Game` (файл `app/models/game.rb`).

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

І тести моделі у файлі `spec/models/game_spec.rb`

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

Також фабрика для тестів, файл `spec/factories/game.rb`

```ruby
FactoryGirl.define do
  factory :game do
  end
end
```

Необхідно запустити міграцію в `test` режимі (`test` environment)

    $ RACK_ENV=test rake db:migrate
    Loaded test environment
    == 20150129204548 CreateGames: migrating ======================================
    -- create_table(:games)
       -> 0.0087s
    == 20150129204548 CreateGames: migrated (0.0090s) =============================

Тепер ви можете запустити тести моделі.

    $ rspec

## <a name="create-controller-and-acceptance-tests"></a>Створення контролера і `acceptance` тестів

Нарешті ми створюємо роути для гри! Створіть будь ласка файл `app/controllers/games_controller.rb` з наступним вмістом.

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

Ми називаємо цей файл - контролер, але це тільки купа роутів (і шаблон), пов'язаних з API для управління грою. Тепер ми готові до запуску нашого сервісу - за допомогою команди `ruby application.rb` з теки `noughts_and_crosses`.

    $ ruby application.rb

Крім того, ми можемо створити файл `config.ru`

```ruby
require_relative 'application.rb'
run Sinatra::Application
```

І запускати сервіс наступною командою

    $ rackup -p 4567

Цей файл буде, ймовірно, необхідний для розгортання програми в `production`. Розширення "ru" - це скорочення від "rack up". Ви можете переконатися, що веб-сервіс працює належним чином, перевіривши його за допомогою `curl`. Для того, щоб забезпечити правильне функціонування веб-сервісу в майбутньому, дуже бажано мати `acceptance` тести.

Створіть файл `spec/acceptance/games_spec.rb` (або `spec/features/games_spec.rb`) з наступним вмістом:

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

Потім ви можете запустити тести

    $ rspec
    Loaded test environment
    ..................................

    Finished in 0.4139 seconds (files took 1.8 seconds to load)
    34 examples, 0 failures

Все добре, тепер можна йти в паб. Ой, зачекайте! Ще дві речі ...

## <a name="create-console"></a>Створити консоль

Ми можемо створити `development` консоль для швидкого доступу до даних. Створіть, будь ласка, теку `script` в кореневій теці сервісу і файл `console` в ній:

```ruby
#!/bin/bash
bundle exec irb -r ./application.rb
```

Зробіть файл виконуваним (для Unix-подібних систем)

    $ chmod +x script/console

Ми можемо використовувати його в `development` середовищі за замовчуванням

    $ script/console
    Loaded development environment
    irb >

Або в `test` або `production` режимі

    $ RACK_ENV=test script/console
    Loaded test environment
    irb >

Для виходу наберіть `quit` і натисніть Enter (або `Ctrl` + `c`), це звичайна консоль `irb` з завантаженим файлом `application.rb`.

## <a name="add-custom-rake-task"></a>Створення `rake` задачі

ДДобре, що робити, якщо нам потрібно створити нову `rake` завдання? Наприклад, щоб ми могли би бути в змозі видаляти всі записи ігор, які старші, ніж один день.

Для початку, створимо індекс бази даних для колонки `created_at` для кращої продуктивності. Припускаючи, що попередня міграція була вже виконана на `production` сервері, створимо нову міграцію для додавання індексу.

    $ rake db:create_migration NAME=add_index_on_games_created_at
    Loaded development environment
    db/migrate/20150129215128_add_index_on_games_created_at.rb

Змініть створений файл міграції.

```ruby
class AddIndexOnGamesCreatedAt < ActiveRecord::Migration
  def change
    add_index :games, :created_at
  end
end
```

Виконайте міграцію в `development` середовищі і `test` середовищі.

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

Створіть теку `lib` в теці `noughts_and_crosses`, створіть теку `tasks` в теці `lib` і створіть файл `delete_old_games.rake` в теці `tasks` (`lib/tasks/delete_old_games.rake`).

```ruby
desc 'Delete all games that are older that one day'
task :delete_old_games do
  Game.where(Game.arel_table[:created_at].lt(1.day.ago)).delete_all
end
```

Додайте один рядок в `Rakefile` для підключення всіх завдань з теки `lib/tasks`.

```ruby
require_relative 'application'
require 'sinatra/activerecord/rake'
Dir.glob('lib/tasks/**/*.rake').each { |r| load r }
```

І це все! Переконайтеся, що нова `rake` задача доступна.

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

Запуск задачі

    $ rake delete_old_games

## <a name="summary"></a>Резюме

У цьому розділі були в цілому розглянуті багато аспектів структури веб-сервісу. Ми використали гем [sinatra-activerecord](https://github.com/janko-m/sinatra-activerecord), що додає в `sinatra` `rake` задачі для управління базою даних і автоматично встановлює з'єднання з базою даних. Цей гем є розширенням `sinatra`. Можливо ви захочете докладніше ознайомитись з [розширеннями sinatra](http://www.sinatrarb.com/extensions.html)
