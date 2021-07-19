
# Макросы в Elixir на примерах.

Данная статья рассчитана на тех, кто уже немного познакомился с [метапрограммированием](https://elixirschool.com/ru/lessons/advanced/metaprogramming/) в эликсире, но так и не понял, где же применять и творить магию.

## Повторим: quote и unquote

Выражения в эликсире представляют собой кортежи. Эти кортежи состоят из трех частей: название функции, метаданные и аргументы функции.

Например блок указанный ниже вернет список кортежей для каждого выражения, для List.last, для пайплайна и для Enum.sum.

```elixir
    quote do
      List.last(13, 19)
      |> Enum.sum(11)
    end
```

**quote** — возвращает внутренние структуры выражений

**unquote** - разворачивает внутри этих структур значения переданных переменных, тем самым позволяя указать в структуре кортежа название нужной функции или значение аргументов. За счет этого в эликсире обеспечивается метапрограммирование.

----

Рассмотрим несколько примеров, которые помогут лучше понять область применения макросов в эликсире.

## Пример №1

Внутри модулей мы часто используем значения из конфига приложения, обычно это делается командой:
```elixir
Application.get_env(:my_awesome_app, __MODULE__)[:link]
# Где **:my_awesome_app** — ключ otp_app;
# __MODULE__ — название модуля;
# :link — параметр конфига (ключ).
```

Эта довольно громоздкая конструкция начинает повторяться в разных модулях, что понижает читаемость кода. Тут нам на помощь приходят макросы.

Объявим на уровне приложения **следующий макрос**:
```elixir
defmodule MyAwesomeApp do
  defmacro get_config(key) do
    quote do
      Application.get_env(:my_awesome_app, __MODULE__)[unquote(key)]
    end
  end
end
```

Теперь в модулях приложения MyAwesomeApp можно получать значения конфига простой командой: `MyAwesomeApp.get_config(:link)`

```elixir
defmodule MyAwesomeApp.Foo do
  require MyAwesomeApp, only: :macros
  
  def host do
    MyAwesomeApp.get_config(:host)
  end
end
```

Функция **host** вернет значение host из конфига модуля Foo.

```elixir
config :my_awesome_app, MyAwesomeApp.Foo,
  host: "localhost"
```

## Пример №2

Давайте улучшим стандартный Logger, чтобы он сохранял чуть больше информации, чем обычно. Напишем абстракцию, которая дополнит стандартную запись в лог информацией о хосте и функции, из которой мы производим логирование.

Создадим модуль **MyLogger**
```elixir
defmodule MyLogger do

  defmacro info(message) do
    quote do
      require Logger

      {:ok, host} = :inet.gethostname
      {function, arity} = __ENV__.function
      text = unquote(message)
      Logger.info(
        "[host: #{host}, fun:#{function}/#{arity}] #{text}"
      )
    end
  end

  defmacro error(message) do
    ...
  end

  defmacro warm(message) do
    ...
  end
end
```

Теперь, когда мы будем пользоваться функцией info нашего Logger’а, в логах будет выводиться дополнительная информация.
```elixir
defmodule Foo do
  require MyLogger
  
  def bar(name) do
    # some actions
    MyLogger.info("Actions completed")
  end
end
```

Например при вызове **Foo.bar** в логах будет запись:
> 23:06:09.800 [info] [host: macbook_2017, fun: bar/1] Actions completed

## Пример №3

С помощью метапрограммирования мы можем динамически создавать функции, хоть макросы здесь и не пригодятся.

Напишем модуль, который возвращает целые числа от 1 до 99, где функция будет числовым литералом.
```elixir
defmodule Number do
  @decs [
    {20, "twenty"},
    {30, "thirty"},
    {40, "forty"},
    {50, "fifty"},
    {60, "sixty"},
    {70, "seventy"},
    {80, "eighty"},
    {90, "ninety"}
  ]

  @digs [
    {1, "one"},
    {2, "two"},
    {3, "three"},
    {4, "four"},
    {5, "five"},
    {6, "six"},
    {7, "sever"},
    {8, "eight"},
    {9, "nine"}
  ]

  @digs_btw_ten_and_twenty [
    {10, "ten"},
    {11, "eleven"},
    {12, "twelve"},
    {13, "thirteen"},
    {14, "fourteen"},
    {15, "fifteen"},
    {16, "sixteen"},
    {17, "seventeen"},
    {18, "eighteen"},
    {19, "nineteen"}
  ]

for {dec, dec_literal} <- @decs do
    for {dig, dig_literal} <- @digs do
      {dec + dig, "#{dec_literal}_#{dig_literal}"}
    end
  end
  |> List.insert_at(0, @digs_btw_ten_and_twenty)
  |> List.insert_at(0, @digs)
  |> List.insert_at(0, @decs)
  |> List.flatten
  *# Динамическое создание функций
  *|> Enum.each(fn {value, literal} ->
    def unquote(:"#{literal}")() do
      unquote(value)
    end
  end)
end
```

Теперь можем получать цифровые значения всего лишь обратившись к соответствующей функции:

```elixir
Number.sixty_one # 61
Number.forty_two # 42
Number.one # 1
```

Надеюсь данная статья помогла лучше освоить макросы и метапрограммирование в эликсире. Если остались какие-то вопросы, буду рад ответить на них в комментариях.

[Продолжение: про use и __using__](/posts/elixir_macroses_p2.md)
