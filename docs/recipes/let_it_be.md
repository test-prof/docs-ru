# Let It Be

Предлагаем перейти на новый уровень и добавить ещё щепотку магии для эффективной генерации тестовых данных!

Рассмотрим пример создания данных для теста:

```ruby
describe BeatleWeightedSearchQuery do
  let!(:paul) { create(:beatle, name: "Paul") }
  let!(:ringo) { create(:beatle, name: "Ringo") }
  let!(:george) { create(:beatle, name: "George") }
  let!(:john) { create(:beatle, name: "John") }

  specify { expect(subject.call("john")).to contain_exactly(john) }

  # далее идут другие тесты
end
```

Что бросается в глаза? Мы создаём _великолепную четвёрку_ заново для каждого теста. И хотя Битлз много не бывает, на скорости выполнения наших тестов это сказывается негативно.

Мы могли бы воспользоваться [`before_all`](./before_all.md) для решения проблема _повторяемых данных_:

```ruby
describe BeatleWeightedSearchQuery do
  before_all do
    @paul = create(:beatle, name: "Paul")
    # ...
  end

  specify { expect(subject.call("joh")).to contain_exactly(@john) }

  # ...
end
```

Данный подход работает, но имеет ряд недостатков с точки зрения ускорения тестов _малой кровью_: нам нужно заменить `let/let!` выражения, а также методы на переменные инстанса класса.

С помощью `let_it_be` же мы можем сделать так:

```ruby
describe BeatleWeightedSearchQuery do
  let_it_be(:paul) { create(:beatle, name: "Paul") }
  let_it_be(:ringo) { create(:beatle, name: "Ringo") }
  let_it_be(:george) { create(:beatle, name: "George") }
  let_it_be(:john) { create(:beatle, name: "John") }

  specify { expect(subject.call("john")).to contain_exactly(john) }

  # далее идут другие тесты
end
```

И всё! Меняем `let!` на `let_it_be`. «Под капотом» используется тот же `before_all`, но _внешний_ API ближе к стандартному API RSpec.

