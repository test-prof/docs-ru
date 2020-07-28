# Factory Doctor

Одна из распространенных причин, замедляющих наши тесты - это не нужные манипуляции с базой данных. Рассмотрим _плохой_ пример:

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

Здесь мы создаем новую запись, запуская коллбеки и валидация и сохраняем это в базу данных. Нам все это не нужно!  Вот _хороший_ пример::

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

FactoryDoctor это инструмент, который помогает вам найти такие _плохие_ тесты, т. е. тесты, которые выполняют ненужные запросы к базе данных.

Пример вывода:

```sh
[TEST PROF INFO] FactoryDoctor report

Total (potentially) bad examples: 2
Total wasted time: 00:13.165

User (./spec/models/user_spec.rb:3) (3 records created, 00:00.628)
  validates name (./spec/user_spec.rb:8) – 1 record created, 00:00.114
  validates email (./spec/user_spec.rb:8) – 2 records created, 00:00.514
```

**Примечание**: вы обратили на слово “potentially”(потенциально)? К сожалению, FactoryDoctor не волшебник (он все еще учится)
и иногда он выдает ложные отрицательные и ложные положительные результаты тоже.

Пожалуйста, создайте [issue](https://github.com/test-prof/test-prof/issues) если вы нашли пример, в котором FactoryDoctor ошибся.

Вы также можете сказать FactoryDoctor, чтобы он игнорировал определенные `examples/groups`. Просто добавьте `:fd_ignore` тег для `it`:

```ruby
# won't be reported as offense
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

FactoryDoctor может быть использован вместе с [RSpec Stamp](../recipes/rspec_stamp.md) для автоматического выделению _плохих_ тестов с кастомным тегом. Например:

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

Опция игнорирования определённых тестов, также доступна и для Minitest.
Просто используйте `fd_ignore` внутри вашего теста.:

```ruby
# won't be reported as offense
it "is ignored" do
  fd_ignore

  @user.name = ""
  refute @user.valid?
end
```

### Использование с Minitest::Reporters

Если в вашем проекте вы используете `Minitest::Reporters`,
то вы должны явно объявить в вашем хeлпере это:

```sh
require 'minitest/reporters'
Minitest::Reporters.use! [YOUR_FAVORITE_REPORTERS]
```

**Примечание**: Когда у вас установлен гем `minitest-reporters` но не объявлен в вашем `Gemfile`,
убедитесь, что всегда выполняете команду запуска тестов с `bundle exec` (но мы уверенны, что вы и так это всегда делаете).
Иначе, вы получите ошибку, вызванную системой плагинов Minitest, которая ищет все файлы в `$LOAD_PATH` для любого `minitest/*_plugin.rb`,
таким образом инициализация плагина `minitest-reporters`, который доступен в этом случает, происходи некорректно

## Конфигурация

> @since v0.9.0

Доступны следующие параметры конфигурации (показаны значения по умолчанию):

```ruby
TestProf::FactoryDoctor.configure do |config|
  # Which event to track within test example to consider them "DB-dirty"
  config.event = "sql.active_record"
  # Consider result "good" if the time in DB is less then the threshold
  config.threshold = 0.01
end
```

Вы можете использовать соответствующие переменные окружения, а также: `FDOC_EVENT` и `FDOC_THRESHOLD`.