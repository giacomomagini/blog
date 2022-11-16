---
title: "Architecture and coupling"
date: 2022-10-30T12:05:42Z
draft: false
tags: [architecture, design]
---

## Are we thinking about coupling the right way?

Few weeks ago I was reading "Software Architecture: The Hard Parts" by Neal Ford, Mark Richards, Pramod Sadalage, Zhamak Dehghani.

One concept sticked to my mind. It wasn't something completely new for my brain, but the clarity and simplicty of the sentence surprised me.
It made me think of how many times I haven't kept it in mind when dealing with architetural deicsions.

"An architect can never reduce semantic coupling via implementation,but they can make it worse"

As developer I thoguht many time I'd make a magic trick (implementation) and everything would be cleaner, better; simpler. Well, I was wrong.

If you feel I'm talking no sense, no worries! I'll clarify what I mean for coupling, complexity, semantic, implementation and how they are connected the glorious sentence in the spotlight.

## Coupling and complexity

Complexity grows with coupling. It's pretty straightforward, the more interactions between the components are necessary to process a business transaction, the more complex the flow to process the transaction is.

Interactions can happen in parallel, in sequence or both. They can fail, some on the client, some on the server. Some can be reverted, some cannot. Part of these interactions will be executed only under specific conditions.

A system with 4-5 coupled components between each other, it can already get complex. You don't think so? Let's add more constraints to the mix: data consistency, fault tolerance and high throughtput.

## Semantic complexity vs implementation complexity

Total system complexity = semantic complexity + implementation complexity

Semantic complexity derives from the business problem itself. Some problems are difficult to solve by nature: forecasting the wheater is much harder than managing doctors' appointments.

Implementation complexity derives by the solution chose to solve the business problem. From an spreadsheet to a distrubuted architecture, plenty of nuances for complexity.

## Example time

Enough talking abstract, let's make a real life example with a common coordination problem most companies have: workflows implementation.

Let's suppose that as part of an e-commerce platform we have a component managing the **Checkout** phase.

The responsibility of this component are:

1. Ensure the availability of the product before proceeding with payment
2. Execute the payment
3. Ensure atomic data consistency for product availablity and payments
4. Tigger shipment workflow

Something like this

![workflow example](/images/workflow-example.png)

We can achieve the exact same functionality with countless solutions. Below two of them from the top of my mind.

![workflow implmentations example](/images/workflow-implementations-example.png)

Implementation 1

Straightforward HTTP service in one codebase.
DB, external services integrations and workflow implementation are code.

Implementation 2

HTTP service integrated with a BPMN workflow engine.
The workflow engine has been set up with a workflow definition and a set of custom components implementing the steps.

## Architect choices

What's the best implementation for the job? Someone might argue that the second one it's superior for several reasons but without context there's no winning solution.

I won't go through an exhaustive trade-off list to choose the best solution.
I only want to communicate the concept through a simple business problem and two solutions different enough.
Let's be honest, no e-commerce would hire me for a Checkout team!

### Implementation 1

All external integrations are coded in the same place. The workflow is code as any other component in the codebase.
Code is executed sequentially on the same machine as a request flights in. Not much more to figure out.

When is it a good solution?
Low-mid throughput and retying the entire business transactions don't take more than few seconds.
Running all the code in the same machine allows us for atomic data consistency if needed.
Single coordination point makes error handling easier.

When is it a bad solution?
Slow business transactions (minutes?). Keeping the connection open for every transaction is expensive for the client and the server. Additionaly, connections might saturate fast in case of a spike.
Workflows change/add rate is frequent. Imagine 30-40 of workflows each of them coded in a custom component. How do you share common functionalities? How do you make sure you don't break 39 workflows while changing the behaviour of 1?

### Implementation 2

We have an HTTP service as an entry point.
The actual business logic is implemented with a workflow engine. We define the steps using a standard language definition (BPMN).
We could use both sync and async comminucation between the entrypoint service and the engine.
The service call the engine and wait for a response.
The engine executes all steps we defined.

When is it a good solution?

1. Mid-high throughput. The entry point service and the engine can scale independently.
2. Chage/add rate of new workflows, especislly if reusing common components. A new workflow defintion could be enough to implement a new functionality.
3. Fault tolerance and availablitiy are important: workflow engine can be distributed and replicated. It would take care of ensuring the work gets done and the status of the workflow doesn't get lost in case a sub component (e.g. a node) becomes unavailable.
4. There are enough resources avialable to maintain the engine like infrastructure, updates and documentation.

When is a bad solution?
Low throughput, even if the workflow takes
Few workflow defintions needed.
Worklflows are fast/light and can be retried from beginning to end with no consequences. No need for advanced fault tolerance strategy.
There aren't resources to maintain the engine, no software can be set up and forget!

## Back to the beginning

We've had a look at two solution for the same problem: the first one more straghtforward and second one more structured.

Did we have the chance to reduce the semantic coupling and complexity with any of the solution? No!
The business problem did get simpler and the coupling between the components is always there.

Did we increase the total complaxity with a solution? Yes!
The second solution is definitely more complex. Introducing an general purpose BPMN engine means we need to map the business problem into a BPMN rules. Now multiple workflows for different problems are operatinally coupled together in a single point; the engine.

There are good reason why we increase coupling and complexity when implementing a solution but don't be fooled by your brain: you can't solve complexity with implementations.

"An architect can never reduce semantic coupling via implementation,but they can make it worse"
