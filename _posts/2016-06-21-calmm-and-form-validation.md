---
layout: post
title:    "Validating form data in Calmm.js"
subtitle: "Or: the age-old problem of how do I mark this stuff as invalid???"
date:     2017-06-21 14:26
categories: javascript react frp calmm streams
---

<blockquote class="blockquote">
  <p>
    "I have a form with fields. I need to check if the input is valid based on some arbitrary set of rules before letting the user submit it. What library should I use?"
  </p>
  <footer class="blockquote-footer">
    Most likely every web developer ever
  </footer>
</blockquote>

The problem with form validation is not that it's an inherently difficult thing to do. On the contrary: it's a somewhat trivial problem—in most cases—that is usually solved by approaching the problem from the wrong angle. Add a multitude of different form libraries on top of that, that all promise a simple fix to this problem, but _only if_ the things are done in their way.

This approach is not the silver bullet you're looking for; the other reason why form validation continues to be a pain in the ass is that very rarely are any forms from project to project the same.

I've had the luck of working more or less full-time using the [Calmm stack](https://github.com/calmm-js) since the beginning of March. It's been a continuous learning process of learning and relearning—and rewriting _tons_ of code. But after the initial stumbles I think that for a lot of validation cases _form validation_ continues to be a problem where the solutions are overengineered.

With that said—what can be done to fix this problem—and maybe even be of some help on the way?

For all of the code examples, assume the following are always imported and in scope:

{% highlight js %}
import * as React from 'karet';
import * as U from 'karet.util';
import * as R from 'ramda';
import * as L from 'partial.lenses';
{% endhighlight %}

I'll keep the form simple for now to get this "out there"—and something to expand on in the future.

## State of the Form

Let's handle form validation as data that's dependent on some state—it's data that's derived from state that the user wishes to edit, and that's checked on a set of predefined rules.

The way form data is stored is inside an observable `Atom`:

{% highlight jsx %}
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

The advantage of not storing information about the state's validity in the state itself is to keep unnecessary stuff out of the state itself—and storing validity information about the state in itself is a little bit paradoxical in its own way—is worth asking the question: _does this really belong in the state?_

As a good rule of thumb is to store only independent data in the state—like our form data; keep dependent data out of the state—like form validation. Because validation depends on state data, it can be derived from it. And this is where lenses come into play.

## Gaze into the data

If you're not familiar with partial lenses, I urge you to bookmark [polytypic](https://github.com/polytypic)'s superb lens library [`partial.lenses`][partial.lenses]. We'll use it a lot here.

Let's first sketch down just what kind of validation schema we would like:

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

We've specified both fields as `required`, and some field-specific validation conditions; `mustContainAbc` and `mustEqualFoo`. Let's work on some lens magic here by utilising [`L.pick`][L.pick]'s templating and [`L.when`][L.when] for validation.

```js
const validationL = L.pick({
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

Notice something strange? The conditions in [`L.when`][L.when] are inverse. We're only interested in _invalid data_. This means that the lens will only return data that _matches stuff we don't want_.

Now, let's put the validation into action:

```js
const validationResult = formState.map(L.get(validationL));
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

[partial.lenses]: https://github.com/calmm-js/partial.lenses
[L.pick]: https://github.com/calmm-js/partial.lenses#L-pick
[L.when]: https://github.com/calmm-js/partial.lenses#L-when
