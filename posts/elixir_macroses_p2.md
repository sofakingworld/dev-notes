
# Макросы в Elixir на примерах (ч. 2)

WARNING: Очень много кода

Данная статья является продолжением [предыдущей](/posts/elixir_macroses_p1.md). 

На этот раз мы на примерах рассмотрим директиву **use** и магический метод **__using__**, магический, потому что подобные методы с двумя подчеркиваниями используют в Питоне, называя магическими.

Довольно часто мы пишем **use GenServer**, или **require Logger**, давайте разберемся в чем же разница

## Различие между require и use

Если объяснять на пальцах, то **use** расширяет модуль, в котором он используется: дополняет его методами и зависимостями, описанными в __using__, поэтому может возникнуть конфликт имен. Используйте use осторожно.

Пример:
```elixir
defmodule Extending do
  # мы указываем _any, потому что по умолчанию use 
  # передает дополнительный аргумент, для указания,
  # что именно мы хотим использовать. В данном 
  # случае нам такая конкретика не нужна
  defmacro __using__(_any) do
    quote do
      def twenty_five do
        25
      end
    end
  end
end

defmodule Foo do
  use Extending
end
```

```elixir
Foo.twenty_five #=> 25
```

Тем самым мы расширили модуль **Foo**, добавив в него функцию **twenty_five.**

**require** компилирует макросы, описанные в модуле, и обращаться к ним можно только через namespace модуля. На примере того же Logger’а:

```elixir
defmodule Foo do
  # запрашиваем логер
  require Logger
  
  def bar do
    # используем логер через namespace Logger      
    Logger.info("executing bar")
    ...
  end
end
```

На примере **my_awesome_app_web.ex** — модуль феникса, предоставляющий интерфейсы для view, controller, channel, router. Рассмотрим использование директивы using, и разберемся, почему она так полезна. После этого одной загадкой станет меньше, и в фениксе для вас не останется никакой магии.

```elixir
defmodule MyAwesomeAppWeb do
  @moduledoc """
  This can be used in your application as:

      use MyAwesomeAppWeb, :controller
      use MyAwesomeAppWeb, :view

  """

  def controller do
    quote do
      use Phoenix.Controller, namespace: MyAwesomeAppWeb
      ...
    end
  end

  def view do
    quote do
      ...
    end
  end

  def router do
    quote do
      ...
    end
  end

  def channel do
    quote do
      ...
    end
  end

  @doc """
  When used, dispatch to the appropriate controller/view/etc.
  """
  defmacro __using__(which) when is_atom(which) do
    apply(__MODULE__, which, [])
  end
end

```
Чем нам в данном случае помогает **using**? А тем, что мы не описываем в каждом представлении, контроллере, канале и маршруте список импортируемых и используемых модулей. Так же, например, мы можем расширить все контроллеры какой-либо функцией, добавив её в фукнцию **controller.**

----

## __using__ на примерах:

## Пример №1

Разделим роутер нашего веб-приложения на несколько файлов, после чего элегантно объединим их в одном файле.

Представим, что у нас довольно большое приложение, в котором есть маршруты для админки, api и browser-клиента. Они все описаны в одном файле на 500 строк. Такой файл неудобно поддерживать, особенно когда маршруты можно разделить на части по специфике контекста.

```elixir
    defmodule MyAwesomeAppWeb.RouterBrowser do
      defmacro __using__(_) do
        quote do
          use MyAwesomeAppWeb, :router

          pipeline :browser do
            plug(:accepts, ["html"])
            plug(:fetch_session)
            plug(:fetch_flash)
            plug(:protect_from_forgery)
            plug(:put_secure_browser_headers)
          end

          scope "/", MyAwesomeAppWeb do
            # Use the default browser stack
            pipe_through(:browser)

            resources("/", PageController)
          end    
        end
      end
    end

    defmodule MyAwesomeAppWeb.RouterApi do
      defmacro __using__(_) do
        quote do
          use MyAwesomeAppWeb, :router

          pipeline :api do
            plug(:accepts, ["json"])
          end
          
          scope "/api", MyAwesomeAppWeb do
            pipe_through(:api)

            resources("/", ApiController)
          end 
        end
      end
    end

    defmodule MyAwesomeAppWeb.RouterAdmin do
      defmacro __using__(_) do
        quote do
          use MyAwesomeAppWeb, :router

          pipeline :admin do
            # special check or something like
            plug(:check_admin_credentials)
          end

          scope "/admin", MyAwesomeAppWeb do
            pipe_through([:browser, :admin])

            resources("/", DashboardController)
          end    
        end
      end
    end
```

В итоговом файле осталось всего лишь использовать наши модули с маршрутами:

```elixir
    defmodule MyAwesomeAppWeb.Router do
      use MyAwesomeAppWeb.RouterApi
      # Browser указываем до Admin, потому что в Admin
      # используется pipeline :browser
      use MyAwesomeAppWeb.RouterBrowser
      use MyAwesomeAppWeb.RouterAdmin
    end
```

Теперь мы можем помещать какие-то специфичные маршруты в разные файлы и использовать их в роутере, не нагромождая тем самым файл роутера.

## Пример №2

Расширим стандартный GenServer, чтобы он по умолчанию имел функцию именованного запуска и запускался сразу с Logger’ом, который логирует информацию о запуске.

```elixir
    defmodule NamedGenServer do 
      defmacro __using__(_opts) do
        quote do 
          use GenServer
          require Logger
          
          def start_link(initial_state \\ nil) do
            process = GenServer.start_link(__MODULE__, initial_state, [name: __MODULE__])
            Logger.info "#{__MODULE__} started"
            process
          end
        end
      end
    end
```

И так, мы написали обертку над GenServer’ом, которая дополнительно подключает Logger и логирует информацию о запуске. Давайте применим её при создании GenServer’a.
```elixir
    defmodule SomeGenServer do
      use NamedGenServer

      def handle_call({:show_state}, _from, state) do
        {:reply, state, state}
      end
    end
```

Теперь у нас нет необходимости повторно указывать require Logger, он уже доступен в данном модуле.

Давайте запустим NamedGenServer и посмотрим как он работает:
```elixir
    SomeGenServer.start_link(2)

    IO.puts "1"
    IO.puts GenServer.call(SomeGenServer, {:show_state})
    IO.puts "3"

    ## OUTPUT
    # 00:12:34.567 [info]  Elixir.SomeGenServer started
    # 1
    # 2
    # 3
```
