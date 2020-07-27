# Профилирование тестов со RubyProf

TestProf интегрируется с [ruby-prof](https://github.com/ruby-prof/ruby-prof) для удобного профилирования тестов.

## Инструкция

Установите гем `ruby-prof` (версии >=0.17):

```ruby
# Gemfile
group :development, :test do
  gem "ruby-prof", ">= 0.17.0", require: false
end
```

TestProf позволяет запускать RubyProf в двух режимах: для всего запуска тестов (`global`) или для индивидуальных тестов (`per-example`).

Для профилирования выполнения всех тестов (от загрузки до завершения) укажите переменную окружения `TEST_RUBY_PROF`:

```sh
TEST_RUBY_PROF=1 bundle exec rake test

# or for RSpec
TEST_RUBY_PROF=1 rspec ...
```

Либо программно в коде:

```ruby
TestProf::RubyProf.run
```

TestProf также предоставляет специальный контекст (shared context) для RSpec для профилирования конкретных примеров.
Подключить его можно с помощью тега `rprof`:

```ruby
it "is doing heavy stuff", :rprof do
  # ...
end
```

**Примечание:** индивидуальное профилирование не работает, если уже активировано глобальное.

## Настройки

Наиболее часто используемая настройка — `printer` – позволяет выбирать способ вывода (_printer_) для RubyProf (см. [Printers](https://github.com/ruby-prof/ruby-prof#printers)).

Вы можете указать способ вывода с помощью переменной окружения `TEST_RUBY_PROF`:

```sh
TEST_RUBY_PROF=call_stack bundle exec rake test
```

Либо в коде:

```ruby
TestProf::RubyProf.configure do |config|
  config.printer = :call_stack
end
```

По умолчанию используется `FlatPrinter`.

**Примечания:** для указания способа вывода при индивидуальном профилировании используйте переменную окружения `TEST_RUBY_PROF_PRINTER` (использование `TEST_RUBY_PROF` невозможно, так как оно активирует глобальное профилирование).

Также вы можете указать режим RubyProf (`wall`, `cpu` и т.д.) с помощью переменной окружения `TEST_RUBY_PROF_MODE`.

Все настройки доступны в [ruby_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/ruby_prof.rb).

### Фильтрация методов

Иногда бывает полезно исключить некоторые методы из финального отчёта («почистить шум»).

TestProf автоматически включает базовую фильтрацию из RubyProf ([`exclude_common_methods!`](https://github.com/ruby-prof/ruby-prof/blob/e087b7d7ca11eecf1717d95a5c5fea1e36ea3136/lib/ruby-prof/profile/exclude_common_methods.rb), можно отключить с помощью опции `config.exclude_common_methods = false`).

Кроме того, мы так же добавляем в исключения некоторые другие популярные методы стандартных библиотек и популярных фреймворков (например, RSpec).
Для отключения данной фильтрации укажите в настройках `config.test_prof_exclusions_enabled = false`.

Наконец, вы можете добавить собственные исключения используя опцию `config.custom_exclusions`, например:

```ruby
TestProf::RubyProf.configure do |config|
  config.custom_exclusions = {User => %i[save save!]}
end
```
