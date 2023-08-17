[TOC]

# Chapter 1: Introduction to domain-driven design

> A few weeks of programming can save you hours of planning.

Create a shared model:

- Find *Business Events* and *Workflows*
- Partition problem domain into smaller sub-domains (well-bounded contexts!)
- Create a model of each sub-domain in the solution (Design for Autonomy!)
- Develop a common language shared between everyone involved in the project

## Workflows, Scenarios, Use Cases

* Scenario: Goal for user;
        Use-case: user interactions to take
* Process: goal for business;
        Workflow: detailed part of process (single-team)

Try to model processes:
+ Events -(triggers)-> Commands -(Input)-> Workflows -(Output)-> [Event, Event, Event]

- We might ask: "What made these domain events happen?"
- Commands are always written in the imperative: "do this for me." If the command was: "Make X happen," then, if the workflow made X happen, the corresponding domain event would be: "X happened."

We can define a "domain" as "an area of coherent knowledge." … a "domain" is just that which a "domain expert" is expert in!

## Create a solution using Bounded Contexts

Need to create a distinction between a "problem space" and a "solution space," and they must be treated as two different things. To build the solution we will create a model of the problem domain, extracting only the aspects of the domain that are relevant and then recreating them in our solution space.

> A boundary that is too big or too vague is no boundary at all. As the saying goes: "good fences make good neighbors."

Always focus on business and customer value rather than any kind of "pure" design.

Some domains _are_ more important than others; the ones that provide a business advantage. Focus on those bounded contexts that add the most value.

Making money (or saving money) is almost always the driver behind a development project.

# Chapter 2: Understanding the Domain

> We want to avoid imposing our own mental model on the domain

Pretend you're an anthropologist:
- Avoid having preconceived notions.
- Keep our minds open during requirements gathering.
- do NOT impose our own technical ideas on the domain.

In domain-driven design, let the _domain_ drive the design!

Discuss context: system designed for beginners will often be quite different from a system designed for experts; Skip other bounded context, just keep track of what _this_ workflow needs from _that_ context

**The output of a workflow should always be the events that it generates, the things that trigger actions in other bounded contexts**


## Workflow (problem-space)

Try to capture the domain in a slightly structured way:

```
Bounded context: Order-Taking
Workflow: "Place order"
   triggered by:
      "Order form received" event (when Quote is not checked)
   primary input:
      An order form
   other input:
      Product catalog
   output events:
      "Order Placed" event
   side-effects:
      An acknowledgment is sent to the customer,
      along with the placed order
```

## Domain Model

If our model isn't complicated, we wouldn't be capturing the requirements properly.

Data structures and constraints:

```
context: Order-Taking
data WidgetCode = string starting with "W" then 4 digits
data GizmoCode = string starting with "G" then 3 digits
data ProductCode = WidgetCode OR GizmoCode

data PricedOrder =
    ValidatedCustomerInfo
    AND ValidatedShippingAddress
    AND ValidatedBillingAddress
    AND list of PricedOrderLine  // different from ValidatedOrderLine
    AND AmountToBill             // new
data PricedOrderLine =
    ValidatedOrderLine
    AND LinePrice                // new
data PlacedOrderAcknowledgment =
    PricedOrder
    AND AcknowledgmentLetter

data ValidatedOrder =
    ValidatedCustomerInfo
    AND ValidatedShippingAddress
    AND ValidatedBillingAddress
    AND list of ValidatedOrderLine
data ValidatedOrderLine =
    ValidatedProductCode
    AND ValidatedOrderQuantity

data UnvalidatedOrder =
    UnvalidatedCustomerInfo
    AND UnvalidatedShippingAddress
    AND UnvalidatedBillingAddress
    AND list of UnvalidatedOrderLine
data UnvalidatedOrderLine =
    UnvalidatedProductCode
    AND UnvalidatedOrderQuantity

data OrderQuantity = UnitQuantity OR KilogramQuantity
data UnitQuantity = integer between 1 and 1000
data KilogramQuantity = decimal between 0.05 and 100.00

data CustomerInfo = ???   // don't know yet
data BillingAddress = ??? // don't know yet
```

## Workflow (solution-space)

