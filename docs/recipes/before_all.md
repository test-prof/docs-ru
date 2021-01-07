# Before All

В Rails есть встроенная возможность запускать каждый тест внутри транзакции БД,
которая автоматически откатывается назад после выполнения данного теста (`transactional_tests`).

Таким образом, ни один тест не _загрязняет_ базу данных, у нас не возникает _глобального состояния_ (которое часто приводит к нестабильным тестам).

Но что делать, если есть много тестов с общим контекстом?

Конечно, мы можем сделать что-то вроде этого:

```ruby
describe BeatleWeightedSearchQuery do
  before(:each) do
    @paul = create(:beatle, name: "Paul")
    @ringo = create(:beatle, name: "Ringo")
    @george = create(:beatle, name: "George")
    @john = create(:beatle, name: "John")
  end

  # и около 15 тестов здесь
end
```

Или вы можете попробовать `before(:all)`:

```ruby
describe BeatleWeightedSearchQuery do
  before(:all) do
    @paul = create(:beatle, name: "Paul")
    # ...
  end

  # ...
end
```

Но тогда вам придется иметь дело с очисткой базы данных, т.к. `before(:all)` **вызывается вне транзакции**.
_Подчищать_ базу вручную, как правило, непросто или медленно (например, с помощью DatabaseCleaner).

Есть вариант получше: мы можем **обернуть всю группу примеров в транзакцию**.
И вот как работает `before_all`:

```ruby
describe BeatleWeightedSearchQuery do
  before_all do
    @paul = create(:beatle, name: "Paul")
    # ...
  end

  # ...
end
```

Вот и все!

**Примечание**: требуется RSpec >= 3.3.0.

**Примечание**: Как говорил дядя Питера Паркера: «Чем больше сила, тем больше ответственность».
Обязательно проверьте [раздел «Предостережения»](#предостережения) этого документа для получения более подробной информации.

## Инструкция

### RSpec

В вашем `rails_helper.rb` (или `spec_helper.rb`) **после загрузки ActiveRecord**:

```ruby
require "test_prof/recipes/rspec/before_all"
```

**Примечание**: `before_all` (и [`let_it_be`](./let_it_be.md), который от него зависит),
не оборачивает индивидуальные тесты в собственные транзакции базы данных.
Используйте нативный функционал Rails, `use_transactional_tests`,(`use_transactional_fixtures` в Rails < 5.1),
либо `use_transactional_fixtures` из `rspec-rails`, либо DatabaseCleaner, либо свой код, который будет создавать транзакцию перед тестом и откатывать ее после.

### Minitest

Можно также использовать `before_all` с Minitest:

```ruby
require "test_prof/recipes/minitest/before_all"

class MyBeatlesTest < Minitest::Test
  include TestProf::BeforeAll::Minitest

  before_all do
    @paul = create(:beatle, name: "Paul")
    @ringo = create(:beatle, name: "Ringo")
    @george = create(:beatle, name: "George")
    @john = create(:beatle, name: "John")
  end

  # можно определить тесты, которые используют объекты определённые в `before_all`
end
```

В дополнение к `before_all` TestProf также предоставляет коллбек `after_all`, который вызывается непосредственное перед закрытиием транзакции, открытой в `before_all`, т.е., после выполнения последнего теста из класса (с учётом фильтров).

## Адаптеры баз данных

Вы можете использовать `before_all` не только с ActiveRecord (который поддерживается из коробки), но и с другими инструментами для работы с базой данных.

Все, что вам нужно, - это создать пользовательский адаптер и настроить `before_all` для его использования:

```ruby
class MyDBAdapter
  # before_all адаптеры должны реализовывать два метода:
  # - begin_transaction
  # - rollback_transaction
  def begin_transaction
    # ...
  end

  def rollback_transaction
    # ...
  end
end

# А затем установите адаптер для `BeforeAll` модуля
TestProf::BeforeAll.adapter = MyDBAdapter.new
```

## Callback-функции

> @since v0.9.0

Вы можете зарегистрировать callback-функции для событий открытия и отката транзакции `before_all:

```ruby
TestProf::BeforeAll.configure do |config|
  config.before(:begin) do
    # что-то выполняется до открытия транзакции
  end
  # after(:begin) также доступен

  config.after(:rollback) do
    # что-то выполняется после закрытия транзакции
  end
  # before(:rollback) также доступен
end
```

См. пример из [Discourse](https://github.com/discourse/discourse/blob/4a1755b78092d198680c2fe8f402f236f476e132/spec/rails_helper.rb#L81-L141).

## Предостережения

### База данных откатывается в первоначальное состояние, но объекты - нет

Если вы измените объекты, созданные в блоке `before_all`, в вашем тесте, то вам, возможно, придется повторно инициировать их:

```ruby
before_all do
  @user = create(:user)
end

let(:user) { @user }

it "when user is admin" do
  # мы изменили наш объект на месте!
  user.update!(role: 1)
  expect(user).to be_admin
end

it "when user is regular" do
  # теперь состояние @user зависит от порядка тестов!
  expect(user).not_to be_admin
end
```

Самый простой способ решить эту проблему - перезагрузить запись для каждого теста (это все равно намного быстрее, чем создание новой):

```ruby
before_all do
  @user = create(:user)
end

# Обратите внимание, что @user.reload может быть не достаточным,
# потому что он не сбрасывает ассоциации
let(:user) { User.find(@user.id) }

# или в Minitest
def setup
  @user = User.find(@user.id)
end
```

### База данных не откатывается между тестами

База данных не откатывается между тестами, а только между группами тестов.
Мы не хотим изобретать велосипед, а хотим поощрять использование других инструментов, которые
предоставляют это из коробки.

Если вы используете RSpec Rails, включите `RSpec.configuration.use_transactional_fixtures` в вашем `spec/rails_helper.rb`:

```ruby
RSpec.configure do |config|
  # RSpec позаботится о том, чтобы использовать `use_transactional_tests` или 
  # `use_transactional_fixtures` в зависимости от вашей версии Rails
  config.use_transactional_fixtures = true
end
```

Убедитесь, что установлен `use_transactional_tests` (`use_transactional_fixtures` в Rails < 5.1) на `true`, если вы используете Minitest.

Если вы используете DatabaseCleaner, убедитесь, что он откатывает базу данных между тестами.

## Использование с гемом Isolator

[Isolator](https://github.com/palkan/isolator) — это инструмент для обнаружения потенциальных нарушений атомарности в транзакциях БД (например, выполнение HTTP-вызовов или постановка в очередь фоновых задач).

TestProf распознает Isolator из коробки и делает так, чтобы он игнорировал `before_all` транзакции.

Вам просто нужно убедиться, что вы подключили `isolator` перед загрузкой `before_all` или подключите следующий патч явно:

```ruby
# После загрузки before_all и/или let_it_be
require "test_prof/before_all/isolator"
```

## Использование с Rails fixtures (_экспериментальная функция_)

> @since v1.0.0

Если вы хотите использовать фикстуры в `before_all` коллбеках, вам необходимо явно их инициировать, используя опцию `setup_fixture:`:

```ruby
before_all(setup_fixtures: true) do
  @user = users(:john)
  @post = create(:post, user: user)
end
```

Данная опция поддерживается и в Minitest, и в RSpec.

Вы также можете установить значение по умолчанию для `setup_fixtures`:

```ruby
TestProf::BeforeAll.configure do |config|
  config.setup_fixtures = true
end
```
