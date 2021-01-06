# FactoryDefault

_FactoryDefault_ — это механизм, который помогает бороться с _каскадами фабрик_ (см. [FactoryProf](../profilers/factory_prof.md)) путём переиспользования ассоциированных объектов.

**Примечание**: Поддерживается только FactoryGirl/FactoryBot.

Рассмотрим пример типичного SaaS-приложения с денормализацией по аккаунту. Допустим, у вас есть следующие фабрики:

```ruby
factory :account do
end

factory :user do
  account
end

factory :project do
  account
  user
end

factory :task do
  account
  project
  user
end
```

и следующий тест модели `Task`:

```ruby
describe "PATCH #update" do
  let(:task) { create(:task) }

  it "works" do
    patch :update, id: task.id, task: {completed: "t"}
    expect(response).to be_success
  end

  # ...
end
```

Сколько пользователей (_users_) и аккаунтов (_accounts_) будет создано при выполнении теста? Два и четыре соответственно.
Однако это противоречит логике: все объекты должны относится к одному аккаунту!

Мы могли бы это исправить так:

```ruby
describe "PATCH #update" do
  let(:account) { create(:account) }
  let(:project) { create(:project, account: account) }
  let(:task) { create(:task, project: project, account: account) }

  it "works" do
    patch :update, id: task.id, task: {completed: "t"}
    expect(response).to be_success
  end
end
```

Это будет работать. Но у данного решения есть свои недостатки: больше кода и больше вероятность ошибиться (забыть указать аккаунт для какого-то объекта).

С помощью FactoryDefault мы можем решить это так:

```ruby
describe "PATCH #update" do
  let(:account) { create_default(:account) }
  let(:project) { create_default(:project) }
  let(:task) { create(:task) }

  # Если нам нужны другие объекты, использующие ассоциации account/project,
  # мы пишем (в обоих случаях будет использован аккаунт, объявленный в начале группы тестов)
  let(:another_project) { create(:project) }
  let(:another_task) { create(:task, project: another_project) }

  it "works" do
    patch :update, id: task.id, task: {completed: "t"}
    expect(response).to be_success
  end
end
```

**Примечание**: данный инструмент добавляет немного _магии_ в тесты и должен быть использован с осторожностью. Хорошая идея – использовать значения по умолчанию только для _глобальных_ сущностей.

## Инструкция

В вашем `spec_helper.rb`:

```ruby
require "test_prof/recipes/rspec/factory_default"
```

После загрузки будет добавлено два новых метода для FactoryBot:

- `FactoryBot#set_factory_default(factory, object)` – устанавливает `object` объектом по умолчанию для ассоциаций с именем `factory`.

Например:

```ruby
let(:user) { create(:user) }

before { FactoryBot.set_factory_default(:user, user) }
```

- `FactoryBot#create_default(factory, *args)` – эквивалентно `create` + `set_factory_default`.

**Примечание**: Значения по умолчанию **автоматически сбрасываются после каждого теста**, что не будет работать с `before(:all)` / [`before_all`](./before_all.md) / [`let_it_be`](./let_it_be.md). Смотрите возможный вариант решения этой проблемы [здесь](https://github.com/test-prof/test-prof/issues/125#issuecomment-471706752).

### Использование с FactoryBot traits

Допустим, что у вас есть следующие фабрики:

```ruby
factory :post do
  association :user, factory: %i[user able_to_post]
end

factory :view do
  association :user, factory: %i[user unable_to_post_only_view]
end
```

и вы устанавливаете значение по умолчанию для фабрики `user` — оно будет использовано для обеих фабрик, несмотря на то, что используются разные _трейты_.

Вы можете воспользоваться опцией `FactoryDefault.preserve_traits = true` или указать для конкретного объекта
`create_default(:user, preserve_traits: true)`. В этом случае значение по умолчанию будет использовано только для ассоциаций **без трейтов**.