```
workflow "Place Order" =
    input: OrderForm
    output:
       OrderPlaced event (put on a pile to send to other teams)
       OR InvalidOrder (put on appropriate pile)
    // step 1
    do ValidateOrder
    If order is invalid then:
        add InvalidOrder to pile
        stop
    // step 2
    do PriceOrder
    // step 3
    do SendAcknowledgementToCustomer
    // step 4
    return OrderPlaced event (if no errors)

substep "ValidateOrder" =
    input: UnvalidatedOrder
    output: ValidatedOrder OR ValidationError
    dependencies: CheckProductCodeExists, CheckAddressExists
    validate the customer name
    check that the shipping and billing address exist
    for each line:
        check product code syntax
        check that product code exists in ProductCatalog
    if everything is OK, then:
        return ValidatedOrder
    else:
        return ValidationError
```

# Chapter 3: Functional Architecture

We really shouldn't be thinking too much about architecture at this point. To translate from _logical design_ to _deployable_ equivalent is not critical: keep the bounded contexts decoupled and autonomous.

> We don't understand the system yet. We are at the peak of our ignorance!

Often need to start implementing parts of the domain prior to understanding the rest of it.

## C4 Model

* The entire system ("System Context")
* Deployable Units ("Containers")
* Major structural building blocks ("Components")
* Low-level functions ("Modules")

**A context is an _autonomous_ subsystem with a _well-defined boundary_**

## Transferring data between bounded contexts

Coupling between contexts:
- Events and commands
- Queues vs Function Calls: the exact mechanism for transmitting events between contexts depends on architecture.

Event DTOs (Data Transfer Objects):

* Serialized: `(Domain Model) -> [transform] -> (DTO type) -> [serialize] -> (JSON/XML)`
* Deserialized: `(JSON/XML) -> [deserialize] -> (DTO type) -> [transform] -> (Domain model)`

## Trust Boundaries

- Input Gates: The incoming DTO will have no such constraints and could contain anything; After validation at the input gate, we are sure the Domain object is valid.
- Output Gates: Deliberately "lose" information, private data doesn't leak from bounded context

Two contexts will need to agree on common format for the communication to be successful.
Anti-corruption layer: acts as a translator between two languages: upstream and downstream contexts.

### `Command Object --> [Workflow] --> List of Event Objects`

Workflow does NOT "publish" domain events, how they get published is a separate concern.

if we need an event listener in the workflow, append to the end of the workflow

> Design Principle: Code that changes together belongs together

YUCK: Needing to change every layer. All dependencies must point inward:

