# Начало работы

## Требования

Поддерживаемые версии Ruby:

- Ruby (MRI) >= 2.5.0 (для Ruby 2.2 использутей TestProf < 0.7.0, Ruby 2.3 — TestProf ~> 0.7.0, Ruby 2.4 — TestProf <0.12.0);

- JRuby >= 9.1.0.0 (некоторые инструменты требуют версии 9.2.7+).

При использовании RSpec требуется версия >=3.5.0 (для более старых версий используйте TestProf < 0.8.0).

## Установка

Добавьте гем `test-prof`:

```ruby
group :test do
  gem "test-prof"
end
```

Можете приступать к использованию!

## Настройки

TestProf имеет ряд глобальных настроек, которые используются всеми инструментами:

```ruby
TestProf.configure do |config|
  # путь для сохранения артефактов, например, отчётов ('tmp/test_prof' по умолчанию)
  config.output_dir = "tmp/test_prof"

  # использования уникальных имён для артефактов путём добавления временных меток в название
  config.timestamps = true

  # подсвечивать вывод
  config.color = true
end
```

### Идентификаторы для отчётов/артефактов

Вы также можете динамически задавать суффикс для гененируемых отчётов/артефактов с помощью переменной окружения `TEST_PROF_REPORT`. Это позволяет идентифицировать отчёты для разных запусков одного и того же профилировщика с целью последующего сравнения.

**Пример.** Сравним время загрузки тестов с и без использования `bootsnap` с помощью [`stackprof`](./profilers/stack_prof.md):

```sh
# Первый отчёт будет иметь суффикс `-with-bootsnap`
$ TEST_STACK_PROF=boot TEST_PROF_REPORT=with-bootsnap bundle exec rake
$ #=> StackProf report generated: tmp/test_prof/stack-prof-report-wall-raw-boot-with-bootsnap.dump

# А для второго зададим суффикс `no-bootsnap`
$ TEST_STACK_PROF=boot TEST_PROF_REPORT=no-bootsnap bundle exec rake
$ #=> StackProf report generated: tmp/test_prof/stack-prof-report-wall-raw-boot-no-bootsnap.dump
```

Теперь у вас есть два отчётов с «говорящими» названиями!
