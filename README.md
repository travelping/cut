# Cut

## Introduction

A syntax extension (i.e. parse transform) that adds support for _cuts_ to
Erlang. These are inspired by the [Scheme form of cuts](http://srfi.schemers.org/srfi-26/srfi-26.html).

Cuts can be thought of as a light-weight form of abstraction, with
similarities to partial application (or currying).

## Use

To use this parse-transformer, you must add the necessary
`-compile` attribute to your Erlang source files. For example:

    -module(test).
    -compile({parse_transform, cut}).
    ...

Then, when compiling `test.erl`, you must ensure `erlc` can locate `cut.beam`
by passing the suitable path to `erlc` with a `-pa` or `-pz` argument. For
example:

    erlc -Wall +debug_info -I ./include -pa ebin -o ebin  src/cut.erl
    erlc -Wall +debug_info -I ./include -pa test/ebin -pa ./ebin -o test/ebin test/src/test.erl

*Note*: If you're using QLC, you may find you need to be careful as to the
placement of the parse-transformer attributes. For example, I've found that
`-compile({parse_transform, cut}).` must occur before
`-include_lib("stdlib/include/qlc.hrl").`

## Motivation

The *cut* parse-transformer is motivated by the frequency with which simple
function abstractions are used in Erlang, and the relatively noisy
nature of declaring `fun`s. For example, it's quite common to see code
like:

    with_resource(Resource, Fun) ->
        case lookup_resource(Resource) of
            {ok, R}          -> Fun(R);
            {error, _} = Err -> Err
        end.

    my_fun(A, B, C) ->
        with_resource(A, fun (Resource) ->
                             my_resource_modification(Resource, B, C)
                         end).

That is, a `fun` is created in order to perform variable capture
from the surrounding scope but to leave holes for further
arguments to be provided. Using a *cut*, the function `my_fun` can be
rewritten as:

    my_fun(A, B, C) ->
        with_resource(A, my_resource_modification(_, B, C)).


## Definition

Normally, the variable `_` can only occur in patterns: that is, where a
match occurs. This can be in assignment, in cases, and in function
heads. For example:

    {_, bar} = {foo, bar}.

*Cut* uses `_` in expressions to indicate where abstraction should
occur. Abstraction from *cut*s is **always** performed on the
*shallowest* enclosing expression. For example:

    list_to_binary([1, 2, math:pow(2, _)]).

will create the expression

    list_to_binary([1, 2, fun (X) -> math:pow(2, X) end]).

and not

    fun (X) -> list_to_binary([1, 2, math:pow(2, X)]) end.

It is fine to use multiple *cut*s in the same expression, and the
arguments to the created abstraction will match the order in which the
`_` var is found in the expression. For example:

    assert_sum_3(X, Y, Z, Sum) when X + Y + Z == Sum -> ok;
    assert_sum_3(_X, _Y, _Z, _Sum) -> {error, not_sum}.

    test() ->
        Equals12 = assert_sum_3(_, _, _, 12),
        ok = Equals12(9, 2, 1).

It is perfectly legal to take *cut*s of *cut*s as the abstraction created
by the *cut* is a normal `fun` expression and thus can be re-*cut* as
necessary:

    test() ->
        Equals12 = assert_sum_3(_, _, _, 12),
        Equals5 = Equals12(_, _, 7),
        ok = Equals5(2, 3).

Note that because a simple `fun` is being constructed by the *cut*, the
arguments are evaluated prior to the *cut* function. For example:

    f1(_, _) -> io:format("in f1~n").

    test() ->
        F = f1(io:format("test line 1~n"), _),
        F(io:format("test line 2~n")).

will print out

    test line 2
    test line 1
    in f1

This is because the *cut* creates `fun (X) -> f1(io:format("test line
1~n"), X) end`. Thus it is clear that `X` must be evaluated first,
before the `fun` can be invoked.

Of course, no one would be crazy enough to have side-effects in
function argument expressions, so this will never cause any issues!

*Cut*s are not limited to function calls. They can be used in any
expression where they make sense:


### Tuples

    F = {_, 3},
    {a, 3} = F(a).


### Lists

    dbl_cons(List) -> [_, _ | List].

    test() ->
        F = dbl_cons([33]),
        [7, 8, 33] = F(7, 8).

Note that if you nest a list as a list tail in Erlang, it's still
treated as one expression. For example:

    A = [a, b | [c, d | [e]]]

is exactly the same (right from the Erlang parser onwards) as:

    A = [a, b, c, d, e]

That is, those sub-lists, when they're in the tail position, **do not**
form sub-expressions. Thus:

    F = [1, _, _, [_], 5 | [6, [_] | [_]]],
    %% This is the same as:
    %%  [1, _, _, [_], 5, 6, [_], _]
    [1, 2, 3, G, 5, 6, H, 8] = F(2, 3, 8),
    [4] = G(4),
    [7] = H(7).

However, be very clear about the difference between `,` and `|`: the
tail of a list is **only** defined following a `|`. Following a `,`,
you're just defining another list element.

    F = [_, [_]],
    %% This is **not** the same as [_, _] or its synonym: [_ | [_]]
    [a, G] = F(a),
    [b] = G(b).


### Records

    -record(vector, { x, y, z }).

    test() ->
        GetZ = _#vector.z,
        7 = GetZ(#vector { z = 7 }),
        SetX = _#vector{x = _},
        V = #vector{ x = 5, y = 4 } = SetX(#vector{ y = 4 }, 5).


### Case

    F = case _ of
            N when is_integer(N) -> N + N;
            N                    -> N
        end,
    10 = F(5),
    ok = F(ok).


See
[test_cut.erl](https://github.com/truqu/cut/blob/master/test/src/test_cut.erl)
for more examples, including the use of *cut*s in list comprehensions and
binary construction.

Note that *cut*s are not allowed where the result of the *cut* can only be
useful by interacting with the evaluation scope. For example:

    F = begin _, _, _ end.

This is not allowed, because the arguments to `F` would have to be
evaluated before the invocation of its body, which would then have no
effect, as they're already fully evaluated by that point.

## License

(The MPL)

Software distributed under the License is distributed on an "AS IS"
basis, WITHOUT WARRANTY OF ANY KIND, either express or implied. See
the License for the specific language governing rights and limitations
under the License.

The Original Code is Erlando.

The Initial Developer of the Original Code is VMware, Inc.
Copyright (c) 2011-2013 VMware, Inc.  All rights reserved.
