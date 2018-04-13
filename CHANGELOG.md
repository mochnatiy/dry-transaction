# 0.11.1 / 2018-03-15

## Added

- Include `operation_name:` in options passed to step adapters. This is helpful for 3rd party step adapters that need the operation's name to fetch it from the container again later. (timriley in [#94][pr94])

[Compare v0.11.0...v0.11.1](https://github.com/dry-rb/dry-transaction/compare/v0.11.0...v0.11.1)

[pr94]: https://github.com/dry-rb/dry-transaction/pull/94

# 0.11.0 / 2018-02-19

## Added

- Around steps, which allow control of the execution of subsequently called steps. If you know how middleware works in rack or how around callbacks can be used in RSpec, it's the same. A typical example of usage would be for DB transactions (now first class support!) or controlling side effects: rolling back the changes, cleaning garbage produced by a failed transaction, etc. See a more detailed explanation of how this works [in the PR][pr85] (flash-gordon in [#85][pr85])
- Broadcast when a step has started by sending the event `step_called` (mihairadulescu in [#82][pr82])
- Add new step `check` that returns `Success` or `Failure` base on conditions (semenovDL in [#84][pr84])
- Support for transaction steps without input values (GustavoCaso and timriley in [#69][pr69])

## Changed

- [BREAKING] Steps no longer broadcast events with their step name followed by `_success` or `_failure`. Now, more generic names are used for the broadcast events. Before each step runs, a `step` event is broadcast, with the step name and its arguments. After a step runs, a `step_succeeded` or `step_failed` event is broadcast, also with the step name, the arguments and the return value (GustavoCaso in [#83][pr83])
- [BREAKING] Pub/sub support is now handled using [dry-events][dry-events] instead of wisper. Subscriber objects should now respond to `#on_step`, `#on_step_succeeded`, or `#on_step_failed` to receive broadcast events (GustavoCaso in [#90][pr90])
- [BREAKING] The step adapter API has been changed in order to support around steps, although, the changes are not significant. Previously, an adapter received a `step` and a list of arguments for calling the operation. The list was passed as `*args` then you were needed to call `call_operation` on `step` and provide the list. From now on an adapter gets an `operation`, its `options`, and `args` (_without_ `*`). `operation` is an ordinary callable object so a typical call is as simple as `operation.(*args)`, that's it. If you want to turn your adapter into an around-like one you need to add `&block` parameter to the list of `call` arguments (e.g. `def call(operation, options, args, &block)`). `block` is responsible for calling the subsequent steps thus you can check or transform the return value and make some decisions based on it. Note capturing the block in the list of arguments is mandatory, a simple `yield` won't work, there are reasons, believe us. Check out the sources of [`around.rb`](https://github.com/dry-rb/dry-transaction/blob/master/lib/dry/transaction/step_adapters/around.rb) for reference, it's dead simple (flash-gordon in [#85][pr85])
- Usages of the `Either` monad was updated with `Result` and its constructors. See [the changes](https://github.com/dry-rb/dry-monads/blob/master/CHANGELOG.md#v040-2017-11-11) in `dry-monads` for more details (flash-gordon in [#81][pr81])
- Minimal Ruby version is 2.2 (flash-gordon in [#72][pr72])

## Fixed

- Pass arguments to a step with keyword arguments with default values (flash-gordon in [#74][pr74])
- Steps can be wrapped with private methods (semenovDL in [#70][pr70])
- Improved error message on missing transaction step (Morozzzko in [#79][pr79])

[Compare v0.10.2...v0.11.0](https://github.com/dry-rb/dry-transaction/compare/v0.10.2...v0.11.0)

[dry-events]: http://dry-rb.org/gems/dry-events/
[pr69]: https://github.com/dry-rb/dry-transaction/pull/69
[pr70]: https://github.com/dry-rb/dry-transaction/pull/70
[pr72]: https://github.com/dry-rb/dry-transaction/pull/72
[pr74]: https://github.com/dry-rb/dry-transaction/pull/74
[pr79]: https://github.com/dry-rb/dry-transaction/pull/79
[pr81]: https://github.com/dry-rb/dry-transaction/pull/81
[pr82]: https://github.com/dry-rb/dry-transaction/pull/82
[pr83]: https://github.com/dry-rb/dry-transaction/pull/83
[pr84]: https://github.com/dry-rb/dry-transaction/pull/84
[pr85]: https://github.com/dry-rb/dry-transaction/pull/85
[pr90]: https://github.com/dry-rb/dry-transaction/pull/90

# 0.10.2 / 2017-07-10

## Fixed

- Only resolve an operation object from the container if the container reports that the key is present. This prevents exceptions from being raised when using dry-container and defining a transaction that includes steps both container-based and local method steps (jonolsson in [#64][pr64])

[Compare v0.10.1...v0.10.2](https://github.com/dry-rb/dry-transaction/compare/v0.10.1...v0.10.2)

[pr64]: https://github.com/dry-rb/dry-transaction/pull/64

# 0.10.1 / 2017-06-30

## Fixed

- Preserve step notification listeners when calling a transaction after passing extra step arguments (jcmfernandes in [#65](https://github.com/dry-rb/dry-transaction/pull/65))

[Compare v0.10.0...v0.10.1](https://github.com/dry-rb/dry-transaction/compare/v0.10.0...v0.10.1)

# 0.10.0 / 2017-06-15

This release makes major changes to the dry-transaction API: transactions are now defined within your own class, support instance methods for defining or wrapping steps, and operation containers are now optional.

## Added

- Added `Dry::Transaction::Operation` convenience mixin. This gives easy access to `Success` and `Failure` result builders within your operation objects, and also enables dry-matcher's block-based result matching API for when the operations are called individually. (timriley in [#58](https://github.com/dry-rb/dry-transaction/pull/58))

    ```ruby
    class MyOperation
      include Dry::Transaction::Operation

      def call(input)
        Success(input)
      end
    end

    my_op = MyOperation.new
    my_op.("hello") do |m|
      m.success do |v|
        "Success: #{v}
      end
      m.failure do |v|
        "Failure: #{v}"
      end
    end
    # => "Success: hello"

## Changed

- [BREAKING] Transactions are now defined within your own classes using a mixin & class-level API for step definitions (GustavoCaso & timriley in [#58](https://github.com/dry-rb/dry-transaction/pull/58))

    ```ruby
    class CreateUser
      include Dry::Transaction(container: Container)

      step :process, with: "operations.process"
      step :validate, with: "operations.validate"
      step :persist, with: "operations.persist"
    end

    create_user = CreateUser.new
    create_user.call("name" => "Jane Doe")
    ```

  Instance methods can wrap operations by calling `super`:

    ```ruby
    class CreateUser
      include Dry::Transaction(container: Container)

      step :process, with: "operations.process"
      step :validate, with: "operations.validate"
      step :persist, with: "operations.persist"

      def process(input)
        adjusted_input = do_something_with(input)
        super(adjusted_input)
      end
    end
    ```

  Substitute operations can be injected when initializing objects (helpful for testing):

    ```ruby
    create_user = CreateUser.new(process: substitute_process_operation)
    ```

- Transactions can be defined without an operations container, using instance methods only.

    ```ruby
    class CreateUser
      include Dry::Transaction

      step :process
      step :validate

      def process(input)
        input = do_something_with(input)
        Success(input)
      end

      def validate(input)
        if input[:email].include?("@")
          Success(input)
        else
          Failure(:not_valid)
        end
      end
    end
    ```

- [BREAKING] You can no longer extend existing transactions with `#prepend`, `#append`, `#insert`, or `#remove`. Since transactions will now be instances of your own classes, with their own different behaviors, there’s no predictable way to combine the behaviors of two different classes. If you need the ability to add or remove steps, you can create separate transactions for the different behaviours you need to offer, or build into your own transaction class a way to skip steps based on input or step arguments.
- [BREAKING] Blocks in step definitions are no longer accepted. If you want to wrap a step with some local behavior, wrap it with an instance method (see above).
- [BREAKING] There is no longer an option for configuring the result matcher block API - we now use `Dry::Transaction::ResultMatcher` by default. If you want to provide your own matcher, you can do this by overriding `#call` in your transaction classes and using your own matcher when a block is given.

[Compare v0.9.0...v0.10.0](https://github.com/dry-rb/dry-transaction/compare/v0.9.0...v0.10.0)

# 0.9.0 / 2016-12-19

## Added

- Procs (or any callable objects) can be passed as a step's `with:` option instead of a container identifier string (AMHOL in [#44](https://github.com/dry-rb/dry-transaction/pull/44))

    ```ruby
    Dry.Transaction(container: MyContainer) do
      step :some_step, with: "operations.some_thing"
      step :another, with: -> input {
        # your code here
      }
    end
    ```

- Support for passing blocks to step adapters (am-kantox in [#36](https://github.com/dry-rb/dry-transaction/pull/36))

    ```ruby
    Dry.Transaction(container: MyContainer) do
      my_custom_step :some_step do
        # this code is captured as a block and passed to the step adapter
      end
    end
    ```

## Changed

- Whole step object is passed to `StepFailure` upon failure, which provides more information to custom matchers (mrbongiolo in [#35](https://github.com/dry-rb/dry-transaction/pull/35))
- `#call` argument order for step operations is now `#call(input, *args)`, not `#call(*args, input)` (timriley in [#48](https://github.com/dry-rb/dry-transaction/pull/48))
- `Dry::Transaction::Sequence` renamed to `Dry::Transaction` (timriley in [#49](https://github.com/dry-rb/dry-transaction/pull/49))

[Compare v0.8.0...v0.9.0](https://github.com/dry-rb/dry-transaction/compare/v0.8.0...v0.9.0)

# 0.8.0 / 2016-07-06

## Changed

- Match block API is now provided by `dry-matcher` gem (timriley)
- Matching behaviour is clearer: match cases are run in order, the first match case “wins” and is executed, and all subsequent cases are ignored. This ensures a single, deterministic return value from the match block - the output of the single “winning” match case. (timriley)

## Added

- Provide your own matcher object via a `matcher:` option passed to `Dry.Transaction` (timriley)

[Compare v0.7.0...v0.8.0](https://github.com/dry-rb/dry-transaction/compare/v0.7.0...v0.8.0)

# 0.7.0 / 2016-06-06

## Added

* `try` steps support a `:raise` option, so a caught exception can be re-raised as a different (more domain-specific) exception (mrbongiolo)

## Changed

* Use dry-monads (e.g. `Dry::Monads::Either::Success`) instead of kleisli (`Kleisli::Either::Success`) (flash-gordon)

## Fixed

* Add `#respond_to_missing?` to `StepFailure` wrapper class so it can more faithfully represent the failure object it wraps (flash-gordon)
* Stop the DSL processing from conflicting with ActiveSupport's `Object#try` monkey-patch (joevandyk)

[Compare v0.6.0...v0.7.0](https://github.com/dry-rb/dry-transaction/compare/v0.6.0...v0.7.0)

# 0.6.0 / 2016-04-06

## Added

* Allow custom step adapters to be supplied via a container of adapters being passed as a `step_adapters:` option to `Dry.Transaction` (timriley)
* Raise a meaningful error if a `step` step returns a non-`Either` object (davidpelaez)

## Internal

* Change the step adapter API so more step-related information remains available at the time of the step being called (timriley)

[Compare v0.5.0...v0.6.0](https://github.com/dry-rb/dry-transaction/compare/v0.5.0...v0.6.0)

# 0.5.0 / 2016-03-16

* Renamed gem to `dry-transaction`
* Transactions are now defined via `Dry.Transaction(options)`

# 0.4.0 / 2015-12-26

* Added `Transaction#append`, `#prepend`, `#insert` and `#remove` for modifying existing transactions and returning new copies.
* Removed `raw` step adapter name - `step` should be used instead.

# 0.3.2 / 2015-11-13

* Fixed a bug where additional step arguments were passed in the wrong order for `map` and `raw` steps.

# 0.3.1 / 2015-11-12

* Removed deterministic gem dependency

# 0.3.0 / 2015-11-11

* Use Kleisli’s `Either` monad for wrapping the result value instead of Determinstic’s `Result`.
* Add built-in, expressive result matching via a block passed to `#call`.
* Fixed a bug in which the input object could be modified over multiple calls.

# 0.2.0 / 2015-11-02

* Added support for subscriptions to step success and failure events.
* Stopped catching all exeptions in `try` steps. Instead, require a `catch:` option to be provided with one or more exception classes, e.g. `try :some_step, catch: MyError` or `try :another_step, catch: [MyError, AnotherError]`. This makes the step's failure conditions explicit and ensures that unexpected exceptions bubble up and halt the program as usual.
* Added a `step` alias for the `raw` step adapter. This is a more natural expression for transactions in which most of the steps already return `Result` objects and otherwise need no special handling.
* Removed support for inline procs passed to the `with:` step option. This ensures that every piece of logic in the transaction resides in directly addressable operations in the container, which encourages simplicity in the transaction, and improved testability and reusability across the application as a whole.

# 0.1.0 / 2015-10-28

Initial release.
