---
layout:     post
title:      "Validating form data in Calmm.js"
subtitle:   "Or: the age-old problem of how do I mark this stuff as invalid???"
date:       2017-06-21 14:26
categories: javascript react frp calmm streams
---

<blockquote class="blockquote">
  <p>
    I have a form with fields. I need to check if the input is valid based on some arbitrary set of rules before letting the user submit it. What library should I use?
  </p>
  <footer class="blockquote-footer">
    Most likely every web developer ever
  </footer>
</blockquote>

## Preface

The problem with form validation is not that it's an inherently difficult thing to do. On the contrary: it's a somewhat trivial problem—in most cases—that is usually solved by approaching the problem from the wrong angle. Add a multitude of different form libraries on top of that—going from form validation to entire form components—that all promise simple fixes to a problem that a lot of times are a result of some project where it was first used. Usually they enforce the coder to write boilerplate or change how the data is being structured and stored to fit its API.

Form validation continues to be a tricky problem, because rarely any two projects and how they are structured are the same. The [Calmm stack][calmm] provides us with some tooling that help us alleviate this pain and allows us to create simple components that work independently of how the data is structured.

This approach is not the silver bullet you're looking for; the other reason why form validation continues to be a pain in the ass is that very rarely are any forms from project to project the same.

I've had the luck of working more or less full-time using the Calmm stack since I started working at [Siili][] the beginning of March in 2017. It's been a continuous learning process of learning, relearning, and rewriting _a lot_ of code.

After stumbling around this problem and having some discussions about it with people I started realizing that a lot of the form validation solutions, in fact, are either overly generic or then overengineered. By keeping the approach to this problem simple, we can write some fairly generic functionality that can be re-used, and even made into a validation-specific [DSL][].

Having said that, what can we do to fix this problem?

## Dependencies and imports

Because this article talks about Calmm, there are a couple of libraries we'll be using.

 - [`karet`][karet] – React with Kefir bindings to allow embedding observables into VDOM—replaces the regular React import in React components
 - [`karet.util`][karet.util] – collection of really useful utility functions for handling observables
 - [`partial.lenses`][partial.lenses] – an efficient and highly composable library for viewing and modifying data
 - [`ramda`][ramda] – the functional utility belt choice, plays well with `karet` and `partial.lenses`

For all of the code examples, assume that the following are always imported:

```js
import * as React from 'karet';
import * as U from 'karet.util';
import * as R from 'ramda';
import * as L from 'partial.lenses';
```

## State of the Form

We'll start by creating a simple form with a couple of text fields, that we can use as a base for expanding and extending our functionality.

Let's handle form validation as data that's dependent on some state—it's data that's derived from state that the user wishes to edit, and that's checked on a set of predefined rules.

The way form data is stored is inside an observable `Atom`:

{% highlight javascript %}
const formState = U.atom({
  inputName: { value: '' },
  anotherInput: { value: '' }
});
{% endhighlight %}

We'll get to it later in more detail, but we're storing fields as objects instead of their value directly, as to accommodate other fields than just text input fields. Like radio button groups and dropdowns.

Next, let's write a simple form for this and bind the fields to the state:

{% highlight js %}
const Form = ({ state }) => {
  const inputNameValue = U.view(['inputName', 'value'], state);
  const anotherInputValue = U.view(['anotherInpu', 'value'], state);

  return (
    <form>
      <input type="text"
             name="inputName"
             {...U.bind({ value: inputNameValue })} />

      <input type="text"
             name="anotherInput"
             {...U.bind({ value: anotherInputValue })} />
    </form>
  );
};
{% endhighlight %}

Great! Now if we `formState.log()`, we can see that the state is updated with whatever we're typing.

## Where's my validation data?

So where do we put the information on the form state's validity? As tempting as it would be as a first thought, would be to add an `isValid` property to each field along with its value, let's not do that. We'll _keep state and any information of the form's validity separate from state_. And because atoms are Kefir observables, we can use all of Kefir's methods on atoms.

The advantage of not storing information about the state's validity in the state itself is to keep unnecessary stuff out of the state itself—and storing validity information about the state in itself is a little bit paradoxical in its own way—is worth asking the question: _does this really belong in the state?_ Because validation is derived from state, it's data we don't need to store in the state—the validation results will be updated whenever the state it's derived from is updated.

Now, let's look into lenses, what they're good for and how we can use them to create the results of the validation.

## Gaze into the data

If you're not familiar with partial lenses and optics, I strongly recommend you to check out [polytypic](https://github.com/polytypic)'s superb [`partial.lenses`][partial.lenses]. We'll use it a lot here. Let's start by writing down a schema in pseudocode (for illustrative purposes) for what we'd like the validation results to look like.

```js
schema = {
  inputName: {
    required,
    mustContainAbc
  },
  anotherInput: {
    required,
    mustEqualFoo
  }
}
```

We've specified both fields as `required`, and some field-specific validation conditions; `mustContainAbc` and `mustEqualFoo`. Generally speaking, we'd like our validation results to match the following TypeScript signature.

```ts
type ValidationResult = {
  [fieldName?: string]: {
    [validatorResultName?: string]: any
  }
};
```

