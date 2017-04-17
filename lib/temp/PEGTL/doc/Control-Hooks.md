# Control Hooks

Beyond the top-level grammar rule, which *has* to be supplied to a parsing run, and an action class template, which *can* be supplied to a parsing run, a third customisation point within the PEGTL allows the user to provide a *control* class (template) to a parsing run.

The control class must implement a set of static hook functions that are called by the PEGTL in order to

1. customise which exceptions are thrown in case of errors, and

2. add custom debug logging to follow the details of a parsing run.

## Contents

* [Normal Control](#normal-control)
* [Debug Functions](#debug-functions)
* [Exception Throwing](#exception-throwing)
* [Debugging and Tracing](#debugging-and-tracing)
* [Advanced Control](#advanced-control)
* [Changing Control](#changing-control)

## Normal Control

The `normal` control class template included with the PEGTL is used by default and shows which hook functions there are.

```c++
template< typename Rule >
struct normal
{
   template< typename Input,
             typename... States >
   static void start( const Input&, States&&... )
   {
   }

   template< typename Input,
             typename... States >
   static void success( const Input&, States&&... )
   {
   }

   template< typename Input,
             typename... States >
   static void failure( const Input&, States&&... )
   {
   }

   template< typename Input,
             typename... States >
   static void raise( const Input& in, States&&... )
   {
      throw parse_error( "parse error matching " +
                         internal::demangle< Rule >(), in );
   }

   template< template< typename... > class Action,
             typename... States >
   static void apply0( States&&... st )
   {
      Action< Rule >::apply0( st... );
   }

   template< typename Input,
             template< typename... > class Action,
             typename... States >
   static void apply( const count_data& begin, const count_data& end, const char* source, States&&... st )
   {
      const Input in( begin, end.data, source );
      Action< Rule >::apply( in, st... );
   }

   template< apply_mode A,
             rewind_mode M,
             template< typename ... > class Action,
             template< typename ... > class Control,
             typename Input,
             typename ... States >
   static bool match( Input& in, States&&... st )
   {
      constexpr bool use_control = !internal::skip_control< Rule >::value;
      constexpr bool use_action = use_control && ( A == apply_mode::ACTION ) && ( !is_nothing< Action, Rule >::value );
      constexpr bool use_apply0 = use_action && internal::has_apply0< Action< Rule >, internal::type_list< States... > >::value;
      constexpr dusel_mode mode = static_cast< dusel_mode >( static_cast< char >( use_control ) + static_cast< char >( use_action ) + static_cast< char >( use_apply0 ) );
      return internal::duseltronik< Rule, A, M, Action, Control, mode >::match( in, st... );
   }
};
```

The `start()`, `success()` and `failure()`-functions can be used to debug a grammar by using them to provide insight into what exactly is going on during a parsing run.

The `raise()`-function is used to create a global error, and any replacement should again throw an exception, or abort the application.

The `apply()` and `apply0()`-functions can customise how actions with, and without, receiving the matched input are called, respectively.

The `match`-function wraps the actual matching duseltronik and chooses the correct one based on whether control is enabled for `Rule`, and whether the current action is enabled and has an `apply()` or `apply0()`-function.

## Debug Functions

Before attempting to match a rule `R`, the PEGTL calls `C< R >::start()` where `C` is the current control class template.

Depending on what happens during the attempt to match `R`, one of the other three functions might be called.

- If `R` succeeds, then `C< R >::success()` is called; compared to the call to `C< R >::start()`, the input will have consumed whatever the successful match of `R` consumed.

- If `R` finishes with a failure, i.e. a return value of `false` from its `match()`-function, then `C< R >::failure()` is called; a failed match is not allowed to consume input.

- If `R` is wrapped in `must< R >`, a global failure is generated by calling `C< R >::raise()` to throw some exception as is expected by the PEGTL in this case.

- If a sub-rule of `R` finishes with a global failure, and the exception is not caught by a `try_catch` or similar combinator, then no other function of `C< R >` is called after `C< R >::start()`.

Additionally, if matching `R` was successful, actions are enabled, and `A< R >` is not derived from `tao::pegtl::nothing`, where `A` is the current action class template:

- If `A< R >::apply0()` exists, then `C< R >::apply0()` is called with the current state-arguments.

- Else `C< R >::apply()` is called with the begin and end of the matched input, and the current state arguments.

## Exception Throwing

The `raise()`-control-hook-function *must* throw an exception.
For most parts of the PEGTL the exception class is irrelevant and any user-defined data type can be thrown by a user-defined control hook.

The `try_catch` rule only catches exceptions of type `tao::pegtl::parse_error`!

When custom exception types are used then `try_catch_type` must be used with the custom exception class that they are supposed to catch as first template argument.

## Debugging and Tracing

For debugging a grammar and tracing exactly what happens during a parsing run, the control class methods `start()`, `success()` and `failure()` can be used.
In addition, `apply()` and `apply0()` can be used to see which actions are invoked.

The included class `tao::pegtl::tracer` in `<tao/pegtl/tracer.hpp>` gives a pratical example that can be used as control class to debug grammars.
When an instance of class `tao::pegtl::trace_state` is used as single state in a parsing run with the tracer-control then the debug output contains a line number and rule number as additional information.

## Advanced Control

The control template's `match()`-function is the first, or outside-most, function that is called in the flow that eventually leads to calling a rule's `match()`-function.

For advanced use cases it is possible to create a custom control class with a custom `match()`-function that can change "everything" before calling the actual rule's `match()`-function.

Similarly the control's `apply()` and `apply0()`-functions can customise action invocation; in particular `apply()` can change how the matched portion of the input is presented to the action.

## Changing Control

Just like the action class template, a custom control class template can be used by either

1. supplying it as explicit template argument to the `parse()`-functions, or

2. setting it for a portion of the grammar with the `tao::pegtl::control` combinator.

Copyright (c) 2014-2017 Dr. Colin Hirsch and Daniel Frey