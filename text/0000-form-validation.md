# RFC 0: Add Form Validation Utilities to gca-react-components

## Summary
On the RLC FE team we have decided to standardize on [React Final Form](https://github.com/final-form/react-final-form) (RFF) for our front-end forms. React Final Form has a callback hook for integrating with form validation logic. I have implemented a generalized way of creating a form validation callback function which should be utilized by any of our front end projects that use React Final Form. In addition I've created a few basic field validators which can be integrated with the form validation callback which are of general utility, and should also be shared.

## Motivation
I recently created a generalized form validation callback factory function for creating validation callbacks to be used with React Final Form. As it is a generalized solution it could be used with any of our front-end projects utilizing RFF. This would speed development of form based projects, and they would benefit from using a tested common solution.

The RFF documentation provides a [reference example](https://codesandbox.io/s/yk1zx56y5j) of performing synchronous form validation:

```
    <Form
      onSubmit={onSubmit}
      validate={values => {
        const errors = {};
        if (!values.firstName) {
          errors.firstName = "Required";
        }
        if (!values.lastName) {
          errors.lastName = "Required";
        }
        if (!values.age) {
          errors.age = "Required";
        } else if (isNaN(values.age)) {
          errors.age = "Must be a number";
        } else if (values.age < 18) {
          errors.age = "No kids allowed";
        }
        return errors;
      }}
      ...
    />
```

A hand-tooled solution such as the above is somewhat laborious to construct, and as depicted does not provide for any re-use — valiation functions need to be hand-rolled per form.

## Detailed Design
### Background
React Final Form provides a [`validate`](https://github.com/final-form/final-form#validate-values-object--object--promiseobject) callback property on form instances:

```
validate?: (values: Object) => Object | Promise<Object>
```

> A whole-record validation function that takes all the values of the form and returns any validation errors. There are three possible ways to write a validate function:

>* Synchronously: returns {} or undefined when the values are valid, or an Object of validation errors when the values are invalid.
>* Asynchronously with a Promise: returns a Promise<?Object> that resolves with no value on success or resolves with an Object of validation errors on failure. The reason it resolves with errors is to leave rejection for when there is a server or communications error.
>
>Validation errors must be in the same shape as the values of the form.

This solution currently only addresses the synchronous case.

To paraphrase the above, when RFF wants a validation result it will call the `validate` callback with an object map where the keys are the field names, and the values are the current field values:

```
{
	firstName: 'Joe',
	lastName: 'Bloe',
	email: 'joe.bloe@mailinator',
	phone: '1234567890'
}
```

If all the values are valid, an empty object `{}` or `undefined` should be returned. If there are any invalid values, an object map should be returned with defined values for only the invalid fields:

```
{
	email: 'Invalid email'
}
```

As shown here the defined value can be used to provide a specific error message for that field.

### Solution Overview
There are two components to the solution:

1. **Field level validation functions** — these are provided a value, and return `undefined` if valid, or a string value if invalid.
2. **Form validator factory function** — this is provided a map of fields to validation functions, and returns a validate callback function to provide to RFF.

### 1) Field Validation Functions
Field validation functions will check a provided value to determine the validity of the value. If the value is valid the function returns `undefined`. If the value is invalid the function returns a non-empty string — for our purposes this string is expected to succinctly describe the error (in fact I think it makes sense for it to be a key to a smartling provided translation). Here are some examples (written in TypeScript):

#### Required Field Validation
Perhaps the simplest example of a straight-up validation function:

```
/**
 * Required field validation. Value must not be `undefined` to pass.
 */
export const validateRequired: Validator = (value: any): Error => (
  value !== undefined ? undefined : 'validate.required'
);
```

#### Validator Factory Functions — Regular Expression Validator
A lot of validators perform a certain kind of validation, but require a configuration parameter in order to evaluate a value. Min Chars, Max Chars, and RegEx are examples of this. A Validator Factory Function accepts this test parameter and returns a validator function to use with that parameter. For instance, here's a factory function for creating RegEx based validators:

```
/**
 * RegEx validator factory function
 *
 * @param {regex} a regular expression to test a string against
 * @param {string} errorMessage an error message key
 */
export function createRegExValidator(regex: RegExp, errorMessage: string): StringValidator {
  return (value: string | undefined): Error => (!value || !regex.test(value))
    ? errorMessage
    : undefined;
}
```

Then an email validator would be a certain kind of RegEx validator:

```
//
// regex source: http://emailregex.com/
// based on: RFC 5322
//
const emailRegEx = /^(([^<>()[\]\\.,;:\s@"]+(\.[^<>()[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;

/**
 * Email validation. Must not be falsy, and must pass RegEx pattern.
 */
export const validateEmail: StringValidator = createRegExValidator(emailRegEx, 'validate.email');
```

#### Composing Field Validators
It's not uncommon that a field has a number of validation rules that need to be applied — for instance, _required_, _greater than one character_, _less than twenty-one characters_. A custom validation function can be written to perform a specific pattern of tests, but a general purpose validator composition factory allows more atomic validators to be re-used more easily and predictably:

```
/**
 * Create a composed validator which will execute all passed in validators for a provided value. Will return the first
 * error encountered, or `undefined` if none of the validators evaluate to invalid.
 */
export const composeValidators = (...validators: Validator[]): Validator => (value: any): Error =>
  validators.reduce((error: Error, validator: Validator) => error || validator(value), undefined);

```

Then to create the compound/composed validator outlined above:

```
composeValidators(validateRequired, createMinCharsValidator(1), createMaxCharsValidator(21))
```

### 2) Form validator factory function
This is a factory function that when provided with a map of form fields to validator functions return a RFF `validate` callback function (recall that a RFF `validate` callback consumes a map of fields and values, and returns a map of fields and errors).

For instance:

```
this.formValidator = createFormValidator({
  firstName: validateRequired,
  lastName: validateRequired,
  title: validateRequired,
  email: composeValidators(validateRequired, validateEmail),
  phone: composeValidators(validateRequired, createMinCharsValidator(1), createMaxCharsValidator(21)),
});
```
...

```
<Form
  onSubmit={this.handleDispatchSubmit}
  validate={this.formValidator}
  validateOnBlur={false}
  initialValues={formData}
>
```

And the implementation of `createFormValidator()`:

```
/**
 * Form validator factory function.
 * @param {ValidatorsMap} validators a map of `Validator`s whose keys should correspond to named form fields
 * @return {FormValidator} a function called with a map of field values which returns a corresponding map of errors if
 *  there are any, or any empty object if there are no errors
 */
export function createFormValidator(validators: ValidatorsMap): FormValidator {
  return (values: ValueMap): StringMap => Object.keys(validators)
    .reduce((errors: StringMap, valueKey: string) => {
        const error: Error = validators[valueKey](values[valueKey]);
        if (error) {
          errors[valueKey] = error;
        }
        return errors;
      },
      { },
    );
}
```

## Drawbacks/Consequences

* This solution only considers synchronous form validation — I haven't thought about issues with extending or adapting this solution for asynchronous form validation.
* As a consequence of the implementation of `composeValidators()`, only the first encountered error will be returned from the composed validators. The implementation could be modified if some other result is desired (return an array of all errors, or provide a generic catch-all error to the factory).
* This solution is targeted at React Final Form (which is derived from Final Form). It's unknown if it would be generally applicable to other JS/React form libraries.
* Field validators are expected to return a truthy value, such as a non-empty string. It's most generally useful for the string to describe the error. To integrate with our smartling translations I propose the returned string be a smartling key, but other solutions are possible.

## Alternatives
`validate` callback functions can be hand-rolled on a per usage basis. Or a hybrid solution could draw on a common set of field validators and create a custom `validate` callback using those.

## Unresolved Questions
* Reference implementation is in TypeScript, but for general usage would probably need to be stripped down to standard JS with a type definition file.
* Would this live under _utils_ in gca-react-components?

## Status
Proposed