We can work with some lens magic here through some optics. Let's work on some lens magic here by utilising [`L.pick`][L.pick]'s templating features and [`L.when`][L.when] for validating the user input.

`L.when` is a simple one; it's given a predicate function and if it returns `true`, its view will be whatever data passes that predicate, otherwise its view will be `undefined`. For example:

```js
L.collect([L.elems, L.when(x => x < 3)],
          [1, 2, 3, 4, 5, 6]);
          // => [1, 2]
```

`L.pick` allows us to pick and choose from a structure whatever we're interested in and return a simpler structure for us to operate on.

```js
const data = {
  foo: 6,
  bar: {
    baz: 42
  },
  ohai: [1, 2, 3]
};

L.get(L.pick({
        foo: 'foo',
        bar: ['bar', 'baz']
      }), data);
      // => { foo: 6, bar: 42 }

L.get(L.pick({
        foo: ['foo', L.when(x => x === 3)],
        x: [L.when(x => x.ohai != null), 'bar', 'baz']
      }), data);
      // => { x: 42 }
```

With just these two lenses, we can create a lens that validates our (still simple) form data:

```js
const validationLens = L.pick({
  inputName: ['inputName', 'value', L.pick({
    required: L.when(x => !x),
    mustContainAbc: L.when(x => !/abc/.test(x))
  })],
  anotherInput: ['anotherInput', 'value', L.pick({
    required: L.when(x => !x),
    mustEqualFoo: L.when(x => x != 'foo')
  })]
});
```

We've created a lens template that will result in an object the same shape as we did the sketch of before it. [`L.pick`][L.pick] takes an object, and returns an object with a view for each key through the lens we've given it as value. So here we're essentially saying something like this:

<blockquote class="blockquote" markdown="1">
  Give me an object, where the key `inputName` is a view of the path `inputName.value`, which should contain an object with the key `required` if `inputName.value` is falsy.
</blockquote>

Notice something strange? We're essentially taking the logical complement—that is turning `true` into `!true`—of a function that returns `true` for valid input in the case of `L.when`. This means `L.when` will return undefined for valid input, and a focus for the lens otherwise.

Before we put this into the test, let's rewrite it into using some handy ramda functions.

```js
const validationLens = L.pick({
  inputName: ['inputName', 'value', L.pick({
    required: L.when(R.complement(R.identity)),
    mustContainAbc: L.when(R.complement(R.test(/abc/))
  })],
  anotherInput: ['anotherInput', 'value', L.pick({
    required: L.when(R.complement(R.identity)),
    mustEqualFoo: L.when(R.complement(R.equals('foo')))
  })]
});
```

> *Note that we will explore creating our own DSL for form validation in a later post*

Now, let's put the validation into action:

```js
const validationResult = formState.map(L.get(validationLens));
```

That's all that's needed to work the magic.

## A closer look

Let's consider that the form's state is perfectly valid and all conditions are met, the validation result in the form above would be:

```js
{
  inputName: {
    required: undefined,
    mustContainAbc: undefined
  },
  anotherInput: {
    required: undefined,
    mustEqualFoo: undefined
  }
}
```

If [`L.when`](https://github.com/calmm-js/partial.lenses#L-when) returns false, it will result in `undefined`, otherwise it will return a view of the data that passes the given predicate—for example `{ inputName: 'fooabc' }` returns `undefined` for its validation, but `{ inputName: 'foobar' }` returns `{ mustContainAbc: 'foobar' }` for the validation in `inputName`.

But due to how partial lenses work, if something is `undefined`, it's equal to it not existing in the first place. So there's no use in keeping it around—so that objects' keys that have a value of `undefined` will be removed. This leaves us with

```js
{
  inputName: undefined,
  anotherInput: undefined
}
```

But wait—didn't I just write that keys with `undefined` get removed? I did, and because of this, the object is reduced further, because all of its values are `undefined`. The result will simply be

```js
undefined
```

Due to how partial lenses work, unless we've specified invariants in our structure to exist in case of removal, all empty values such as `[]`, `{}` and `undefined` will be removed.

## Form-level validation

Due to how the validation works, checking if the form in itself is valid is trivial:

```js
const formIsValid = validationResult.map(x => typeof x !== 'undefined');
```

or or more concisely in Ramda terms

```js
const formIsValid = validationResult.map(R.isNil);
```

Now, let's add a submit button to the form.

```jsx
const SubmitButton = ({ disabled }) =>
  <button disabled={disabled}
          onClick={e => whateverFormSubmit()}>
    Submit
  </button>;
```

## Field-level validation

[DSL]: https://en.wikipedia.org/wiki/Domain-specific_language

[Siili]: https://www.siili.com

[calmm]: https://github.com/calmm-js
[ramda]: https://ramdajs.com
[karet]: https://github.com/calmm-js/karet
[karet.util]: https://github.com/calmm-js/karet.util
[partial.lenses]: https://github.com/calmm-js/partial.lenses
[L.pick]: https://github.com/calmm-js/partial.lenses#L-pick
[L.when]: https://github.com/calmm-js/partial.lenses#L-when
