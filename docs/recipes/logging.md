# Логирование в Rails

Иногда анализ логов — это лучший способ определения проблемы. По умолчанию логи в Rails в тестовом режиме
пишутся в файл (`test.log`) — это вряд ли нам может как-то помочь, так?

Мы рекомендуем отключать логи в тестах по умолчанию (это, кстати, [увеличит их скорость на 5-6%](https://jtway.co/speed-up-your-rails-test-suite-by-6-in-1-line-13fedb869ec4) и использовать специальные инструменты для активация вывода логов в консоль по запросу.

## Инструкция

Добавьте следующую строчку в ваш `rails_helper.rb` (или `spec_helper.rb`):

```ruby
require "test_prof/recipes/logging"
```

### Вывод логов для всего запуска тестов

Для включения логирования глобально, укажите переменную окружения `LOG`:

```sh
# Вывод всех логов
LOG=all rspec ...

# или
LOG=all rake test

# Показывать только логи Active Record
LOG=ar rspec ...
```

### Вывод логов для конкретного теста

**Примечание:** Только для RSpec.

Для вывод логов при выполнении конкретного теста укажите тег `:log`:

```ruby
it "does smthng weird", :log do
  # ...
end

# или для группы тестов
describe "GET #index", :log do
  # ...
end
```

Для вывода только логов Active Record используйте тег `log: :ar`:

```ruby
describe "GET #index", log: :ar do
  # ...
end
```

### Вспомогательные методы для логирования

Для большего контроля вы можете использовать методы `with_logging` (все логи) и
`with_ar_logging` (логи Active Record):

```ruby
it "does somthing" do
  do_smth
  # Будут выведены логи при выполнения кода внутри блока
  with_logging do
    # ...
  end
end
```

**Примечание:** для использования данных методов в Minitest необходимо явно добавить модуль `TestProf::Rails::LoggingHelpers`:

```ruby
class MyLoggingTest < Minitest::Test
  include TestProf::Rails::LoggingHelpers
end
```
