# FactoryAllStub

_Factory All Stub_ — это _заклинание_, которое заставляет FactoryBot/FactoryGirl использовать стратегию `build_stubbed` даже когда вы вызываете `create` или `build`.

Это позволяет одним движением «волшебной палочки» вылечить тесты, которые [Factory Doctor](../profilers/factory_doctor.md) считает «больными».

## Инструкция

Сначала необходимо явно активировать `FactoryAllStub`:

```ruby
TestProf::FactoryAllStub.init
```

В процессе инициализации TestProf _встраивается_ в логику генератора FactoryBot.

После этого необходимо также явно включить _all-stub_ режим:

```ruby
TestProf::FactoryAllStub.enable!
```

Для отключения _all-stub_ режима выполните:

```ruby
TestProf::FactoryAllStub.enable!
```

## Использование с RSpec

Добавьте в `spec_helper.rb` (или `rails_helper.rb`):

```ruby
require "test_prof/recipes/rspec/factory_all_stub"
```

Это автоматически активирует `FactoryAllStub` (не нужно делать `.init`) и добавить shared context
`"factory:stub"`, который вы сможете подключать вручную или с помощью тега `factory: :stub` для включения режима _all-stub_:

```ruby
describe "User" do
  let(:user) { create(:user) }

  it "is valid", factory: :stub do
    # будет использована стратегия `build_stubbed` вместо `create`
    expect(user).to be_valid
  end
end
```

**Бонус**: Для автоматического исправления тестов, обнаруженных `FactoryDoctor`, выполните следующую команду:

```sh
FDOC=1 FDOC_STAMP=factory:stub rspec ./spec/models
```
