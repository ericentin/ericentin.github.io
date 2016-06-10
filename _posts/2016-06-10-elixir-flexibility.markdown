---
layout: post
title:  "Elixir's (Almost) Insane Flexibility: \"Mutable\" Iteration via Macros"
date:   2016-06-10 12:07:35 -0400
categories: elixir macros
---
## Comparing iteration in JavaScript and Elixir

In JavaScript (and languages like it), it is common to see code like this:

```javascript
// JavaScript

let array = [-1, 2, -3, 4]
let sum = 0

for (let i = 0; i < array.length; i++) {
  if (array[i] < 0)
    continue

  sum = sum + array[i]
}

assert(sum == 6)
```

This code is doing a pretty simple task. It calculates the sum of all elements in `a` which are positive. However, it's difficult to read at a glance, and this style of iteration is error prone.

An idiomatic translation of this code to Elixir would be:

```elixir
# Elixir

sum =
  [-1, 2, -3, 4]
  |> Enum.reject(fn elem -> elem < 0 end)
  |> Enum.sum()

assert sum == 6
```

The Elixir version is more declarative, and certainly easier to read. (To be fair, JavaScript now includes functions like `Array.prototype.filter` and `Array.prototype.reduce` that allow you to use a similar style.)

However, a new user might initially try something familiar from other languages and run into some problems:

```elixir
# Elixir

sum = 0

Enum.each [-1, 2, -3, 4], fn elem ->
  unless elem < 0 do
    sum = sum + elem
  end
end

assert sum == 6 # Failure: sum == 0
```

Why doesn't this work?

In languages like JavaScript, variables which are captured by a closure (or from enclosing scopes, like in the first example above) are captured by reference:

```javascript
// JavaScript

let a = 1

let get = () => a
let set = (value) => a = value

set(2)

assert(get() == 2)
```

However, in Elixir, variables are captured by value. When we refer to `sum` in the anonymous function we are passing to `Enum.each/2`, we are referring to the value that `sum` had at the time we created the function:

```elixir
# Elixir

a = 1

get = fn -> a end
set = fn value -> a = value end

set.(2)

assert get.() == 1

a = 2

assert get.() == 1
```

Additionally, in Elixir, even when we "reassign" a variable, we are not actually changing the original variable. Code like this:

```elixir
# Elixir

a = 1
a = 2

assert a == 2
```

is actually equivalent to the following code:

```erlang
% Erlang

A1 = 1,
A2 = 2,
assert(A2 =:= 2).
```

So, in our example before, `set` is not actually assigning to the original `a`. It's assigning to a "new" `a`, and the original `a` has not changed.

<!-- If we wanted to implement the original example in Elixir without using any standard library functions, we could do it like so:

```elixir
defmodule MySum do
  def sum_positive(list) do
    do_sum_positive(list, 0)
  end

  def do_sum_positive([elem | rest], acc) when elem < 0 do
    do_sum_positive(rest, acc)
  end

  def do_sum_positive([elem | rest], acc) do
    do_sum_positive(rest, acc + elem)
  end

  def do_sum_positive([], acc) do
    acc
  end
end

assert MySum.sum_positive([-1, 2, -3, 4]) == 6
```

All iteration in Elixir must be implemented using recursion. Even the functions in `Enum` are ultimately implemented using recursion. -->

## To each their own...

Given everything we just discussed, this code may be somewhat shocking:

```elixir
# Elixir

use MutableEach

sum = 0

each elem <- [-1, 2, -3, 4], mutable: {sum} do
  if elem < 0,
    do: continue

  sum = sum + elem
end

assert sum == 6
```

"How is this possible in Elixir? I thought we couldn't refer to variables by reference!", you might say. Well, thanks to the power of macros, almost [anything](https://github.com/wojtekmach/oop) is possible in Elixir.

Introducing: [`MutableEach`](https://github.com/antipax/mutable_each), a library which implements "mutable" iteration in Elixir.

Ultimately, the example above expands to something that looks somewhat like:

```elixir
sum = 0

{sum} =
  Enum.reduce_while [-1, 2, -3, 4], {sum}, fn elem, {sum} ->
    try do
      if elem < 0,
        do: throw {:mutable_each_continue, {sum}}

      sum = sum + elem
      {:cont, {sum}}
    catch
      {:mutable_each_continue, mutable} -> {:cont, mutable}
      {:mutable_each_break, mutable} -> {:halt, mutable}
    end
  end

assert sum == 6
```

`MutableEach` mostly relies on `Enum.reduce_while/3` and `throw`. Under the hood, there is still no actual mutability. The variables that are declared as "mutable" are simply provided as an accumulator within `reduce_while`, automatically returned at the end of each iteration, and then exported back into the original vars after the reduce is complete.

`continue` and `break` are implemented as macros which `throw` a tuple containing an atom representing the type of interrupt (`continue` or `break`) and the "mutable" variables. The function generated and passed to `reduce_while` contains a catch clause that returns `{:cont, values}` or `{:halt, values}` depending on the type of interrupt that was thrown.

Just another great example of how powerful and flexible Elixir is, thanks to macros.

## Don't try this at home

I probably shouldn't have to say this, but this was simply an experiment, and `MutableEach` should not be used in your Elixir code. Simply using `Enum.reduce_while/3` (no `throw` needed, probably!) directly in your own code is more explicit, and explicit is better than implicit.
