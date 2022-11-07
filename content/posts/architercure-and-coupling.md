---
title: "Architecture and coupling"
date: 2022-10-30T12:05:42Z
draft: false
tags: [architecture, design]
---

## Are we thinking about coupling the right way?

Few weeks ago I was reading "Software Architecture: The Hard Parts" by Neal Ford, Mark Richards, Pramod Sadalage, Zhamak Dehghani.

One concept sticked to my mind. It wasn't something completely new for my brain, but the clarity and simplicty of the sentence, made think of how many times I haven't given it enough weight when dealing with architetural deicsions.

"An architect can never reduce semantic coupling via implementation,but they can make it worse"

I'll expand clarify what I mean for coupling, complexity, semantic and implmentation to then get back to this point.

## Coupling and complexity

Complexity grows with coupling. It's pretty straightforward, the more interactions between the components necessary to process a business transaction, the more complex the flow to process it is.

Interactions can happen in parallel, in sequence or both. Can fail in different ways producing multiple scenarios to handle. Some of these interactions will be need only under specific conditions.

Relaitively small systems where 4-5 components are coupled, can already get complex, especially when we add more constraints to the mix: data consistency, fault tolerance and high throughtput.

## Semantic complexity vs implementation complexity

Total system complexity = semantic complexity + implementation complexity

Semantic complexity derives from business problem itself. (sum of two number is less complex than forecast the weahter)

Implementation complexity derives by the solution chose to solve the business problem. (performing a sync HTTP call is simpler than publishing a message on a broker and wait for another message back with a worker constantly polling)

Let's make a real life example with a common coordination problem most company have: workflows implementation.

The example only have the purpose of showing how the implementation can increase complexity.

Let's suppose that as part of an e-commerce platform we have a component managing the **Checkout** phase.

The responsibility of this component are:

1. Ensure the availability of the product before proceeding with payment
2. Execute the payment
3. Ensure data consistency for availablity of the product
4. Tigger shipment workflow

Something like this

![workflow example](/images/workflow-example.png)

We can achieve the exact same functionality with countless solutions. Let's have a look at the examples above.

![workflow implmentations example](/images/workflow-implementations-example.png)

Implementation 1

Straightforward HTTP service in one codebase.
DB, external services integrations and workflow implementation are code.

Implementation 2

HTTP service integrated with a BPMN workflow engine.
The workflow engine has been set up with a workflow definition and a set of custom component implementing the steps.

## Architect choices

What's the best implementation for the job? Someone might argue that the second one it's superior for several reasons but without context there's winner solution.

### Implementation 1

Implementation 1 is simple. There's one codebase. External integrations are coded in the same place. The workflow is just set of statements and conditions. Code is executed sequentially on the same machine as a request flight in. Not much more to figure out.

When is it a good solution?
Throughput isn't high and business transactions don't take more than few seconds.
Running all the code in the same machine allows us for atomic data consistency if needed.
Single coordination point makes error handling easier.

When is it a bad solution?
Slow business transactions (minutes?). Keeping the connection open for every transaction is expensive for the client and the server. Connection might saturate fast in case of a spike.
Workflows change/add rate is frequent. Imagine 30-40 of them coded with custom code. How do you share functionalities? How do you make sure you don't break 39 workflow to change the behaviour of 1?

### Implementation 2

TODO

## Back to the beginning