1. [The Onion Architecture](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)
1. [Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/)
1. [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

## Keep I/O at the edges

Want functions that are predictable and easy to reason about: **without looking inside them**

You can't model your workflow around the database if you can't access the DB in your workflow.


# Chapter 4: Understanding Types

What does a functional programmer mean by `type`? The set of possible values (inputs, outputs, function signatures)

A value is a member of a type. Values are immutable.

- **AND** Record / Product: `type fruitSalad = { Apple: AppleVariety, Banana: BananaVariety }`
- **OR** Choice / Union: `type fruitSnack = Apple of AppleVariety | Banana of BananaVariety`
- Compound vs Simple: Multi vs Single-case choices

The type associated with the `Ok` or the type associated with the `Error` state.

```
type Result<'Success, 'Failure> = Ok of 'Success | Error of 'Failure

type PayInvoice =
        UnpaidInvoice -> Payment -> Result<PaidInvoice, PaymentError>
```

Result is documentation of the "error-effect" that may happen.

# Chapter 5: Domain Modeling with Types

> Can we use source code directly and avoid the need for UML diagrams?
>
> The answer is: Yes.

The implementation can never get out of sync with the design because it is represented in the code itself.

It's important that data doesn't get mixed up. Just because they're both represented by `ints` doesn't mean they're interchangeable!

Type aliases: Removes the overhead, but looses type-safety at compile-time.

1. [Data-oriented design](https://en.wikipedia.org/wiki/Data-oriented_design)

## Modeling unknown types

Model them as a placeholder, one that is explicitly `undefined`.

## Value Objects

- _Entities_: Objects with persistent identity.
- _Value Objects_: Without persistent identity.

_Entities_ have a lifecycle, transformed from one state to another throughout business processes. Also, context-dependent whether it is modeled as Entity or Value (e.g. is it unique in this context, or is it one of many)

## Identity

Removing Equality-operators at the object-level removes ambiguity, forces explicit comparison of "what is equality"

Values are immutable, meaning after they're initialized they cannot be changed.

## Aggregates and Consistency

Aggregate is a collection of Entities which acts on the consistency boundary: When one part of the aggregate is updated, the other parts may be updated to ensure consistency.

Its the only component that "knows" how to preserve consistency: At the top level.

A collection of Entities can just be a list; it doesn't have a "root" top-level Entity.

# Chapter 6: Integrity inside the domain

> Create a bounded context that always contains data we can trust, distinct from the untrusted outside world

The more information we can capture in the type system, the less documentation and less of risk in implementation

How do we ensure constraints are enforced? The _Smart Constructor_ approach: Cant be created without a private constructor.

Invariant: a condition that stays true no matter what else happens.

Self-documenting code, removing the need to write unit-tests for a requirement.

> When domain experts talk about "verified" and "unverified" emails, model them as separate things_

What in the design makes that rule explicit? Model it as a choice. Create a new type!
Have two distinct types, with different rules around them.

> Make illegal states unrepresentable

Invalid situations can never exist in our code, and we no longer need to write unit-tests for them.

Types: don't have to worry about someone passing in an UnValidatedEmail by accident (breaking a business rule) from not reading docs.

> And now we can guarantee, without ever writing a unit test, that addresses in a validated order have been validated

Consistency is a business term, not a technical one; it is context-dependent.

## Consistency between contexts

We must keep each bounded context isolated and decoupled.

Businesses generally do not require that every process move in lock-step. There's no need for rigid coordination between the bounded contexts.

Only update the Aggregate once per transaction. If you need to define a new aggregate just for one use case, go ahead.

Constraints can be shared between multiple aggregates. Example: requirement that balance never be below zero could be modeled with a NonNegativeMoney type.

# Chapter 7: Modeling Workflows as Pipelines

> Transformation-oriented programming

Each smaller pipe will do one transformation, then we'll glue the smaller pipes together.

Sharing common structures using generics:

```
type Command<'data> = {
        Data: 'data
        Timestamp: DateTime
        UserId: string
        // etc
}
```

## Modeling as a Set of States

How should we model states? Model the domain by create a new type for each state

Finally, create a top-level type that's a choice between all the states.

## State Machines

A document record can be in one or more states, with paths from one state to another ("transitions") triggered by commands

```
(StateA) -> (StateB)
(StateB) -> (StateA)
(StateB) -> (StateC)
```

From the caller's point of view, the set of states is treated as one thing for general manipulation.
When processing the events internally, each state is treated separately.

When `Result` is used anywhere, it "contaminates" all it touches, and the "result-ness" is passed up until a top-level function handles it.

Put the dependencies _first_ in the parameter order, making partial application easier (functional-equivalent of dependency injection).

> Rather than embedding that logic into the workflow, make it someone else's problem!

Take a function of this type as a dependency

Using the `unit/void/none` type to indicate that there's some side effect. Rather use a simple `Sent/NotSent` choice instead of a bool.

Workflow returns a _list_ of events, a choice type of the events, emit the list of events.

The `Async` effect is contagious for any code containing it.

## Composing the Workflow from the Steps

Wire the output of one step to the input of the next one, building up the overall workflow.

> It won't quite be that simple!

We have to juggle the input and output types so they are compatible and can be fitted together.

Dependencies for the top-level workflow function should _not_ be exposed, because the caller doesn't need to know aobut them.

## Long-Running Workflows

Broken the original workflow into smaller, independent chunks: Mini-workflows.

State machine: before each step, loaded from storage, having been persisted by one of its states.

**Sagas**: Long-running workflows.

Break a workflow into decoupled, stand alone pieces connected by events

> As always, you should do what serves the domain and is most helpful for the task at hand.

**we should be continually mixing requirements gathering with modeling and modeling with prototyping.**
