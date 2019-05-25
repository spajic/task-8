# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.
Успешное прохождение тестов занимало вплоть до 4-х минут. И это только для ~400 тестов.

Я решил исправить эту проблему, оптимизировав тестирование.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений.

Вот как я построил `feedback_loop`:
  - add guard gem
  - add terminal-notifier-guard

Конфигурация:
```
rspec_options = {
  cmd: "bundle exec rspec",
  run_all: {
    cmd: "bundle exec parallel_rspec -o '",
    cmd_additional_args: "'"
  }
}

guard :rspec, rspec_options do
  watch('spec/acceptance_helper.rb') { "spec/acceptance" }
  watch('app/controllers/application_controller.rb') do
    ["spec/requests", 'spec/acceptance']
  end

  watch(%r{^lib/(.+)\.rb$}) { |m| "spec/lib/#{m[1]}_spec.rb" }
  watch(%r{^app/(models|policies|services)/(.+)\.rb$}) do |m|
    "spec/#{m[1]}/#{m[2]}_spec.rb"
  end
  watch(%r{^app/controllers/(.+)/api_controller\.rb$}) do |m|
    ["spec/requests/#{m[1]}", "spec/acceptance/#{m[1]}"]
  end
  watch(%r{^app/controllers/(.+)\.rb$}) do |m|
    ["spec/requests/#{m[1]}_spec.rb", "spec/acceptance/#{m[1]}_spec.rb"]
  end
  watch(%r{^spec/(.+)_spec\.rb$}) { |m| "spec/#{m[1]}_spec.rb" }
  watch(%r{^spec/support/shared_(contexts|examples)/acceptance/(.+)\.rb$}) do |m|
    'spec/acceptance'
  end
  watch(%r{^spec/(support|factories|fixtures)/(.+)\.rb$}) { |m| 'spec' }
  watch(%r{^spec/(spec|rails)_helper\.rb$}) { 'spec' }
end

```

## Вникаем в детали системы, чтобы найти 20% точек роста
### Находка №1
Для начала я просмотрел gemfile на поиск используемых гемов, и увидел, что мы не используем `parallel_test`.
Это дало возможность сократить время прохождения тестов локально, но не на CI. Т.к. на данный момент
мы используем бесплатный сервис, который дает возможность прогонять тесты только в 1 поток.
После установки `parallel_test` время проходжения тестов локально сократилось в 4 раза.
С *4 minutes 2.8 seconds* до *1.2 минуты*. С установленной глобальной переменной `TEST_ENV_NUMBER=3`.

### Находка №2
Я также обнаружил, что мы используем логирование при тестировании. После чего я добавил возможность
управления логированием в test окружении с помощью глобальной переменной. Это измнение сократило время
прохождения тестов с *4 minutes 2.8 seconds* до *3 minutes 52.9 seconds*
(*здесь и далее время прохождения тестов будет указываться для тестирования без `parallel_test`*)

Для поиска проблемных мест в тестировании, был установлен `test-prof` и сформирована 2 вида отчета
в stack-prof: в json формате для разбора на *speedscope.app* и для *qcachegrind*.
C помощью команды: `TEST_STACK_PROF=1 TEST_STACK_PROF_FORMAT=json rspec`

### Находка №3
Я обнаружил, что 73% от всего времени занимает `BCrypt::Engine.hash_secret`.
После исправления данной настройки время сократилось с *3 minutes 52.9 seconds* до *1 minute 26.68 seconds*
Что сократило время тестирования почти в 3 раза.
Для `parallel_test` время составило: *40 seconds*

Также я получил отчет через `rspec-dissect` и получил топ тестов для let и before, которые стоит взглянуть.

### Находка №4
Я обнаружил, что в некоторых тестах я трачу довольно много времени на let. Потому я переписал его с использованием
let_it_be. Также заменил часть create на build_stubbed, create_default. Частично использовал anyfixture для
тестов, которые были помечены, как `factory: slow`. В итоге я смог добится ускорения тестов до *1 minute 13.86 seconds*

## Результаты
Смог увеличить скорость успешного прохождения тестов с *4 minutes 2.8 seconds* до *1 minute 13.86 seconds*.
Считаю, что это не предел и есть возможность ускорить ряд медленных тестов. Путем внедрения `bundle_stubbed` и `fixture`.
Также необходимо быть более осторожным при использовании `create_default`. Т.к. это добавляет некоторый слой магии в код,
хоть и усложняет выстрелить себе в ногу через factory cascade.

## TODO
Добиться ускорения тестов как минимум еще в 2 раза.