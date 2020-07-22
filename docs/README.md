[![Gem Version](https://badge.fury.io/rb/test-prof.svg)](https://rubygems.org/gems/test-prof) [![Build](https://github.com/test-prof/test-prof/workflows/Build/badge.svg)](https://github.com/test-prof/test-prof/actions)
[![JRuby Build](https://github.com/test-prof/test-prof/workflows/JRuby%20Build/badge.svg)](https://github.com/test-prof/test-prof/actions)

# TestProf

> Коллекция инструментов для профилирования и оптимизации тестов приложений на Ruby

<img align="right" height="150" width="129"
     title="TestProf logo" class="home-logo" src="./assets/images/logo.svg">

TestProf представляет собой набор инструментов для анализа производительности тестов на Ruby, а так же быстрого исправления обнаруженных проблем.

Почему стоит беспокоиться о скорости выполнения тестов? В первую очередь, из-за того, что это является неотъемлемой частью цикла обратной связи (feedback loop) разработчика (см., например, [доклад](https://vimeo.com/145917204) по теме от [@searls](https://github.com/searls)), а так же одним из этапов выкатки приложений.

Проще говоря, медленные тесты отнимают ваше время, оказывая негативное влияние на общую продуктивность.

Задача TestProf — помочь вам обнаружить и устранить основные болевые точки в ваших тестах. Для этого предосталяются следующие возможности:

- Интеграция с профилировщиками общего назначения, такими как [`ruby-prof`](https://github.com/ruby-prof/ruby-prof) и [`stackprof`](https://github.com/tmm1/stackprof).

- Специальные инструменты для анализа использования фабрик в тестах (FactoryBot, Fabrication).

- Профилировщики _событий_ на основе ActiveSupport Notifications.

- [Расширения](#recipes) для RSpec и Minitest, позволяющие «ускорять» тесты.

- Копы для RuboCop.

- И многое другое.

<p align="center">
  <a href="http://bit.ly/test-prof-map">
    <img src="./assets/images/coggle.png" alt="TestProf map" width="738">
  </a>
</p>

<p align="center">
  <a href="https://evilmartians.com/?utm_source=test-prof">
    <img src="https://evilmartians.com/badges/sponsored-by-evil-martians.svg"
         alt="Sponsored by Evil Martians" width="236" height="54">
  </a>
</p>

## Кто уже использует TestProf

- [Discourse](https://github.com/discourse/discourse) уменьшили [скорость выполнения тестов на ~27%](https://twitter.com/samsaffron/status/1125602558024699904)
- [Gitlab](https://gitlab.com/gitlab-org/gitlab-ce) уменьшили [скорость выполнения тестов API на 39%](https://gitlab.com/gitlab-org/gitlab-ce/merge_requests/14370)
- [CodeTriage](https://github.com/codetriage/codetriage)
- [Dev.to](https://github.com/thepracticaldev/dev.to)
- [Open Project](https://github.com/opf/openproject)
- [...и другие](https://github.com/test-prof/test-prof/issues/73)

## Доклады с конференций

- RailsClub, Moscow, 2017, доклад "Тесты тоже должны быть быстрыми" [[видео](https://www.youtube.com/watch?v=8S7oHjEiVzs), [слайды](https://speakerdeck.com/palkan/railsclub-moscow-2017-faster-tests)]

- Paris.rb, 2018, "99 Problems of Slow Tests" (EN) [[видео](https://www.youtube.com/watch?v=eDMZS_fkRtk), [слайды](https://speakerdeck.com/palkan/paris-dot-rb-2018-99-problems-of-slow-tests)]

- BalkanRuby, 2018, "Take your slow tests to the doctor" (EN) [[видео](https://www.youtube.com/watch?v=rOcrme82vC8)], [слайды](https://speakerdeck.com/palkan/balkanruby-2018-take-your-slow-tests-to-the-doctor)]

- RubyConfBy, 2017, "Run Test Run" (EN) [[видео](https://www.youtube.com/watch?v=q52n4p0wkIs), [слайды](https://speakerdeck.com/palkan/rubyconfby-minsk-2017-run-test-run)]

## Публикации

- [TestProf: a good doctor for slow Ruby tests](https://evilmartians.com/chronicles/testprof-a-good-doctor-for-slow-ruby-tests)

- [TestProf II: Factory therapy for your Ruby tests](https://evilmartians.com/chronicles/testprof-2-factory-therapy-for-your-ruby-tests-rspec-minitest)

- [Tips to improve speed of your test suite](https://medium.com/appaloosa-store-engineering/tips-to-improve-speed-of-your-test-suite-8418b485205c) by [Benoit Tigeot](https://github.com/benoittgt)

## Установка и начало работы

Смотрите [документацию](./getting_started.md).

## Профилирование

- [Интеграция с RubyProf](./profilers/ruby_prof.md)

- [Интеграция со StackProf](./profilers/stack_prof.md)

- [Event Prof](./profilers/event_prof.md) (e.g. ActiveSupport notifications)

- [Tag Prof](./profilers/tag_prof.md)

- [Factory Doctor](./profilers/factory_doctor.md)

- [Factory Prof](./profilers/factory_prof.md)

- [RSpec Dissect](./profilers/rspec_dissect.md)

## Инструменты

- [`before_all`](./recipes/before_all.md)

- [`let_it_be`](./recipes/let_it_be.md)

- [AnyFixture](./recipes/any_fixture.md)

- [FactoryDefault](./recipes/factory_default.md)

- [FactoryAllStub](./recipes/factory_all_stub.md)

- [RSpec Stamp](./recipes/rspec_stamp.md)

- [Tests Sampling](./recipes/tests_sampling.md)

- [Active Record Shared Connection](./recipes/active_record_shared_connection.md)

- [Rails Logging](./recipes/logging.md)

## Прочее

- [RuboCop копы](./misc/rubocop.md)

## Что ещё?

Есть идея, что добавить в TestProf? Пожалуйста, [напишите нам](https://github.com/test-prof/test-prof/issues/new)!

Уже используете TestProf? [Поделитесь своей историей!](https://github.com/test-prof/test-prof/issues/73)

## Лицензия

Исходный код распространяется по лицензии [MIT](http://opensource.org/licenses/MIT).
