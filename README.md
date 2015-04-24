Telephonist
===========

Telephonist makes it easy to design state machines for [Twilio][twilio] calls. 
These state machines bring TwiML and logic together in one place, making call 
flows easier to maintain.

## Installation

Get it from Github by adding it to your `deps` in `mix.exs`: 

```elixir
def deps do
  [{:telephonist, github: "danielberkompas/telephonist"}]
end
```

Run `mix deps.get` to install the package.  Then, add `:telephonist` to your 
applications list. For example:

```elixir
def application do
  [mod: {YourApp, []},
   applications: [:logger, :telephonist]]
end
```

This will ensure that all Telephonist's processes are started and supervised
properly.

## Usage


### Basic Concepts

Like most state machines, Telephonist state machines are based on two concepts: 
state and transitions.

#### State

A state is represented by the `Telephonist.State` struct.

```elixir
%Telephonist.State{
  machine: MachineName,
  name: :state_name,
  options: [],
  twiml: "<?xml ..."
}
```

States are primarily used to define what TwiML should be displayed to Twilio for
a given call at a particular time. Telephonist provides a simple macro to make 
generating and returning `Telephonist.State` structs easy:

```elixir
defmodule CustomStateMachine do
  use Telephonist.StateMachine, initial_state: :introduction

  state :introduction, _twilio, _options do
    say "Welcome to my phone tree!"
  end
end
```

The `state/3` macro is just sugar, and defines a function like this:

```elixir
def state(:introduction, _twilio, options) do
  xml = twiml do
    say "Welcome to my phone tree!"
  end

  %Telephonist.State{
    machine: __MODULE__,
    name: :introduction,
    options: options,
    twiml: xml
  }
end
```

The three arguments are as follows:

- `state_name`: the name of the state, obviously.
- `twilio`: a map of parameters sent in from Twilio.
- `options`: a map of custom options defined by you at various points during the
  call's lifecycle.

Whenever Telephonist wants to get a particular state out of your module, it will
call the `state/3` function generated by the `state/3` macro, like so:

```elixir
# twilio  -> a map of parameters that came from Twilio
# options -> any custom options that are appended to the call over time
CustomStateMachine.state(:introduction, twilio, options)
```

You can pattern match with the `state/3` struct just like a function definition.

```elixir
state :introduction, _twilio, %{error: msg} do
  say "An error occurred! #{msg}"
end

state :introduction, _twilio, _options do
  say "Welcome to my phone tree!"
end
```

### Transitions

Transitions are handled through the `transition/3` function. It takes the same
three arguments as the `state/3` function or macro.

- `state_name`: the name of the state that is being transitioned from.
- `twilio`: a map of parameters passed in from Twilio.
- `options`: a map of custom parameters defined by you.

You can define it on your state machines like so:

```elixir
defmodule CustomCallFlow do
  use Telephonist.StateMachine, initial_state: :choose_language

  state :choose_language, twilio, options do
    say "#{options[:error]}" # say any error, if present
    gather timeout: 10 do
      say "For English, press 1"
      say "Para español, presione 2"
    end
  end

  state :english, twilio, options do
    say "Proceeding in English..."
  end

  state :spanish, twilio, options do
    say "Procediendo en español..."
  end

  # If the user pressed "1" on their keypad, transition to English state
  def transition(:choose_language, %{Digits: "1"} = twilio, options) do
    state :english, twilio, options
  end

  # If the user pressed "2" on their keypad, transition to Spanish state
  def transition(:choose_language, %{Digits: "2"} = twilio, options) do
    state :spanish, twilio, options
  end

  # If neither of the above are true, append an error to the options and
  # remain on the current state
  def transition(:choose_language, twilio, options) do
    options = Map.put(options, :error, "You pressed an invalid digit. Please try again.")
    state :choose_language, twilio, options
  end
end
```

Note that `transition/3` must return a `Telephonist.State`. This is easily done
by simply calling the `state/3` function. Also, note that you can easily switch
to _another_ state machine by simply calling `state` on it:

```elixir
def transition(:choose_language, %{Digits: "1"} = twilio, options) do
  EnglishCallFlow.state(:introduction, twilio, options)
end

def transition(:choose_language, %{Digits: "2"} = twilio, options) do
  SpanishCallFlow.state(:introduction, twilio, options)
end
```

Control of the call will then be passed to the other state machine. This allows
you to keep your state machines small, focused, and potentially reusable.

#### on_complete/3

When a call completes, Telephonist will call the `on_complete/3` callback. It
will receive the `Telephonist.State` of the call at the time it completed,
Twilio's final request parameters, and the custom options the call accumulated
during its life:

```elixir
def on_complete({sid, twilio_call_status, state}, twilio, options) do
  :ok
end
```

This is a good place to put any cleanup logic that you need to perform after a
call completes.

#### on_transition_error/4

This callback will be run if a transition fails due to an exception. This will
most often occur when you fail to define a transition or state, or if your
pattern matching left a case out. It provides you an opportunity to recover the
call and prevent the user from hearing a Twilio error message.

```elixir
def on_transition_error(exception, state_name, twilio, options) do
  # To prevent an error, return a new state:
  state :recover, twilio, options
end
```

The default implementation of `on_transition_error/4` that comes with
`Telephonist.StateMachine` will simply re-raise the error.

### Processing Calls

Once you've defined a state machine, you can process calls through it using
`Telephonist.CallProcessor`.

```elixir
# The web framework shown here is pseudo-code
def index(conn, twilio) do
  options = %{} # Whatever I want to be able to use in my states and transitions
  state = Telephonist.CallProcessor.process(MyStateMachine, twilio, options)
  render conn, xml: state.twiml
end
```

That's it! New calls will start off in `MyStateMachine.initial_state` and
progress from there. Existing calls will be looked up in an ETS table managed by
`Telephonist.OngoingCall` and will progress from where they left off.

#### Under the Hood

- `CallProcessor.process/3` spins off a new process to do the actual call
  processing for maximum concurrency.
- The current state of all ongoing calls is stored in the ETS table managed by
  the `Telephonist.OngoingCall` process. See its docs for more details.

### Subscribing to Events

Telephonist publishes events via `GenEvent`. In fact, `Telephonist.Logger` is
simply a subscriber to these events. Look there for an example of how to
implement your own subscriber.

## Other Twilio Libraries

See these other Elixir libraries I've written for Elixir:

- [ExTwilio][ex_twilio]. A Twilio API client.
- [ExTwiml][ex_twiml]. Render TwiML from Elixir. This is actually a dependency
  of Telephonist, and is used in the `state/3` macro.

## LICENSE
Telephonist is under the MIT license. See the [LICENSE](/LICENSE.md) file for 
more details.


[ex_twilio]: https://github.com/danielberkompas/ex_twilio
[ex_twiml]: https://github.com/danielberkompas/ex_twiml
[twilio]: http://twilio.com
