# Clang-Primer
A description of tricks  and tips when using Clang AST for developing tools such a clang-tidy checker

# Introduction

I recently embarked on writing a new clang-tidy checker. This was relatively simple process, however the devil was in the detail.

The most challenging part of the process was understanding how to use the
various Clang types,function and what they mean. Some are obvious and some are
not.

In my case I was looking to add a new modernize-use-nodiscard checker, I wanted
to be able to find non-void functions which had no arguments or took no "in"
arguments. I wanted to be able to identify when a return type should have been
consider by was ignored, to do that I needed to add `[[nodiscard]]` everywhere
I thought the return type was important.

The people in the clang team who reviewed the code were extremely helpful, but I
soon realized my checker had corner cases, trying to determine those corner
cases was hard, well not determining what I wanted to do, just how to do it

The purpose of this document is to collect some of those examples in a document that I can modify as I find better ways of doing things.  I'd hope that this might serve as a reminder to me, but also as a small set of snippets of code that you could use in your work.

I offer no guarantees of them being correct, I'm just trying to learn openly, so
expect mistakes, but feel free to correct them via pull requests.

Ok lets get started..

# Getting started

I don't really want to repeat the work of many other blogs on how to write your
own checker, there are really lots of good resources out there..suffice to say I
downloaded the clang source, built it, then ran add_new_checker.py in
clang-tidy..

I wanted to look for members of a class that I could add "[[nodiscard]]" to

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

Ok not rocket science, I'd never done any clang work before so everything is
new and most likely I'd make mistakes.. I started with a really simply matcher.

```c++
void UseNodiscardCheck::registerMatchers(MatchFinder *Finder) {

  Finder->addMatcher(
      cxxMethodDecl().bind("no_discard"),
      this);
} 

```

It didn't stay like this for long, but this matcher pulls out EVERY member function from every class... (thats quite a few in a large code base) I didn't really initially understand the matches syntax, but again there are better resources for finding that out but let me show you the process I went through to learn.

So I know that I want functions that return a value, so of course I don't want
to add `[[nodiscard]]` to void functions, so my first task was how can I
exclude void functions like this:

```c++
class Foo
{
    void clear();
};
```
So the final result is this, (don't worry about the bind("") part this isn't
important just yet, I'll try and remember to always put it on the next line, its
simply used to give this method a name so it can be retrieved later in the
actual check..

Back to the mathcher to remove void functions I needed the following code

```c++
  Finder->addMatcher(
      cxxMethodDecl(unless(returns(voidType())))
      .bind("no_discard"),
      this);
```

This was simply a matter of adding `unless(returns(voidType()))`, it took me
a bit to get this but you build up a bool predicate to filter the methods,

The actual checker is pretty trivial in this case, its assuming that everything,
that comes out of the matcher, is a candidate to add `[[nodiscard]]`, and so all
the work is done inside the matcher

```c++
void UseNodiscardCheck::check(const MatchFinder::MatchResult &Result) {
  const auto *MatchedDecl = Result.Nodes.getNodeAs<CXXMethodDecl>("no_discard");

  SourceLocation retLoc = MatchedDecl->getInnerLocStart();
  diag(retLoc, "function %0 should be marked [[nodiscard]]")
      << MatchedDecl
      << FixItHint::CreateInsertion(retLoc, "[[nodiscard]] ");
}
```

With the above function I'd get all member functions but not those that returned void, so this is a good start, but not every non-void function needs `[[nodiscard]]` infront of it, so how would I filter out those cases.

I had some rules that I wanted to use, I determined if a function were
const then a member function couldn't be altering anything on the class, and so
there was only a couple of ways of that function getting data out of the
function.

1. By returning the value
2. By being passed a non-const reference or a pointer to something that I could change

So it seems obvious, any non-void, const function that wasn't being passed
in any arguments should be [[nodiscard]] right?... well maybe.

So how do you do that at least...well there are a number of this functions like
`voidType()` and `returns(x)` that you can use, and once such example is `isConst()`. This will basically ask is the cxxMethodDecl in this case const.
Great lets use that, where do I put it..

Inside your matcher, you now want to logically combine the rules to build up a
more complex predicate filter for the matcher once such combiner is `allOf(x)`
another is `anyOf(x)`,  these are  variadic functions that allows you to provide multiple clauses (all/one of) of which must be true (respectively)

I'm sure by now you've noticed I used `unless(x)` that works just like not, so
`unless(returns(voidType()))` means "Doesn't return a void type"

so if I say

```c++
  Finder->addMatcher(
      cxxMethodDecl(allOf(unless(returns(voidType())),isConst()))
      .bind("no_discard"),
      this);
```

We are saying, "give me all the member functions which are const and don't return a void"

Similarily the following would choose all const member function and those that
did return void, (leaving out any non const,functions which return non void)

```c++
      cxxMethodDecl(anyOf(returns(voidType())),isConst()))
```

seems ok right? Well the simple cases are probably easy, but what about when
things get more complex. I haven't considered about the functions parameters yet

# Parameters



