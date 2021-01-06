# AnyFixture

Фикстуры в Rails работает ни в пример быстрее фабрик, однако за скорость приходится платить меньшей гибкостью и временем на поддержку.
Прежде всего это связано с тем, что фикстуры используют декларативный подход (YAML файлы с описанием данных).

Мы предлагаем альтернативный подход к генерации глобального состояния для тестов – AnyFixture.

AnyFixture позволяет использовать любой блок кода в качестве фикстуры. При этом не нужно думать о _чистоте_ базы данных — AnyFixture позаботится о _зачистке_ затронутых таблиц после выполнения всех тестов.

Рассмотрим пример:

```ruby
# Мы рекомендуем использовать контексты для подключения фикстур
RSpec.shared_context "account" do
  # Внимание! Вызов AnyFixture должен происходит вне транзакций;
  # для этого мы используем before(:all)
  before(:all) do
    # Имя ("account") — это уникальный идентификатор фикстуры
    @account = TestProf::AnyFixture.register(:account) do
      # Внутри блока вы можете гененировать данные любым удобным вам способом.
      # Например, с помощью FactoryBot
      FactoryBot.create(:account)

      # или Fabrication
      Fabricate(:account)

      # или ActiveRecord
      Account.create!(name: "test")
    end
  end

  # Повторный вызов .register вернёт ранее созданный объект
  let(:account) { TestProf::AnyFixture.register(:account) }

  # Рекомендуется «пересоздавать» объект для избежания утечки состояния в тестах
  let(:account) { Account.find(TestProf::AnyFixture.register(:account).id) }
end

RSpec.configure do |config|
  config.include_context "account", account: true
end

# Активировать фикстуру можно, добави тег
describe UsersController, :account do
  # ...
end

# Этот тест будет использовать тот же объект аккаунта
describe PostsController, :account do
  # ...
end
```

