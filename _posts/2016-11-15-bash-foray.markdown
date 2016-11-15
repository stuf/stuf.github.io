---
layout: post
title: "Foray into shell scripting"
date: 2016-11-15 15:25
tags: scripting
---

I've been delving into the _wonderful magical world_ of shell scripting the during the past year and a half. During that time, I've been shuffling around some thoughts about how things work (or don't work) in general, and decided that I need to have an outlet for all of this stuff. Generally, I feel quite passionate about technology and the sorts, so it ends up becoming these long rants at inappropriate occasions. I'm not yet at the stage where I would go to a random person in the real world and talk about _the wonderfulness on infix operators in functional programming languages_ (and instead, hey, I'm doing that over the Internet right now), but I'm treading down that path if I don't do this.

Now, before you call me names and point out that "_this is how it's always been_", I'm writing this more as a means of performing [rubber duck debugging][wikipedia-rubber-duck-debugging] by and for myself, understanding (maybe) how something has ended up like it has, and maybe have a catharsis in the process. This is something for another piece, but I digress.

I am by far not an expert in shell scripting and there are most likely a number of best practices that I'm breaking—or slaughtering for that matter—since a lot of the things I do start off from "_How do I do [thing]?_". It's a constant process of revisiting previously written scripts—as best practices and scripting patterns start emerging—to see how those things can be improved in the future.

Also, a lot of these might not even be that hard for a lot of people in general, and it's by no means difficult for me either. I have a compulsive need to make sense of the world around me, and trying to understand _why_ something is like it is. Sometimes it doesn't seem to make any kind of sense, and there's a layer of implied history, if you will, that will make a lot of sense in regards to things.

And in the case of shell scripting, it turns out to be quite an interesting ride through the history of computing in general.

## How to even conditional

The first thing that took a while to realize, coming from what I'd consider "ordinary programming", is that shell scripting in `bash` (which I'm most familiar with now) revolves around the idea of return codes—or more elaborately—it revolves around performing actions, and acting based on the results of the invoked commands.

Of course, programming in general is about performing actions and acting upon the results of said actions. _But it's different and not at all the same thing_.

The concept of booleans don't exist in the traditional sense that you may have gotten used to while programming. You _can_ use a variable as a boolean value for many tasks that you do, but it might not be a clean way to do things.

My first scripts more or less looked like this:

{% highlight bash %}
#!/usr/bin/bash

WAS_SUCCESSFUL="0"

action --argument -q
if [ $? -eq 1 ]; then
    WAS_SUCCESSFUL="1"
fi

# ... later on in the script

if [ $WAS_SUCCESSFUL = "1" ]; then
    echo "Huge success!"
else
    echo "Just no."
fi
{% endhighlight %}

This is a very contrived example, of course, because in this case the entire thing could be combined.

It works. Kind of. You just want something like a ternary operation; perform action _x_, do this if it's successful, otherwise do that—or in other terms: `action == true ? success : fail`.

But if you're mostly familiar with a lot of programming languages, you realize that `$WAS_SUCCESSFUL = "1"` just feels... well, _it feels wrong_. Most likely, you've gotten used to associating the single equals sign (`=`) with _variable assignment_, and here you are merrily using that for comparison. And you might not even understand why `$?` is there and the concept of exit codes is still a mystery for you (this was me, just _hammer away_, think later).

But not only that, but then you use this at work somewhere and there's a `set -e` switch that's causing you a new set of troubles (`set -e` means that the script execution will be halted whenever an error or non-zero exit code happens), because after performing your `action` that fails, your script will exit.

Bah. So you search around for how to work around this, because you might not even think of it that much. "_Let it fail and let me do my stuff based on what the result is_" you think. So you end up turning off said termination flag by plopping in `set +e` before the action and then turning it back on with `set -e` afterwards.

{% highlight bash %}
#!/usr/bin/bash
set -e

WAS_SUCCESSFUL="0"

set +e
action --argument -q
if [ $? -ne 0 ]; then
    WAS_SUCCESSFUL="1"
fi
set -e

if [ $WAS_SUCCESSFUL = "1" ]; then
    echo "Huge success!"
else
    echo "Just no."
fi
{% endhighlight %}

As a programmer, you can't stop that nagging feeling that's telling you that _you're doing it wrong_. It can't be like that, can it? Of course it can't.

## No length like zero

So, unfamiliar with how your shell's inner workings you stumble upon a myriad of different solutions. You might stumble upon [a list of `bash` conditionals][bash-conditionals]

Maybe you just make an empty variable to give it a value, and decide based on that whether it worked or not:

{% highlight bash %}
#!/usr/bin/bash
DID_FAIL=

action --argument -q
if [ $? -ne 0 ]; then
    DID_FAIL=$?
fi

if [ ! -z $DID_FAIL ]; then
    echo "Fail."
else
    echo "Go on."
fi
{% endhighlight %}

No but that doesn't feel right either. It's still there. That feeling that _something isn't right_.

And then you get confused as to why there's `-eq` and `=` used here and there. This is shell country, so `-eq` will only work for numbers, while `=` only works for strings. You might even out of habit write `$? > 0`, and realize nothing works anymore, because the `>` operator will check if the _string_ on the left-hand side will sort after the _string_ on the right-hand side. You should've been doing `$? -gt 0` instead. How silly of you.

{% highlight bash %}
{% endhighlight %}

{% highlight bash %}
[[ action --argument -q ]] && echo "Yay!" || echo "Fail."
{% endhighlight %}

Someone says. Pure and simple. And what are those double angled braces doing there? Wasn't it just a single set of them? We'll look at that later on. And why is it using the command itself as a condition? From somewhere you remember that you can get the result of the command by doing `$(action --argument -q)`, but that doesn't work like you expect it to nor does it make sense either, and you're confused. And alone. And you cast doubt upon your very existence in the process.

But why is that used? Doesn't `set -e` mean it will terminate on error? That's because `set -e` doesn't cause script termination if it's in `until`, a `while` loop, an `if` test or a list construct.

`if` is preceded by a conditional statement, and desired actions are put into `then ... [elif] ... else` blocks. List constructs are commands chained together; `action --argument -q && another --action && ...`. A list construct can be one or more items, grouped by parentheses, and the items themselves may be just commands or squared-bracket conditionals.

So good news! Onwards to the next thing to do! But before that...

### The concept of return codes

If you've ever done a simple program in C, you might remember something like the following snippet.

{% highlight c %}
#include <stdio.h>

int main() {
    printf("Hello world!");
    return 0;
}
{% endhighlight %}

In case you've just looked into things as a hobby or just skimmed through stuff, you might not even give the `return 0` any more thought; it just seems as a convention.

That `return` statement tells the program to exit with that number. And shells _use that value_ to determine if whatever you just tried making the program do, succeeded, by (usually) return `0` for success, while anything else than `0` means that an error happened. Maybe you gave arguments that don't exist or weren't allowed, or maybe the program failed in its execution.

If the program or command has a `man` page, it may have a list of return codes to expect. For example, the `man` page for `grep` has this:

```
EXIT STATUS
     The grep utility exits with one of the following values:

     0     One or more lines were selected.
     1     No lines were selected.
     >1    An error occurred.
```

Whenever you run a command in the shell, it will give you a return code (which is the same as _exit status_, but for the sake of convention I will refer to statuses as _return codes_), which you can access through a special variable—namely `$?`. `$?` will always contain the return code of the last command you ran. Some programs might have a slew of exit codes for different scenarios, while others will just throw you `0` for success and `1` for failure.

And you _will_ get headaches from all that virtual banging-of-your-head-into-a-wall you're doing, since some programs might just throw a `0` to you on exit, even if the command did not succeed.

Why?

Because some people just want to watch the world burn.

### Conditionals, with conditions

The more you dabble with shell scripting, the more you will find oddities here and there that seem like they don't make a lot of sense. Why do we use `if ! command`, and then use square brackets later on and in that case, why is it a single or double square bracket?

Turns out, it goes a little bit deeper than I initially thought.

But I believe that for the time being, that will be the topic for another time.

[wikipedia-rubber-duck-debugging]: https://en.wikipedia.org/wiki/Rubber_duck_debugging
[bash-conditionals]: http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_07_01.html
