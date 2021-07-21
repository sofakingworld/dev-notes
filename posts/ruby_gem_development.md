
# Ruby — Разработка Gem’а в 2018 году.

> Данная статья рассчитана на тех, кому интересен язык Ruby, и хочется создать собственный Gem.*

Условные обозначения:
• gem — гем

В данной статье мы создадим гем расчета чисел Фибоначчи.

Прежде всего необходимо установить `bundler`, он все подготовит для разработки собственного гема.

```bash
gem install bundler
```
После установки вводим команду:

```bash
bundle gem <имя_разрабатываего_гема>
```

## Для создания гема bundler предложит

* выбрать инструмент для тестирования кода (rspec/minitest/none);

* выбрать тип распространения пакета (лицензия);

* создать пользовательское соглашение.

![Рис.1 Диалог создания гема](/images/ruby_gem_development:dialog.png)
`Диалог создания гема`

Рассмотрим созданный **bandler’ом** каркас поподробней:

![](/images/ruby_gem_development:folder.png)

* **bin/console** — запуск ruby консоли с загруженным гемом fib;
**bin/setup** — описание необходимых действий и зависимостей для * установки гема, в данном случае — bundle install;
* **lib/fib/version.rb** — константа, описывающая версию гема;
* **lib/fib/fib.rb** — основные функции гема;
* **spec/*_spec.rb** — тесты;
**spec/spec_helper.rb** — набор функций, необходимый для запуска * тестов;

* **.gitignore** — сообщает git, какие файлы (или шаблоны) он должен * игнорировать;
* **.rspec** — файл конфигурации тестов rspec;
* **.travis.yml** — файл конфигурации [CI](https://ru.wikipedia.org/wiki/Непрерывная_интеграция) travis;
* **fib.gemspec** — файл конфигурации/описания гема Fib;
* **Gemfile** — файл зависимостей/подключения gem’ов;
* **LICENSE.txt** — лицензия гема Fib;
* **Rakefile** — предоставляет DSL для написания задач на ruby, * которые можно запускать из командной строки;

Создадим репозиторий (на github, bitbucket и т.п.). Неважно где, он нам понадобится, чтобы указать домашнюю страницу в спецификации гема (fib.gemspec).

Теперь, когда bundler создал нам все необходимые файлы, заполняем информацию в файле спецификации fib.gemspec: указываем автора, его электронную почту, краткое и полное описание гема, домашнюю страницу.

```
    spec.authors = [“Dmitry Shpagin]
    spec.email = [“sofakingcode@gmail.com”]
    spec.summary = %q{This gem does calculate Fibonacci number}
    spec.description = %q{Long description}
    spec.homepage = “<ссылка_на_репозиторий>”
```

Давайте применим методологию [TDD](https://ru.wikipedia.org/wiki/Разработка_через_тестирование), при которой сначала пишется тест, а потом код. В папке spec генератором создан fib_spec.rb, в нем уже по умолчанию есть тест на проверку номера версии гема, добавим полезные для нас тесты: первое и второе числа Фибоначчи равны 1, третье равно 2, а двадцатое равно 6765.

```ruby
    it "Fib first number equal 1" do
      expect(Fib.number(1)).to eq(1)
    end

    it "Fib second number equal 1" do
      expect(Fib.number(2)).to eq(1)
    end

    it "Fib third number equal 2" do
      expect(Fib.number(3)).to eq(2)
    end

    it "Fib 20th number equal 6765" do
      expect(Fib.number(20)).to eq(6765)
    end
```

Дальше добавим в файл **lib/fib.rb** метод **number**, который принимает аргумент **n** (номер числа Фибоначчи). Запустим тесты командой **rspec**, тесты должны провалиться.

![Рис. 2. Тесты завершены с ошибкой](/images/ruby_gem_development:failed_tests.png)
`Тесты завершены с ошибкой`

Теперь допишем реализацию метода number. Листинг fib.rb представлен ниже.

```ruby
    module Fib
      extend self

      def number(n)
        if n > 2 then
          number(n-1)+number(n-2)
        else
          1
        end
      end
    end
```

Если заново запустим тесты, то они должны быть успешно пройдены.

![Рис. 3. Успешно пройденные тесты](/images/ruby_gem_development:success_tests.png)
`Успешно пройденные тесты`

На данном этапе наш гем готов. Далее перейдем в корень папки и выполним следующую команду:

```
    rake install
```

В результате гем будет собран и помещен в папку pkg. Он так же будет сразу установлен в систему.

Проверим работоспособность гема, запустим консоль руби:

```ruby
    # Подключим только что созданный гем
    require 'fib'

    Fib.number(15) => 610
```

[В следующей статье](/posts/ruby_gem_publish.md) мы доработаем гем в CLI пакет, после чего опубликуем его на [rubygems](https://rubygems.org).