**Примечание**: Как говорил дядя Питера Паркера: «Чем больше сила, тем больше ответственность».
Обязательно проверьте [раздел «Предостережения»](#предостережения) для более подробной информации.

## Инструкция

В вашем `rails_helper.rb` или `spec_helper.rb` добавьте строчку:

```ruby
require "test_prof/recipes/rspec/let_it_be"
```

Теперь вы можете использовать `let_it_be` в тестах:

```ruby
describe MySuperDryService do
  let_it_be(:user) { create(:user) }

  # ...
end
```

**Примечание**: `let_it_be` не оборачивает индивидуальные тесты в собственные транзакции базы данных.
Используйте нативный функционал Rails, `use_transactional_tests`,(`use_transactional_fixtures` в Rails < 5.1),
либо `use_transactional_fixtures` из `rspec-rails`, либо DatabaseCleaner, либо свой код, который будет создавать транзакцию перед тестом и откатывать ее после.

## Предостережения

### База данных откатывается в первоначальное состояние, но объекты - нет

Если вы измените объекты, созданные в блоке  `let_it_be`, в вашем тесте, то вам, возможно, придется повторно инициировать их.
Для это вы можете использовать [модификаторы](#модификаторы).

См. также [предостережения для `before_all`](./before_all.md#предостережения).

## Модификаторы

Модификаторы позволяют совершать дополнительные действия над объектами, генерируемыми через `let_it_be`, перед **каждым тестом**.
Это помогает решать проблему утечки состояния объектов между тестами (база данных _отказывается_ после каждого примера, но Ruby объекты — нет).

Доступны следующие модификаторы:

```ruby
# используйте reload: true для перезагрузки объекта для каждого теста (предполагается, что вы используете ActiveRecord)
let_it_be(:user, reload: true) { create(:user) }

# эквивалентно
before_all { @user = create(:user) }
let(:user) { @user.reload }

# модификатор refind: true инициирует полностью новый объект (в некоторых случаях reload недостаточно)
let_it_be(:user, refind: true) { create(:user) }

# эквивалентно
before_all { @user = create(:user) }
let(:user) { User.find(@user.id) }
```

**Примечение**: убедитесь, что `let_it_be` загружается после `active_record` (например, в `rails_helper.rb` **после** загрузки Rails приложения); в противном случае модификаторы `refind` и `reload` не будут доступны.

Модификаторы могут также использоваться с массивами объектов, например, созданными с помощью `create_list`:

```ruby
let_it_be(:posts, reload: true) { create_list(:post, 3) }

# эквивалентно
before_all { @posts = create_list(:post, 3) }
let(:posts) { @posts.map(&:reload) }
```

### Пользовательские модификаторы

Вы можете добавить свои модификаторы:

```ruby
# rails_helper.rb
TestProf::LetItBe.configure do |config|
  # Так выглядит определение модификатора `reload`:
  # первый аргумент — это объект, второй — значение модификатора
  config.register_modifier :reload do |record, val|
    # игнорируем, если указано `reload: false`
    next record unless val
    # игнорируем не-ActiveRecord объекты
    next record unless record.is_a?(::ActiveRecord::Base)
    record.reload
  end
end
```

### Модификаторы по умолчанию

Вы можете указать, какие модификаторы должны быть использованы для `let_it_be` по умолчанию:

- глобально:

```ruby
TestProf::LetItBe.configure do |config|
  config.default_modifiers[:refind] = true
end
```

- для отдельных групп тестов, используя теги:

```ruby
context "with let_it_be reload", let_it_be_modifiers: {reload: true} do
  # ...
end
```

**Примечени**: Теги для вложенных контекстов перезаписываются:

```ruby
TestProf::LetItBe.configure do |config|
  config.default_modifiers[:freeze] = false
end

context "with reload", let_it_be_modifiers: {reload: true} do
  # здесь будет использовано freeze: false, reload: true

  context "with freeze", let_it_be_modifiers: {freeze: true} do
    # здесь будет использовано только freeze: true (reload: true будет перезаписан)
  end
end
```

## Псевдонимы (_алиасы_)

Не всем нравится Битлз 😞 Нет проблем — вы можете настроить псевдоним для метода `let_it_be` и использовать любое другое имя:

```ruby
# rails_helper.rb
TestProf::LetItBe.configure do |config|
  config.alias_to :да_будет_так
end
```

Вы также можете указывать модификаторы по умолчанию для псевдонимов:

```ruby
# rails_helper.rb
TestProf::LetItBe.configure do |config|
  config.alias_to :let_it_be_with_refind, refind: true
end

describe "smth" do
  let_it_be_with_refind(:foo) { Foo.create }

  # refind может быть переопределён
  let_it_be_with_refind(:bar, refind: false) { Bar.create }
end
```

## Обнаружение утечек состояния

Из [документации `rspec-rails`](https://relishapp.com/rspec/rspec-rails/v/3-9/docs/transactions) про `before(:context)`:

> Even though database updates in each example will be rolled back, the object won't know about those rollbacks so the object and its backing data can easily get out of sync.

Так как `let_it_be` выполняет код внутри `before(:context)`, мы имеем дело с описанной выше проблемой: Руби-объекты, переиспользуемые в тестах, могут изменяться, тем самым приводя к _утечкам_. Например, тестируемый код может изменять модели, обновлять значение `updated_at`; или в самих тестах вы можете изменять объекты в `before` хуках.

Подобные утечки приводят к возникновению зависимостей между тестами и вносят нестабильность в успешность их выполнения.
С другой стороны, обнаружить подобные проблемы бывает не так-то просто.

Мы предлагаем специальный модификатор, `freeze`, который делает объекты, генерируемые с помощью `let_it_be`, неизменяемыми:

```ruby
let_it_be(:user, freeze: true) { create(:user) }

# эквивалентно
before_all { @user = create(:user).freeze }
let(:user) { @user }
```

Если после добавления модификатора `freeze` ваши тесты падают с ошибкой `FrozenError`, то:

- добавьте `reload: true`/`refind: true`; в большинстве случаев это устраняет проблему. Перезагрузка модели (`reload`), как правило, значительное быстрее, чем пересоздание (`refind`), поэтому для начала рекоммендуем пробовать использовать `reload: true`;
- перепишите проблемный тест.

Для проектов, которые только начинают использовать `let_it_be`, мы рекомендуем активировать модификатор `freeze` по умолчанию (см. выше).
Существующие проекты могут использовать теги для постепенного исправления тестов:

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  # ...
  config.define_derived_metadata(let_it_be_frost: true) do |metadata|
    metadata[:let_it_be_modifiers] ||= {freeze: true}
  end
end
```