[Пример из боевого проекта](http://bit.ly/any-fixture).

## Инструкция

### RSpec

В вашем `spec_helper.rb` (или `rails_helper.rb` при наличии) добавьте строчку:

```ruby
require "test_prof/recipes/rspec/any_fixture"
```

Теперь модуль `TestProf::AnyFixture` доступен в ваших тестах.

### Minitest

При использовании с Minitest вам необходимо вручную добавить код для зачистки базы после выполнения тестов. Например:

```ruby
# test_helper.rb

require "test_prof/any_fixture"

at_exit { TestProf::AnyFixture.clean }
```

## DSL

AnyFixture предоставляет дополнительный DSL (_синтаксический сахар_) для более лаконичного определения фикстур:

```ruby
require "test_prof/any_fixture/dsl"

# Подключаем DSL с помощью refinements
using TestProf::AnyFixture::DSL

# Теперь вам доступен метод `fixture` (псевдоним для `TestProf::AnyFixture.register`)
before(:all) { fixture(:account) { create(:account) } }

# Он также может быть использован для доступа к уже созданному объекту
let(:account) { fixture(:account) }
```

## `ActiveRecord#refind`

Для пересоздания объектов ActiveRecord (и избежания утечек состояния) вы можете использовать метод `#refind`, доступный через refinements:

```ruby
# вместо
let(:account) { Account.find(fixture(:account).id) }

# загружаем refinement
require "test_prof/ext/active_record_refind"

using TestProf::Ext::ActiveRecordRefind

let(:account) { fixture(:account).refind }
```

## Временный откат фикстур

Некоторые тесты могут требовать _чистой базы данных_, а значит их выполнение вместе с AnyFixture может приводить к ошибкам.

Вы можете «отключить» фикстуры _локально_ (т.е., очистить таблицы в БД), используя тег `:with_clean_fixture`:

```ruby
context "global state", :with_clean_fixture do
  # либо подключив контекст явно
  # include_context "any_fixture:clean"

  specify "table is empty or smth like this" do
    # ...
  end
end
```

Как это работает? Группа тестов оборачивается в транзакцию (с помощью [`before_all`](./before_all.md)), а затем вызывается `TestProf::AnyFixture.clean`.

Зачистка — это довольно тяжёлая операция. Поэтому рекомендуется избегать подобных ситуаций, когда это возможно.

## Статистика использования

AnyFixture позволяет выводит информацию о том, как часто фикстуры были использованы и сколько времени было сэкономлено:

```sh
[TEST PROF INFO] AnyFixture usage stats:

       key    build time  hit count    saved time

      user     00:00.004          4     00:00.017
      post     00:00.002          1     00:00.002

Total time spent: 00:00.006
Total time saved: 00:00.019
Total time wasted: 00:00.000
```

Вывод статистики выключен по умолчанию. Вы можете включить его в настройках (`TestProf::AnyFixture.config.reporting_enabled = true`)
или указав переменную окружения `ANYFIXTURE_REPORT=1`.

## Использование автоматических SQL дампов

> @since v1.0, experimental

AnyFixture подразумевает генерацию данных для тестов единожды для каждого запуска. Однако даже это может занимать много времени (например, для системных тестов или тестов производительности, где, как правило, требуется много данных); значит, нам необходимо придумать, как ещё лучше оптимизировать популяцию тестовой базы данных!

Представляем вашему вниманию ещё один метод AnyFixture—`#register_dump`. При первом запуске он работает аналогично `#register`: выполняет блок кода и отслеживает SQL запросы. Помимо этого AnyFixture также генерирует текстовый SQL дамп из этих запросов, и этот дамп будет использован для последующих запусков тестов — блок кода не будет выполнятся.

Рассмотрим пример:

```ruby
RSpec.shared_context "account", account: true do
  # Как и в случае .register, необходимо использовать .register_dump вне транзакций
  before(:all) do
    # Блок кода выполняется максимум один раз за весь запуск тестов.
    # Код не выполняется вообще, если уже существует соответствующий SQL дамп.
    TestProf::AnyFixture.register_dump("account") do
      account = FactoryGirl.create(:account, name: "test")

      account = Fabricate(:account, name: "test")

      account = Account.create!(name: "test")

      account.update!(tag: "sql-dump")
    end
  end

  # Так как данные могут быть загружены из SQL дампа, минуя Ruby код,
  # мы должны использовать специфичный код для доступа к объектам
  let(:account) { Account.find_by!(name: "test") }
end
```

Когда мы запускаем тесты, происходит следующее:

```sh
# первый запуск
$ bundle exec rspec

# Вызов AnyFixture.register_dump:
# - Есть ли подходящий SQL дамп? Нет
# - Выполняем блок кода, записываем все модифицирующие запросы в текстовый SQL дамп
# Вызов AnyFixture.clean в конце запуска:
# - удаляем данные из всех затронутых таблиц

# повторной запуск
$ bundle exec rspec

# Вызов AnyFixture.register_dump:
# - Есть ли подходящий SQL дамп? Да
# - Восстанавливаем дамп
# Вызов AnyFixture.clean в конце запуска:
# - удаляем данные из всех затронутых таблиц
```

### Требования

В настоящий момент поддерживаются только PostgreSQL 12+ и SQLite3.

### Инвалидация дампов

Сгенерированный дамп может стать неактуальным по множеству причин: схема БД изменилась, код фикстуры обновился и т.д..
Для отслеживания актуальности дампа AnyFixture использует хэш-суммы соответствующих файлов. По умолчанию это `db/schema.rb`, `db/structure.sql` и файл, в котором происходит вызов `#register_dump`.

Вы можете изменить список файлов, за которыми нужно _следить_ с помощью параметра `default_dump_watch_paths`:

```ruby
TestProf::AnyFixture.configure do |config|
  # Можно использовать как точные пути, так и маски
  config.default_dump_watch_paths << Rails.root.join("spec/factories/**/*")
end
```

Также вы можете указать дополнительные файлы при вызове `#register_dump`:

```ruby
TestProf::AnyFixture.register_dump("account", watch: ["app/models/account.rb", "app/models/account/**/*,rb"]) do
  # ...
end
```

**Примечание**: Когда вы используете опцию `watch`, текущий файл не добавляется автоматически в список отслеживаемых. Вы можете добавить его вручную, используя `__FILE__`.

Наконец, если вы хотите умышленно пересоздать дамп, используйте переменную окружения `ANYFIXTURE_FORCE_DUMP`:

- `ANYFIXTURE_FORCE_DUMP=1` пересоздаст все дампы.
- `ANYFIXTURE_FORCE_DUMP=account` пересоздаст только дампы, принимаемые регулярным выражением `/account/`.

#### Дополнительные ключи кэша

Вы также можете указывать ключи для кэширования (в дополнение к хэш-суммам файлов):

```ruby
# cache_key может быть чем угодно, отвечающим на #to_s
TestProf::AnyFixture.register_dump("account", cache_key: ["str", 1, {key: :val}]) do
  # ...
end
```

### Хуки

#### `before` / `after`

Before хуки вызываются перед блоком кода (для первого запуска) или восстановлением дампа (для повторных запусков).
Например, вы можете их использовать для пересоздания схемы в БД:

```ruby
TestProf::AnyFixture.register_dump(
  "account",
  before: proc do
    begin
      Apartment::Tenant.create("test")
    rescue
      nil
    end
    Apartment::Tenant.create("test")
  end
) do
  # ...
end
```

Аналогичным образом, after хуки вызываются после выполнения блока кода или восстановления из дампа.

Вы также можете указать глобальные хуки (т.е., вызываемые для всех фикстур):

```ruby
TestProf::AnyFixture.configure do |config|
  config.before_dump do |dump:, import:|
    # dump — это объект, содержащий информацию о текущем дампе (например, dump.digest)
    # import — это флаг, который равен true тогда и только тогда, когда мы восстанавливаем данные из дампа
  end

  config.after_dump do |dump:, import:|
    # ...
  end
end
```

**Примечание**: after выполняются всегда, даже если блок кода или восстановление дампа упали с ошибкой. Статус создания/восстановления дампа может быть получен с помощью метода `dump.success?`.

#### `skip_if`

Данный хук позволяет полностью пропустить инициализацию фикстуры (как из кода, так и из дампа). Это полезно, если вы хотите сохранить состояние базы данных между запусками тестов (т.е., не выполняете очистку).

Пример:

```ruby
TestProf::AnyFixture.register_dump(
  "account",
  # затронутые таблицы не будут добавлены в лист зачистки для AnyFixture.clean (однако другие фикстуры могут их добавить)
  clean: false,
  skip_if: proc do |dump:|
    Apartment::Tenant.switch!("test")
    # проверяем, актуальное ли состояние данных в базе
    Account.find_by!(name: "test").meta["dump-version"] == dump.digest
  end,
  before: proc do
    begin
      Apartment::Tenant.create("test")
    rescue
      nil
    end
    Apartment::Tenant.create("test")
  end,
  after: proc do |dump:, import:|
    next if import || !dump.success?

    Account.find_by!(name: "test").then do |account|
      account.meta["dump-version"] = dump.digest
      account.save!
    end
  end
) do
  # ...
end
```

### Настройка

Доступны также следующие настройки:

```ruby
TestProf::AnyFixture.configure do |config|
  # Куда сохранять дампы (относительно TestProf.artifact_path)
  config.dumps_dir = "any_dumps"
  # Включать в дамп подходящие под регулярное выражение запросы (в дополнение к INSERT/UPDATE/DELETE)
  config.dump_matching_queries = /^$/
  # Использовать или нет консольные утилиты для восстановления дампов (psql или sqlite3)
  config.import_dump_via_cli = false
end
```

**Примечание**: При использовании консольных утилит для восстановления дампов невозможно отследить выполняемые запросы и затронутые таблицы; следовательно, `AnyFixture.clean` не будет работать.
