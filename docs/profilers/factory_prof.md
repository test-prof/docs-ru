# FactoryProf

FactoryProf помогает вам анализировать использование фабрик в тестах: как часто и каким образом создаются те или иные объекты.
Это помогает находить _каскады фабрик_ — одну из самых главных причин медленных тестов.

> Подробнее о каскадах фабрик читайте в статье [TestProf II: Factory therapy for your Ruby tests](https://evilmartians.com/chronicles/testprof-2-factory-therapy-for-your-ruby-tests-rspec-minitest).

Пример отчёта FactoryProf:

```sh
[TEST PROF INFO] Factories usage

Total: 15285
Total top-level: 10286
Total time: 299.5937s
Total uniq factories: 119

 total   top-level   total time    time per call      top-level time            name
  6091        2715    115.7671s          0.0426s            50.2517s            user
  2142        2098     93.3152s          0.0444s            92.1915s            post
  ...
```

В данном отчёте отображается как общее число использования фабрики, так и отдельно число верхнеуровневых вызовов, т.е. без учёта вызовов внутри других фабрик (например, при создании ассоциированных объектов).

**NOTE**: FactoryProf учитывает только объекты, сохраняемы в базе данных, то есть объекты, созданные с помощью метода `#create` FactoryGirl/FactoryBot или Fabrication.

## Инструкция

FactoryProf работает как с FactoryGirl/FactoryBot, так и с Fabrication.

Для запуска профилирования используйте переменную окружения `FPROF`:

```sh
FPROF=1 rspec

# или
FPROF=1 bundle exec rake test
```

## Флеймграфы для фабрик

Наиболее полезным форматом отчёта FactoryProf является так называемый _FactoryFlame_ отчёт. Данный формат является адаптацией оригинальной идеи _флеймграфов_, [представленной Брэндоном Греггом](http://www.brendangregg.com/flamegraphs.html). Данный формат помогает **находить каскады фабрик**.

Для генерации FactoryFlame отчёте укажите `flamegraph` в качестве значение переменной `FPROF`:

```sh
FPROF=flamegraph rspec

# или
FPROF=flamegraph bundle exec rake test
```

В результате вы получите ссылку на HTML-файл с интерактивным графиком:

<img alt="FactoryFlame" data-origin="/assets/factory-flame.gif" src="/assets/factory-flame.gif">

Давайте разберёмся, какую информацию мы можем из него получить?

Каждая колонка представляет собой _стеб фабрик_ или _каскад_ — он начинается (снизу) с имени фабрики, которая была вызвана в тестовом коде, и продолжается цепочкой вложенных вызовов метода `#create`.

Рассмотрим пример:

```ruby
factory :comment do
  answer
  author
end

factory :answer do
  question
  author
end

factory :question do
  author
end
```

Допустим, мы хотим создать комментарий:

```ruby
create(:comment) #=> создаёт 5 объектов
```

Соответствующий стек фабрик выглядит следующим образом:

```ruby
[:comment, :answer, :question, :author, :author, :author]
```

Вернёмся к графику.

Ширина колонки соответствует _популярности_ данного стека: чем шире, тем чаще он встречается

Ячейка `root` показывает общее число вызовов метода `#create`.

Для оптимизации времени выполнения тестов вам нужно стремиться избавиться от одновременно _широких_ и _высоких_ стеков на графике.
