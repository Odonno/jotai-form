# jotai-form

👻📃

## Tweets

- [Initial announcement](https://twitter.com/dai_shi/status/1518562466627821570)
- [Example with Joi](https://twitter.com/dai_shi/status/1518823782127124480)

## Install

The library can be simply installed by adding `jotai-form` to your dependencies

```sh
npm i jotai-form
# or
yarn add jotai-form
# or
pnpm add jotai-form
```

> **Note**: The library depends on `jotai` so please make sure you have jotai
> installed, if not you can do so by using the below commands
>
> ```sh
> npm i jotai
> ```

## Usage

- [Basic Usage](#basic-usage)
- [Form Level Validation](#form-level-validation)

#### Basic Usage

The very basic form of validation and the most common use case will involve
using `atomWithValidate`, which creates a similar atom to
[jotai](https://jotai.org)

This `atom` can be used with the `useAtom` helper but with a slight change. The
change is that instead of directly giving you the state value you now have form
based properties

```js
const form = {
  value, // the current value of the state
  isDirty, // boolean value based on if the value has changed from the initial value
  isValid, // boolean value representing if the current atom state is valid
  error, // if an error was thrown from the validator, that'll be available here

  // and a conditional
  // boolean value when working with async validation, this
  // will be `true` for the duration of the validation and `false` once the validation is complete
  isValidating,
};
```

> **Note**: `isValidating` will not be a property if you are using a synchronous
> validation function, your IDE should already help you with this but use this
> property only when using asynchronous validation functions.

An example, combining all that information. You can find more such examples in
the [examples](/examples/) folder

```js
import { useAtom } from 'jotai';
import { atomWithValidate } from 'jotai-form';
import * as Yup from 'yup';

// defining a validation schema for the atom
const emailSchema = Yup.string().email().required();

// creating the atom with an async validation function
const emailAtom = atomWithValidate('', {
  validate: async (v) => {
    await emailSchema.validate(v);
    return v;
  },
});

const Form = () => {
  // Same API as other jotai atom's with the difference being
  // addition of form properties on `email`
  const [email, setEmail] = useAtom(emailAtom);

  return (
    <div>
      <input value={email.value} onChange={(e) => setEmail(e.target.value)} />
      <span>{email.isValid && 'Valid'}</span>
      <span>{!email.isValid && `${email.error}`}</span>
    </div>
  );
};
```

#### Form Level Validation

You might have cases where you would like to validate the combination of
multiple values.

Example, Maybe the combined name length of first and last names cannot exceed 15
characters.

For this you can use `validateAtoms` which takes in a atom group and a validator
function which works very similar to the `atomWithValidate`'s validate function.

The atom group validation atom is a read only atom and with similar properties
as the atom returned from `atomWithValidate`. The only change being `values`
instead of `value`. This contains the passed atom group's values

> **Note**: In most cases the `atomWithValidate` should suffice, form validation
> comes into the picture when dealing with fields that do need a final
> validation, the example here is one of the many cases but do not abuse form
> level validation.

> **Note**: `validateAtoms` will not trigger each atom's internal validate
> functions, while adding disabled states or any other case where you need to
> check the atom's validity, please use the atom's own `isValid` and `error`
> properties.

```js
import { useAtom } from 'jotai';
import { atomWithValidate, validateAtoms } from 'jotai-form';
import * as Yup from 'yup';

const nameSchema = Yup.string().required();

const firstNameAtom = atomWithValidate('', {
  validate: async (v) => {
    await nameSchema.validate(v);
    return v;
  },
});

const lastNameAtom = atomWithValidate('', {
  validate: async (v) => {
    await nameSchema.validate(v);
    return v;
  },
});

const formGroup = validateAtoms(
  {
    firstName: firstNameAtom,
    lastName: lastNameAtom,
  },
  (values) => {
    if (values.firstName.length + values.lastName.length > 15) {
      throw new Error("Overall name can't be longer than 15 characters");
    }
  },
);

const Form = () => {
  const [firstName, setFirstName] = useAtom(firstNameAtom);
  const [lastName, setLastName] = useAtom(lastNameAtom);
  const [formState] = useAtom(formGroup);

  return (
    <div>
      <label>
        firstName
        <input
          value={firstName.value}
          onChange={(e) => setFirstName(e.target.value)}
        />
        <span>{firstName.isValid && 'Valid'}</span>
        <span>{!firstName.isValid && `${email.error}`}</span>
      </label>

      <label>
        lastName
        <input
          value={lastName.value}
          onChange={(e) => setLastName(e.target.value)}
        />
        <span>{lastName.isValid && 'Valid'}</span>
        <span>{!lastName.isValid && `${email.error}`}</span>
      </label>

      <button
        disabled={!formState.isValid || !firstName.isValid || !lastName.isValid}
      >
        Submit
      </button>
      {
        // or
        // <button disabled={!(formState.isValid && firstName.isValid && lastName.isValid)}>Submit</button>
      }
      <span>{!formState.isValid && `${formState.error}`}</span>
    </div>
  );
};
```

## API

#### `atomWithValidate`

```js
atomWithValidate(initialValue, options);
```

| param              | description                                                             | default     |
| ------------------ | ----------------------------------------------------------------------- | ----------- |
| `initialValue`     | The initial value of the atom                                           | `undefined` |
| `options.validate` | function returning value or `throw`s an error                           | `undefined` |
|                    | receives the current value of the atom. [more...](#validator-functions) |             |
| `options.areEqual` | function to compare the initial value to the changed values             | `Object.is` |

#### `validateAtoms`

```js
validateAtoms(atomGroup, validator);
```

| param       | description                                                                         | default     |
| ----------- | ----------------------------------------------------------------------------------- | ----------- |
| `atomGroup` | an object of `atomsWithValidate` atoms                                              | `undefined` |
|             | the keys of the atomGroup are used as names/labels                                  |             |
|             | when passed to the `validator` function                                             |             |
| `validator` | function returning void or `throw`s an error                                        | `undefined` |
|             | receives the atomGroup's values with the same keys. [more...](#validator-functions) |             |

### Validator Functions

While the API for both the validator functions, `options.validate` or
`validator` are very similar there's a few differences and also let's talk about
what they expect to work as a form validator.

**Rules**

Things that apply to both the validator functions

1. If the function executes without any error, the validation was successful.
2. If there's any error , even something raised from an inner function, it'll
   make the form invalid.
3. Can pass in a `async` / sync function for validation.
4. The function will be passed the value of the atom that they are supposed to
   validate. (`options.validate` will get the value for it's own atom and
   `validator` will get all the values from the passed in `atomGroup`)

**Differences**

1. `options.validate` expects you to return the value at the end of the
   validation to make sure the validation was run for the same value and your
   state and validated state doesn't have any mismatch.

2. `validator` doesn't expect a return value since it acts as a listener to the
   group of atoms passed in and the values will not need the same state check

> **Note**: The keys of the object defined for the atomGroup is used as the keys
> for the `values` that are passed to `validator`
>
> ```js
> const atomGroup = { name: nameAtom, age: ageAtom };
> const validator = (values) => {
>   values.name; // nameAtom's value
>   values.age; // ageAtom's value
> };
> validateAtoms(atomGroup, validator);
> ```
