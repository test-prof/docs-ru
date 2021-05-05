# EventProf

EventProf интегрируется с библиотеками для инструментации (такими как [`ActiveSupport::Notifications`](https://edgeguides.rubyonrails.org/active_support_instrumentation.html)), позволяя собирать более детальную информацию о различных _событиях_ в вашем коде (например, запросах к БД, создании фоновых задач, и т.д.).

В результате вы получаете отчёт о том, в каких тестах и группах то или иное событие происходит чаще и занимает больше времени. Можно сказать, что это продвинутый `rspec --profile`, который умеет замерять не только общее время выполнения, но и время, потраченное на определённые действия.

Наиболее полезен данный профилировщик для поиска тестов, в которых происходит чрезмерное взаимодействие с базой данных (событие `sql.active_record` в Rails) или, например, создание объектов с помощью фабрик в тестах (см. событие `factory.create` ниже).

Пример отчёта:

```sh
[TEST PROF INFO] EventProf results for sql.active_record

Total time: 00:00.256 of 00:00.512 (50.00%)
Total events: 1031

Top 5 slowest suites (by time):

AnswersController (./spec/controllers/answers_controller_spec.rb:3) – 00:00.119 (549 / 20) of 00:00.200 (59.50%)
QuestionsController (./spec/controllers/questions_controller_spec.rb:3) – 00:00.105 (360 / 18) of 00:00.125 (84.00%)
CommentsController (./spec/controllers/comments_controller_spec.rb:3) – 00:00.032 (122 / 4) of 00:00.064 (50.00%)

Top 5 slowest tests (by time):

destroys question (./spec/controllers/questions_controller_spec.rb:38) – 00:00.022 (29) of 00:00.064 (34.38%)
change comments count (./spec/controllers/comments_controller_spec.rb:7) – 00:00.011 (34) of 00:00.022 (50.00%)
change Votes count (./spec/shared_examples/controllers/voted_examples.rb:23) – 00:00.008 (25) of 00:00.022 (36.36%)
change Votes count (./spec/shared_examples/controllers/voted_examples.rb:23) – 00:00.008 (32) of 00:00.035 (22.86%)
fails (./spec/shared_examples/controllers/invalid_examples.rb:3) – 00:00.007 (34) of 00:00.014 (50.00%)
```

Смотрите официальную [документацию Rails](http://guides.rubyonrails.org/active_support_instrumentation.html) для списка всех доступных событий.

**Примечание**. Если вы используете [rom-rb](http://rom-rb.org), для профилирования времени, потраченного на запросы к БД, вам потребуется событие `'sql.rom'`.

## Инструкции

EventProf поддерживает `ActiveSupport::Notifications` «из коробки».

Используйте переменную окружения `EVENT_PROF` с указанием названия события для активации профилировщика:

```sh
# Для RSpec
EVENT_PROF='sql.active_record' rspec ...
# Для Minitest аналогично
EVENT_PROF='sql.active_record' rake test
```

Вы можете следить за несколькими событиями одновременно, указав их через запятую:

```sh
EVENT_PROF='sql.active_record,perform.active_job' rspec ...
```

### Minitest

Если вы используете Minitest, вы также можете активировать EventProf, используя опцию CLI:

```sh
ruby test/my_super_test.rb --event-prof=sql.active_record
```

## Настройки

По умолчанию EventProf агрегирует статистику по группам тестов (верхнеуровневые `describe` для RSpec, файлы для Minitest.
Опционально вы можете включить профилирование отдельных тестов:

```ruby
TestProf::EventProf.configure do |config|
  config.per_example = true
end
```

Данную опцию можно также активировать с помощью переменной окружения `EVENT_PROF_EXAMPLES=1`.

Другая полезная настройка — `rank_by`. Она отвечает за то, по какому показателю формировать _рейтинг_: затраченному времени (_time_) или количеству событий (_count_):

```sh
EVENT_PROF_RANK=count EVENT_PROF='instantiation.active_record' be rspec
```

Все настройки доступны в [event_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/event_prof.rb).

## Интеграция с RSpecStamp

EventProf интегрируется с [RSpec Stamp](../recipes/rspec_stamp.md) для автоматической пометки _медленных_ тестов тегами. Например:

```sh
EVENT_PROF="sql.active_record" EVENT_PROF_STAMP="slow:sql" rspec ...
```

После запуска тестов с данными параметрами самые медленные группы тестов (или отдельные примеры, если выбрана данная опция) будут помечены тегом `slow: :sql`.

## Использование с другими библиотеками для инструментации

Для того, чтобы использовать EventProf с отличными (от поддерживаемых) библиотеками для авторизации, необходимо выполнить следующие шаги:

- Добавить класс-обёртку для целевой библиотеки, реализующей метод `#subscribe(event, &block)`, который вызывает переданный блок при возникновении события и передаёт потраченное время в качестве единственного аргумента:

```ruby
module MyEventsWrapper
  def self.subscribe(event)
    raise ArgumentError, "Block is required!" unless block_given?

    ::MyEvents.subscribe(event) do |start, finish, *|
      yield (finish - start)
    end
  end
end
```

- Указать данный класс в настройках:

```ruby
TestProf::EventProf.configure do |config|
  config.instrumenter = MyEventsWrapper
end
```

## Дополнительные события

EventProf также предоставляет собственную инструментацию (на базе ActiveSupport::Notifications) для некоторых событий, анализ которых особенно полезен при поиске проблемных мест в тестах.

### `"factory.create"`

Несмотря на то, что гем FactoryBot предоставляет собственную инструментацию (событие `'factory_bot.run_factory'`), использовать её для целей учёта времени, потраченного на фабрики, просто так не получится – событие вызывается при каждом создании объекта с помощью фабрик, включая вложенные вызовы (например, когда у вас есть ассоциации). Это может привести к двойным вычислениям, а значит некорректным результатам.

Для решения этой проблеме EventProf предоставляет собственное событие, которое вызывается только для _внешних_ вызовов `FactoryBot#create`, — `'factory.create'`:

```sh
EVENT_PROF=factory.create bundle exec rspec
```

Кроме того, данное событие универсально и работает также с библиотекой [Fabrication][].

### `"sidekiq.jobs"`

Данное событие возникает при выполнении задач Sidekiq **в режиме inline** (`Sidekiq::Testing.inline!`):

```sh
EVENT_PROF=sidekiq.jobs bundle exec rspec
```

**Примечание**: для данного события автоматически включается сортировка по количеству событий (`rank_by=count`).

### `"sidekiq.inline"`

Данное событие возникает при выполнении задач Sidekiq **в режиме inline** (`Sidekiq::Testing.inline!`). Данное событие отличается от `"sidekiq.jobs"` тем, что не вызывается только вложенных задач. Таким образом, предполагается использовать его для вычисления времени, потраченного _внутри_ Sidekiq в тестах:

```sh
EVENT_PROF=sidekiq.inline bundle exec rspec
```

## Инструментация для произвольных методов

Вы можете добавлять инструментацию произвольных методов любых классов в вашем приложении, т.е. замерять, сколько времени вы проводите _внутри_ этих методов или как часто там оказываетесь в процессе выполнения тестов (например, после того, как вы обнаружили «тяжёлые» методы с помощью [RubyProf](./ruby_prof.md) или [StackProf](./stack_prof.md)).

Допустим, у вас есть следующий класс:

```ruby
class Work
  def do_smth(*args)
    # do something
  end
end
```

Для того, чтобы добавить событие, возникающее при вызове метода `#do_smth` данного класса, вам необходимо добавить _монитор_:

```ruby
# Первый аргумент — это класс, второй — название события, третий — имя метода
TestProf::EventProf.monitor(Work, "my.work", :do_smth)
```

После этого вы можете использовать EventProf как обычно:

```sh
EVENT_PROF=my.work bundle exec rake test
```

Доступны также следующие дополнительные опции при создании монитора:

- `top_level: true | false` (по умолчанию `false`): определяет, учитывать только события верхнего уровня или включая вложенные (описанное выше событие `'factory.create'` как раз [использует эту опцию](https://github.com/test-prof/test-prof/blob/master/lib/test_prof/event_prof/custom_events/factory_create.rb));
- `guard: Proc` (по умолчанию `nil`): вы можете указать объект типа Proc/Lambda, который будет вызываться перед инициацией события; событие будет отправлено, только если блок вернёт `true`; `guard` выполняется в контексте вызова метода (через `instance_exec`) и принимает на вход аргументы, переданные в наблюдаемый метод.

Рассмотрим пример:

```ruby
TestProf::EventProf.monitor(
  Sidekiq::Client,
  "sidekiq.inline",
  :raw_push,
  top_level: true,
  guard: ->(*) { Sidekiq::Testing.inline? }
)
```

Чтобы добавлять патчи мониторов _по запросу_ (т.е., только когда профилирование данного события включено), вы можете использовать специальный метод `TestProf::EventProf::CustomEvents.register`:

```ruby
TestProf::EventProf::CustomEvents.register("my.work") do
  TestProf::EventProf.monitor(Work, "my.work", :do_smth)
end
```

После регистрации всех событий необходимо переинициализировать EventProf, выполнив следующий код:

```ruby
TestProf::EventProf::CustomEvents.activate_all(TestProf::EventProf.config.event)
```

[Fabrication]: https://www.fabricationgem.org
