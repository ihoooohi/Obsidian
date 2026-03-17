## Q1: What's the code smell?

The `amountOf` method only utilizes information from the `Rental` object (via `getMovie().getPriceCode()` and `getDaysRented()`), without utilizing any information from the `Customer` class.

So this method should belong to the Rental class rather than the Customer class. 

## Q2: Why?

This code also only utilizes information from the Rental object:
- each.getMovie().getPriceCode()
- each.getDaysRented()

And there is no information from the Customer class used at all. It violate **Single Responsibility Pricinple**

## Q3: What's good about using methods to replace temporary variables?

If there are multiple places where total rent or total points need to be calculated, methods can now be reused to avoid code duplication. 

If the calculation logic changes, only one place (within the method) needs to be modified, rather than multiple places

## Q4: Why?

Users do not need to know that the internal representation of children's tablets is an integer value of 2. They only need to call beChildrens() to set the type

If the internal implementation changes in the future, only these methods need to be modified, without requiring modifications to the caller

Through private constructors and static factory methods, classes can fully control how objects are created. Users cannot directly use `new Movie(title, priceCode)`, and must use the provided factory methods instead

## Q5: What's good about it?

Obey open-closed principle. When adding a new type, simply create a new subclass of Price.

Each pricing strategy manages its own billing and point rules.

Altering a specific film rule will not affect other genres.
## Q6: What design pattern are we using?

Adapter Pattern

## Q7: We can do similar things as charge. But can we do better?

In the Price base class, provide a default implementation that returns 1. Only NewReleasePrice covers this method (returning 2 when the lease period is > 1)

This can reduce duplicate code, as most types of integration rules are the same, and there is no need to write them out for each subclass.