---
layout: post
title: "Casamento de padrões"
date: 2023-08-11 16:40:25
categories: Erlang Elixir FP TLDR
---

Você sabia que tanto Erlang como Elixir não possuem um operador de atribuição?
Ou seja, o que significa a seguinte expressão abaixo:
```elixir
message = "Hello World"
```

Mais conhecido como **Pattern Matching**, esse mecanismo te permite fazer associações de valores através
da comparação de símbolos entre o lado esquerdo do operador e o lado direito.

Na prática:
```elixir
iex> hello_world = "Hello World"
iex> hello_world
> "Hello World"

iex> "Hello " <> world = "Hello World"
iex> world
> "World"
```

Os 2 exemplos acima ilustram o funcionamento do casamento de padrões.
Onde a primeira execução aparenta ser uma simples atribuição, mas a partir da segunda, é possível perceber uma grande diferença conceitual.

No segundo exemplo a runtime irá tentar encontrar o valor `"Hello "` no valor do lado direito do operador, se esse valor for encontrado(deu match)
o resto do valor será armazenado na variável `world`.d

Quando essa assertiva não acontece, e o padrão não é encontrado, uma exception é gerada.

```elixir
iex> "Hellu " <> world = "Hello World"
> ** (MatchError) no match of right hand side value: "Hello World"
```

Essa é uma das minhas funcionalidades favoritas no ecossistema Erlang/Elixir, e neste texto vou tentar sintetizar um pouco do porquê.

Ex-1. Em Elixir, o casamento de padrões pode ser utilizado em Strings:
```elixir
# Apenas uma comparação e nada acontece, pois o padrão é encontrado (caso contrário, geraria uma exception)
"Hello World" = "Hello World"

# Assimila o valor "World" à variável world
"Hello " <> world = "Hello World"

# Mesmo resultado do exemplo acima, porém escrito de uma forma diferente. (usando operadores de bitstring)
<<"Hello ", world::binary>> = "Hello World"

# Assimila os primeiros 5 caracteres na variável hello, valida um espaço " ", e joga o resto da string na variável world
<<hello::binary-size(5), " ", world::binary>> = "Hello World"

# Como não existe nenhum padrão a ser encontrado, meio que simula uma atribuição.
hello_world = "Hello World"
```

É possível usar casamento de padrões com praticamente todos tipos, strings, listas, tuplas, maps, structs...

Ex-2. Testando outros tipos:
```elixir
# Apenas procura o padrão do lado esquerdo no lado direito do operador, o mesmo com as strings.
[1, 2, 3] = [1, 2, 3]

# Assimila o valor 2 na variável second.
[1, second, 3] = [1, 2, 3]

# Assimila o valor 1 na variável first, e descarta os demais valores. (operadores importantes que usamos para listas `++` e `|`)
[first | _] = [1, 2, 3]

# Ignora apenas o primeiro valor da lista, e assimila os demais na variável rest. (2, 3)
[_ | rest] = [1, 2, 3]

# Simplesmente joga todos os valores da direita do operador na variável list.
list = [1, 2, 3]
```

E o quão relevante é este tipo de operação no dia-a-dia?
Casamento de padrões pode ser utilizado em praticamente tudo, validação, polimorfia de funções, estruturas de controle, funções de ordem maior.

Ex-3. Em Elixir costumamos trabalhar bastante com o conceito de Tuplas de erro ou sucesso, e isso vale para requests,
comunicação externa, funções impuras, e por ai adiante...

Digamos que a execução de uma função qualquer do nosso código irá retornar uma tupla de sucesso, contendo `{status, person}`
Podemos tirar vantagem do casamento de padrões nesse cenário para criar uma tomada de decisão no nosso código.

Ilustração:
```elixir
{:ok, %{name: "Willian"}} = {:ok, %{name: "Willian"}}

# Assimila o valor "Willian" na variável name.
{:ok, %{name: name}} = {:ok, %{name: "Willian"}}

# Assimila todo o map da pessoa na variável person
{:ok, person} = {:ok, %{name: "Willian"}}

# Assimila :ok no status e o map pessoa na variável person
{status, person} = {:ok, %{name: "Willian"}}

# Joga todo o valor da direita do operador na variável tuple_result
tuple_result = {:ok, %{name: "Willian"}}
```

Criando uma tomada de decisão a partir desse valor:
```elixir
defmodule Person do
  def print_name(result) do
    case result do
      {:ok, %{name: name}} ->
        IO.puts(name)

      _ ->
        IO.puts("Houve um erro ao processar o resultado")
    end
  end
end

# se executarmos este código passando nosso resultado do exemplo anterior teríamos algo como:
iex> Person.print_name({:ok, %{name: "Willian"}})
> "Willian"
```

Agora com polimorfismo:
```elixir
defmodule Person do
  def print_name({:ok, %{name: name}}),
    do: IO.puts(name)

  def print_name(_),
    do: IO.puts("Houve um erro ao processar o resultado")
end
```

Em funções de ordem maior:
```elixir
list_results = [
    {:ok, %{name: "Willian"}},
    {:ok, %{name: "Willian"}},
    {:error, "Algo deu ruim"},
    {:ok, %{name: "Willian"}},
    {:ok, %{name: "Willian"}},
    {:ok, %{name: "Willian"}},
    {:error, "Algo deu ruim"}
]

res = Enum.filter(list_results, fn
  {:ok, %{name: name}} -> true
  {:error, _reason} -> false
end)
```

No caso acima, iremos filtrar da lista de resultados somente as tuplas de sucesso, e ignorar as que tiveram erro.


Concluindo, essa funcionalidade me permiti flexibilidade na hora de manipular dados e basicamente escrever toda estrutura do meu código,
tratamento de erros, estruturas condicionais, polimorfismo, e muito mais que não consigo nem lembrar direito.
