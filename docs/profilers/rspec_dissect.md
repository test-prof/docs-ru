# RSpecDissect

Вы когда-нибудь задавались вопросом, сколько времени ваших тестов уходит на подготовку контекста в `before` хука? Или для вычисления значений ленивых `let`? Обычно, именно _там_ большую часть времени проводят ваши тесты.

_RSpecDissect_ — это специальный профилировщик для RSpec, который показывает вам эту информацию и выводит список групп тестов, наиболее _злоупотребляющих_ использованием данных конструкций. Основная задача данного профилировщика — обнаружить тесты, которые можно ускорить, применив [`before_all`](../recipes/before_all.md) или [`let_it_be`](../recipes/let_it_be.md).

Пример отчёта:

```sh
[TEST PROF INFO] RSpecDissect enabled

Total time: 25:14.870
Total `before(:each)` time: 14:36.482
Total `let` time: 19:20.259

Top 5 slowest suites (by `before(:each)` time):

Webhooks::DispatchTransition (./spec/services/webhooks/dispatch_transition_spec.rb:3) – 00:29.895 of 00:33.706 (327)
FunnelsController (./spec/controllers/funnels_controller_spec.rb:3) – 00:22.117 of 00:43.649 (133)
ApplicantsController (./spec/controllers/applicants_controller_spec.rb:3) – 00:21.220 of 00:41.407 (222)
BookedSlotsController (./spec/controllers/booked_slots_controller_spec.rb:3) – 00:15.729 of 00:27.893 (50)
Analytics::Wor...rsion::Summary (./spec/services/analytics/workflow_conversion/summary_spec.rb:3) – 00:15.383 of 00:15.914 (12)


Top 5 slowest suites (by `let` time):

FunnelsController (./spec/controllers/funnels_controller_spec.rb:3) – 00:38.532 of 00:43.649 (133)
 ↳ user – 3
 ↳ funnel – 2
ApplicantsController (./spec/controllers/applicants_controller_spec.rb:3) – 00:33.252 of 00:41.407 (222)
 ↳ user – 10
 ↳ funnel – 5
 ↳ applicant – 2
Webhooks::DispatchTransition (./spec/services/webhooks/dispatch_transition_spec.rb:3) – 00:30.320 of 00:33.706 (327)
 ↳ user – 30
BookedSlotsController (./spec/controllers/booked_slots_controller_spec.rb:3) – 00:25.710 of 00:27.893 e(50)
 ↳ user – 21
 ↳ stage – 14
AvailableSlotsController (./spec/controllers/available_slots_controller_spec.rb:3) – 00:18.481 of 00:23.366 (85)
 ↳ user – 15
 ↳ stage – 10
```

Как можно заметить выше, статистика использования `let` также включает информацию о количестве вызовов каждого определения (по умолчанию показываются только top-3 самых популярных).

## Инструкции

RSpecDissect работает только с RSpec (как можно было догадаться из названия).

Для активации RSpecDissect используйте переменную окружения `RD_PROF`:

```sh
RD_PROF=1 rspec ...
```

Вы также можете указать, какое количество групп тестов выводить в отчёте, с помощью переменной окружения `RD_PROF_TOP`:

```sh
RD_PROF=1 RD_PROF_TOP=10 rspec ...
```

Если вы хотите анализировать только `let` или только `before`, используйте `RD_PROF=let` и `RD_PROF=before` соответственно.

Вы можете изменить максимальное число выводимых в отчёте `let`-определений с помощью переменной `RD_PROF_LET_TOP` (например, ``RD_PROF_LET_TOP=10`).

Наконец, чтобы отключить сбор статистики по `let`-определениям, укажите в конфигурации:

```ruby
TestProf::RSpecDissect.configure do |config|
  config.let_stats_enabled = false
end
```

## Интеграция с RSpecStamp

RSpecDissect интегрируется с [RSpec Stamp](../recipes/rspec_stamp.md) для автоматической пометки _медленных_ тестов тегами. Например:

```sh
RD_PROF=1 RD_PROF_STAMP="slow:setup" rspec ...
```

После запуска тестов с данными параметрами самые _медленные_ группы тестов будут помечены тегом `slow: :setup`.
