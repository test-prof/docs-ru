# Factory Doctor

Одна из распространенных причин, замедляющих наши тесты, — это ненужные манипуляции с базой данных. Рассмотрим _плохой_ пример:

```ruby
# with FactoryBot/FactoryGirl
it "validates name presence" do
  user = create(:user)
  user.name = ""
  expect(user).not_to be_valid
end

# with Fabrication
it "validates name presence" do
  user = Fabricate(:user)
  user.name = ""
  expect(user).not_to be_valid
end
```

Здесь мы создаем новую запись, запуская callback-функции и валидации, и сохраняем её в базу данных. Нам всё это не нужно!  Вот _хороший_ пример:

```ruby
# with FactoryBot/FactoryGirl
it "validates name presence" do
  user = build_stubbed(:user)
  user.name = ""
  expect(user).not_to be_valid
end

# with Fabrication
it "validates name presence" do
  user = Fabricate.build(:user)
  user.name = ""
  expect(user).not_to be_valid
end
```

Подробнее о [`build_stubbed`](https://robots.thoughtbot.com/use-factory-girls-build-stubbed-for-a-faster-test).

FactoryDoctor помогает вам найти такие _плохие_ тесты, т. е. тесты, которые выполняют ненужные запросы к базе данных.

Пример вывода:

```sh
[TEST PROF INFO] FactoryDoctor report

Total (potentially) bad examples: 2
Total wasted time: 00:13.165

User (./spec/models/user_spec.rb:3) (3 records created, 00:00.628)
  validates name (./spec/user_spec.rb:8) – 1 record created, 00:00.114
  validates email (./spec/user_spec.rb:8) – 2 records created, 00:00.514
```

**Примечание**: вы обратили на слово «potentially» (потенциально)? К сожалению, FactoryDoctor не волшебник (он все еще учится),
и иногда он выдает ложноотрицательные или ложноположительные результаты.

Пожалуйста, создайте [issue](https://github.com/test-prof/test-prof/issues), если вы нашли пример, в котором FactoryDoctor ошибся.

Вы можете исключить отдельные тесты из анализа, добавив  тег `:fd_ignore`:

```ruby
# данный тест не будет отмечен FactoryDoctor
it "is ignored", :fd_ignore do
  user = create(:user)
  user.name = ""
  expect(user).not_to be_valid
end
```

## Инструкции

FactoryDoctor поддерживает:

- FactoryGirl/FactoryBot
- Fabrication (**@since v0.9.0**).

### RSpec

Для активации FactoryDoctor используйте переменную окружения `FDOC`:

```sh
FDOC=1 rspec ...
```

### Использование с RSpecStamp

FactoryDoctor может быть использован вместе с [RSpec Stamp](../recipes/rspec_stamp.md) для автоматического тегирования _плохих_ тестов. Например:

```sh
FDOC=1 FDOC_STAMP="fdoc:consider" rspec ...
```

После запуска команды выше все _потенциально_ плохие тесты будут помечены тегом `fdoc: :consider`.

### Minitest

Для активации FactoryDoctor используйте переменную окружения `FDOC`:

```sh
FDOC=1 ruby ...
```

Или используйте опцию, как показано ниже:

```sh
ruby ... --factory-doctor
```

Опция игнорирования определённых тестов также доступна и для Minitest.
Просто используйте `fd_ignore` внутри вашего теста:

```ruby
# данный тест не будет отмечен FactoryDoctor
it "is ignored" do
  fd_ignore

  @user.name = ""
  refute @user.valid?
end
```

### Использование с Minitest::Reporters

Если в вашем проекте вы используете `Minitest::Reporters`,
то вам необходимо явно указать это в загрузочном файле (обычно, `test_helper.rb`):

```sh
require 'minitest/reporters'
Minitest::Reporters.use! [YOUR_FAVORITE_REPORTERS]
```

**Примечание**: Когда гем `minitest-reporters`у вас установлен глобально, но не объявлен в вашем `Gemfile`,
убедитесь, что всегда выполняете команду запуска тестов с `bundle exec` (но мы уверены, что вы и так это делаете).
В противном случае вы получите ошибку, вызванную системой плагинов Minitest, которая ищет все файлы в `$LOAD_PATH` для любого `minitest/*_plugin.rb`,
таким образом инициализация плагина `minitest-reporters`, который доступен в этом случае, происходит некорректно.

## Настройки

> @since v0.9.0

Доступны следующие параметры конфигурации (показаны значения по умолчанию):

```ruby
TestProf::FactoryDoctor.configure do |config|
  # Какое событие инструментации отслеживать для трекинга запросов к БД
  config.event = "sql.active_record"
  # Игнорировать запросы к БД, которые заняли меньше указанного времени (в секундах)
  config.threshold = 0.01
end
```

Вы можете использовать соответствующие переменные окружения для указания данных параметров: `FDOC_EVENT` и `FDOC_THRESHOLD` соответственно.
