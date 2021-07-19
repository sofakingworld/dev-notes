
# Как правильно готовить GenServer


## Обозначения:
 - GenServer — генсервер
 - state — стейт (состояние)
 - бизнес-логика — любые важные вычисления

___

Довольно часто эликсирщики описывают бизнес-логику в генсервере, при этом сами загоняют себя в ловушку.

Ловушка связана с тестированием, т.к. невозможно правильно протестировать процесс, а его стейт будет меняться при выполнении тех или иных тестов, особенно если генсервер запускается с приложением и играет ключевую роль.

Нередко эту проблему пытаются решить с помощью пауз (**:timer.sleep**), либо описывают в одном тесте несколько действий, пытаясь добиться последовательного изменения состояния без артефактов от других тестов.

И то, и другое решение не дает стабильности, и тесты время от времени завершаются с ошибкой.

## Так что же делать?

Все довольно просто — не надо описывать бизнес логику в генсервере. Вся бизнес логика должна лежать в соответствующем модуле, а генсервер должен представлять из себя хранилище стейта и набор коллбэков, которые вызывают функции модуля.

## Какие это дает преимущества?

Отпадает необходимость тестировать генсервер, а именно:

1. не нужно создавать процесс генсервера для тестов

2. нет необходимости контролировать его стейт между конкурирующими тестами

3. не нужно имитировать цепочку действий для достижения нужного состояния генсервера

Нам достаточно покрыть тестами модуль с бизнес логикой

--- 
## Пример

Рассмотрим простой генсервер, реализующий стек.

```elixir
defmodule StackGenserver do
  use GenServer

  # Client

  def start_link(default) when is_list(default) do
    GenServer.start_link(__MODULE__, default)
  end

  def push(pid, item) do
    GenServer.call(pid, {:push, item})
  end

  def pop(pid) do
    GenServer.call(pid, :pop)
  end

  # Server (callbacks)

  @impl true
  def init(stack) do
    {:ok, stack}
  end

  @impl true
  def handle_call(:pop, _from, state) do
    {value, list} = List.pop_at(state, 0)

    {:reply, value, list}
  end

  @impl true
  def handle_call({:push, item}, _from, state) do
    {:reply, [item | state], [item | state]}
  end
end
```

В нем есть функции клиентской части, которые принимают pid процесса, и дополнительный аргумент, в зависимости от функции. Сами же функции и их реализация описаны в коллбэках.

```elixir
defmodule StackGenserverTest do
  use ExUnit.Case

  test "pops first item" do
    {:ok, pid} = StackGenserver.start_link([1, 2, 3])
    assert 1 == StackGenserver.pop(pid)
  end

  test "pops empty list" do
    {:ok, pid} = StackGenserver.start_link([])
    assert nil == StackGenserver.pop(pid)
  end

  test "push to empty list" do
    {:ok, pid} = StackGenserver.start_link([])
    assert [3] == StackGenserver.push(pid, 3)
  end

  test "push to list begining" do
    {:ok, pid} = StackGenserver.start_link([1, 2])
    assert [0, 1, 2] = StackGenserver.push(pid, 0)
  end
end
```

Поэтому, чтобы протестировать работу данного генсервера, а именно возврат ожидаемых значений — необходимо в каждом тесте создавать процесс генсервера, с заданным начальным состоянием, и это скорее минус, чем плюс.


Можно вынести логику в отдельный модуль, методы которого будут вызываться в нашем генсервере

```elixir
defmodule StackImp do
  def pop(list) do
    # Возвращает кортеж с вынутым значением, и остатком списка.
    # подходит в качестве ответа для генсервера
    List.pop_at(list, 0)
  end

  def push(list, item) do
    new_list = [item | list]

    # Дублирем значения в кортеже, т.к. первое будет использоваться
    # в качестве ответа, а второе - новое состояние генсервера
    {new_list, new_list}
  end
end
```

Перепишем генсервер: теперь нам достаточно одного коллбэка. В нем функция **apply** применит на указанном модуле нужную функцию, первым аргументом передаст текущий стейт генсервера и дополнительные аргументы, если они были указаны.

```elixir
defmodule Stack do
  use GenServer

  # Client

  def start_link(default) when is_list(default) do
    GenServer.start_link(__MODULE__, default)
  end

  def push(pid, item) do
    GenServer.call(pid, {:push, [item]})
  end

  def pop(pid) do
    GenServer.call(pid, {:pop, []})
  end

  # Server (callback)

  def handle_call({function_name, arguments}, from, state) do
    # Применяем функцию на нужном модуле
    # и используем результат в качестве ответа и обновленного состояния
    {reply, new_state} = apply(StackImp, function_name, [state] ++ arguments)
    
    {:reply, reply, new_state}
  end
end
```

Для тестирования нам остается покрыть тестами наш модуль, который вызывается в генсервере. В тестах первым аргументом передаём “текущий стейт”, а остальные аргументы перечисляем через запятую.

```elixir
defmodule StackImpTest do
  use ExUnit.Case
  
  test "pops first item" do
    assert {1, [2, 3]} = StackImp.pop([1,2,3])
  end

  test "pops empty list" do
    assert {nil, []} = StackImp.pop([])
  end

  test "push to empty list" do
    assert {[3], [3]} = StackImp.push([], 3)
  end

  test "push to list begining" do
    assert {[0, 1, 2], [0, 1, 2]} = StackImp.push([1, 2], 0)
  end
end
```

Инкапсулируя логику в модулях, мы упрощаем тесты, подставляя возможные значения стейта. Дополнительно мы проверяем что будет возвращено и что будет записано в обновленный стейт. Теперь не нужно тестировать поведение генсервера и каждый раз создавать новый процесс.
