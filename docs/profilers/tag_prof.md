# TagProf

TagProf — это специальный профилировщик для RSpec, который позволяет собирать статистику выполнения тестов (время) с агрегацией по значению выбранного _тега_.

Например, если вы используете `rspec-rails` и настройку [`infer_spec_types_from_location!`](https://relishapp.com/rspec/rspec-rails/docs/directory-structure), тестам автоматически присваивается тег `type` (`controller`, `model` и т.д.). Используя TagProf, вы можете увидеть, какой тип тестов занимает больше всего времени.

Пример отчёта:

```sh
[TEST PROF INFO] TagProf report for type

       type          time   total  %total   %time           avg

    request     00:04.808      42   33.87   54.70     00:00.114
 controller     00:02.855      42   33.87   32.48     00:00.067
      model     00:01.127      40   32.26   12.82     00:00.028
```

В отчёты отображается общее число примеров и количество затраченного времени по группам, среднее время, а так же процент от всего времени прохождения тестов.

Вы также можете получить отчёт в HTML формате:

```sh
TAG_PROF=type TAG_PROF_FORMAT=html bundle exec rspec
```

В результате вы получите HTML с интерактивным графиком:

![TagProf UI](/assets/tag-prof.gif)

## Инструкция

TagProf работает только с RSpec.

Для активации профилировщика укажите переменную окружения `TAG_PROF`:

```sh
# значение — название тега, по которому группировать тесты
TAG_PROF=type rspec
```

## Интеграция с EventProf

TagProf интегрируется с [EventProf](./event_prof.md) для профилирования _событий_, а не только общего времени, затраченного на тесты.

Для этого необходимо дополнительно указать переменную окружения `TAG_PROF_EVENT` с идентификатором события (или событий через запятую):

```sh
TAG_PROF=type TAG_PROF_EVENT=sql.active_record rspec
```

В этом случае в отчёте появится дополнительная колонка (колонки):

```sh
[TEST PROF INFO] TagProf report for type

       type          time   sql.active_record  total  %total   %time           avg

    request     00:04.808           00:01.402     42   33.87   54.70     00:00.114
 controller     00:02.855           00:00.921     42   33.87   32.48     00:00.067
      model     00:01.127           00:00.446     40   32.26   12.82     00:00.028
```

## Бонус: больше типов

По умолчанию, `rspec-rails` добавляет тег `type` только для _классических_ сущностей из Rails (controllers, models, mailers и т.д.). В современных Rails приложениях, как правило, присутствуют и другие абстракции (например, services, forms, presenters и т.д.). Для того, чтобы автоматически добавить типы всем тестам в зависимости от их расположения, мы можем сконфигурировать RSpec следующим образом:

```ruby
RSpec.configure do |config|
  # ...
  config.define_derived_metadata(file_path: %r{/spec/}) do |metadata|
    # если тип уже указан — пропускаем
    next if metadata.key?(:type)

    match = metadata[:location].match(%r{/spec/([^/]+)/})
    metadata[:type] = match[1].singularize.to_sym
  end
end
```
