# Clang-Primer
A description of tricks  and tips when using Clang AST for tool such a clang-tidy

I recently embarked on writing a new clang-tidy checker, by and large what it
was relatively simple process.. but the devil was in the detail..

The most challenging part of the Process was understanding how to use the
various Decls and Types to do something pretty simple.

In my case I was looking to add a new modernize-use-nodiscard checker, I wanted
to be able to find non-void functions which had no arguments or took on "in"
arguments

The people who reviewed the code were extremely helpful, great people..but I
soon realized my checker had corner cases, trying to determine those corner
cases was hard, well not determining what I wanted to do, just how to do it

The purpose of this document is to collect some of those examples in a document that I can modify as I find better ways of doing things.  I'd hope that this might serve as a reminder to me, but also as a small set of snippets of code that you could use in your work.

I offer no gaurantees of them being correct, I'm just trying to learn openly, so
expect mistakes, but feel free to correct them via pull requests

Ok lets get started..

# Getting started

I don't really want to repeat the work of many other blogs on how to write your
own checker, there are really lots of good resources out there..suffice to say I
downloaded the clang source, built it, then ran add_new_checker.py in
clang-tidy..

Ok I wanted to look for members of a class that I could add "[[nodiscard]]" to

transforming this..
```c++
class Foo
{
    bool empty() const;
};
```

into this...

```c++
class Foo
{
    [[nodiscard]] bool empty() const;
};
```

Ok not rocket science, I've never done any clang work before so everything is
new and most likely I'd make mistakes

```c++
void UseNodiscardCheck::registerMatchers(MatchFinder *Finder) {

  Finder->addMatcher(
      cxxMethodDecl().bind("no_discard"),
      this);
} 

```

It didn't stay like this for long, but matcher pulls out EVERY member function from every class... (thats quite a few in a large code base) I didn't really initially understand the matches syntax, but again there are better resources for finding that out but let me show you the process I went through to learn.

So I know that I want functions that return a value, so of course I don't want
to add `[[nodiscard]]` to void functions

```c++
  Finder->addMatcher(
      cxxMethodDecl(unless(returns(voidType()))).bind("no_discard"),
      this);
```

This was simply a matter of adding `unless(returns(voidType()))`, it took me
abit to get this but you basically build up predicates to filter the methods

Now I'd get all member functions but not those that returned void, so this is a
good start, but no every non void function needs `[[nodiscard]]` infront of
it, so how would I filter hits some more

Firstly I had some rules that I wanted to use, I determined if a function were
const then a member function couldn't be altering anything on the class, and so
there was only a couple of ways of that function getting data out of the
function.

1. By returning the value
2. By being given a non-const reference or a pointer to something that I could change

So it seems obvious, any non-void, non-const function that wasn't being passed
in any argument should be [[nodiscard]] right?... well maybe

So how do you do that...well there are a number of this functions like
`voidType()` and `returns(x)` that you can use for example once such function is
`isConst()`. Great lets use that, where do I put it..

inside your matcher, you now want to logically combine the rules to build up a
more complex predicate filter for the matcher once such combiner is `allOf(x)`
this is likely a variadic function that allows you to provide multiple clauses
all of which must be true.

I'm sure by now you've noticed I used `unless(x)` that works just like not, so
`unless(returns(voidType()))` means "Doesn't return a void type"

so if I say

```c++
  Finder->addMatcher(
      cxxMethodDecl(allOf(unless(returns(voidType())),isConst()))
      .bind("no_discard"),
      this);
```

We are saying, "give me all the member function which are const and don't return
a void"

seems ok right? Well the simple cases are probably easy, but what about when
things get more complex. I haven't considered about the functions parameters yet





