---
layout: post
title:  "How I shoot in the foot with a case operator in Ruby"
date:   2017-08-26 20:23:00 +0300
categories: Dev
---
Before I started learning Ruby in 2010, I had been programming in Pascal, Delphi and
C++ to solve ACM-like problems. All of those programming languages have a
`switch`/`case` operator. In all of them it works pretty straightforward:
it compares a variable/object/result of an expression against several values and
decides which branch to execute.
When it was required for me in Ruby for a first time, I just looked up for the
`case` syntax. I thought, what could possibly go wrong with the `case` operator?


Recently I worked with Dropbox API to migrate an application to API v2.0. I
used the [opensource library](https://github.com/Jesus/dropbox_api) that has
been developing by a community. Despite most of the primary endpoints and
features are covered in this gem, I've found that it ignores
`media_info` field for a `DropboxApi::Metadata::File`. This field was urgently
important for my task, so I decided to modify the gem and send a pull request.

First of all, I needed to implement `Hash` type casting to `force_cast` method:
```ruby
def force_cast(object)
  if @type == String
    object.to_s
  elsif @type == Time
    Time.parse(object)
  elsif @type == Integer
    object.to_i
  elsif @type == Symbol
    object[".tag"].to_sym
  elsif @type == :boolean
    object.to_s == "true"
  elsif @type.ancestors.include? DropboxApi::Metadata::Base
    @type.new(object)
  else
    raise NotImplementedError, "Can't cast `#{@type}`"
  end
end
```

Instead of writing another `elsif` branch I decided to rewrite this method with
`case` operator because all the comparisons occur with the same object
`@type`. Code written with the `case` operator is easier to read. From
the first line it is obvious that there will be no conditions that
compare objects other than an argument.

At the first sight the problem could be in the last `elsif`, however
a lambda expression comes here to help us.

```ruby
def force_cast(object)
  case @type
  when String
    object.to_s
  when Time
    Time.parse(object)
  when Integer
    object.to_i
  when Symbol
    object[".tag"].to_sym
  when :boolean
    object.to_s == "true"
  when -> (t) { t.ancestors.include?(DropboxApi::Metadata::Base) }
    @type.new(object)
  else
    raise NotImplementedError, "Can't cast `#{@type}`"
  end
end
```

When I tested the result of my refactoring, I found that it is broken. The
`case` didn't match any branch and threw `NotImplementedError` exception. It was
the first time, when I read how `case` works in official Ruby documentation.


After all these years of programming in Ruby I've discovered that
case doesn't compare argument with `==`, it uses `===` operator on it.
This threequel operator doesn't have anything in common with `==`,
it's not an equality checking. The `===` operator checks if object on
the right side can be included into a set on the left side. In some cases it is
really simple, for example
```ruby
/qwe/ === 'qwerty' #=> true
```
because regular
expression `/qwe/` describes all possible strings that are matched. Or another
example
```ruby
(1..10) === 4 #=> true
```
integer 4 is included into the range 1..10.
```ruby
Integer === 4 #=> true
```
this is also true, because 4 is belong to all possible integers.
It is important to know that `===` operator is not symmetrical.
```ruby
4 === Integer #=> false
```

Lets move on to the tricky part:
```ruby
4 === 4 #=> true
```
Ruby is object-oriented programming language, it allows to write your own
implementation for a method to any class.
Lets check the `===` implementation for a `Fixnum` via pry `pry(main)> $ 4.===`:

```c
From: numeric.c (C Method):
Owner: Fixnum
Visibility: public
Number of lines: 15

static VALUE
fix_equal(VALUE x, VALUE y)
{
    if (x == y) return Qtrue;
    if (FIXNUM_P(y)) return Qfalse;
    else if (RB_TYPE_P(y, T_BIGNUM)) {
        return rb_big_eq(y, x);
    }
    else if (RB_TYPE_P(y, T_FLOAT)) {
        return rb_integer_float_eq(x, y);
    }
    else {
        return num_equal(x, y);
    }
}
```
Now it is clear. In Ruby `Fixnum` objects treat `===` as a simple equality check.
As for the human interpretation, I think that it can be represented like: "This
4 belongs to the set of all possible objects of 4".

```ruby
Integer === Integer #=> false
```
The root of my problem is covered in this line! I've used `case` to compare
types, but it doesn't work as I expected.
Let's jump straight to the implementation `$ Integer.===`:
```c
From: object.c (C Method):
Owner: Module
Visibility: public
Number of lines: 5

static VALUE
rb_mod_eqq(VALUE mod, VALUE arg)
{
    return rb_obj_is_kind_of(arg, mod);
}
```
It uses the standard check via `kind_of`, but recalling the analogy, it isn't
possible to put class `Integer` in the set of class `Integer`.

That's why my implementation of `force_cast` fails. It turns out that `case`
operator isn't suitable to compare classes. In the end, I had to revert my
refactoring and add another `elsif` condition.
