# Копы для RuboCop

TestProf включает в себя специальные копы для [RuboCop](https://github.com/bbatsov/rubocop), помогающие писать тесты более эффективно.

Для подключения добавьте `test_prof/rubocop` в список загружаемых файлов в вашем конфигурационном файле:

```yml
# .rubocop.yml
require:
 - 'test_prof/rubocop'
```

Вы также можете подключить их динамически:

```sh
bundle exec rubocop -r 'test_prof/rubocop' --only RSpec/AggregateExamples
```

## RSpec/AggregateExamples

Данный коп призывает разработчиков использовать специальную возможность RSpec — возможность агрегировать несколько ошибок внутри одного примера.

Вместо того, чтобы слепо следовать правилу «один тест — одна проверка», можно подойти к вопросу с умом и объединить _независимые_ проверки в одном примере и тем самым сэкономить время на подготовке окружения для теста.

Рассмотрим пример:

```ruby
# плохо
it { is_expected.to be_success }
it { is_expected.to have_header("X-TOTAL-PAGES", 10) }
it { is_expected.to have_header("X-NEXT-PAGE", 2) }

# rspec-its также поддерживается
its(:status) { is_expected.to eq(200) }

# хорошо
it "returns the second page", :aggregate_failures do
  is_expected.to be_success
  is_expected.to have_header("X-TOTAL-PAGES", 10)
  is_expected.to have_header("X-NEXT-PAGE", 2)
  expect(subject.status).to eq(200)
end
```

Данный коп помогает находить подобные примеры, а также поддерживает **автоматическую коррекцию**: примеры объединяются в один, добавляется тег `:aggregate_failures`. Если в вашем проекте настроена агрегация по умолчанию, вы можете отключить автоматическое добавление тегов указав в конфигурации RuboCop для копа `RSpec/AggregateExamples` опцию `AddAggregateFailuresMetadata: false`.

**Примечание**: авто-коррекция примеров, использующих блоковые проверки (block matchers), такие как `change`, не поддерживается.
