---
layout: single
author_profile: false
title:  "Rolling our own address validation for free"
date:   2020-05-15 21:17:08 -0500
categories: [tech]
---

I have a distinct childhood memory of dropping an envelope into a mailbox with no stamps on it, only to have it returned, with a handwritten note from my dad explaining why I couldn’t do that.
This was my first hard lesson in postal service validation, but it wouldn’t be my last.

At our e-commerce company, our checkout form made the street, city, state, and zip code fields required. But even with this basic validation, it turns out people (and browsers with their auto-fill capabilities) have some pretty funny ideas on what makes a true address.
Once our fulfillment team started getting addresses that the postal service deemed invalid on a daily basis, it was time for more robust validation.

There are many address validation services out there, but they all cost money, and none of them exactly suited our needs. Even the most robust address provider could give us false negatives or positives. So in any case, we would need to provide our own validation layer on top of whatever we chose. Before resorting to paying for a service, we decided to do what we could with a free API like Google Maps.

Our primary goal was to reduce the number of incoming invalid addresses to nearly zero. Given that this validation would happen at checkout, we couldn’t discourage the user from making their purchase. Our service needed to help people enter a valid address as quickly—and painless—as possible.

A bad address can take many forms. The most common are:

### 1. Missing address fields

For the most past, this was being addressed by making all our address fields required (Street, City, State and Zip Code). All of these fields are text fields, except for the State field, which is a dropdown of state abbreviations. (Luckily, we only had to consider US addresses for this task.)

Of course, simply making these fields required doesn’t ensure that the content of each field is complete or correct. This is when you get:

### 2. Unspecific streets

For example, a street with no number, like “Easy Street” instead of “123 Easy Street.”

### 3. Information reversal

One could put a zip code in the city field, a primary street in the secondary street field, and so on.

We can validate on our own that the zip code field is only numbers and hyphens, for instance, but using a third party service adds another layer. In these cases, a third party service is helpful, but not required.

### 4. Complete, but misspelled, streets

This is where a third party service becomes essential, since it requires querying a large data set of addresses to figure out if the address actually exists.

### 5. Incorrect information

Another area where only a third party service could help. One very common scenario is simply having the wrong zip code for a street/city combination. In many cases, the zip code if off by one or two digits.

### 6. Valid, but unintended, addresses

This problem can’t be solved by an address validation service like this. If an address exists and is formatted correctly, but isn’t yours, it would still pass this validation. This would be handled in other ways beyond the scope of this post.

## Our Process

### Ensure the user gives a correctly formatted address

It’s important to note that the Google Maps API wasn’t really built to validate addresses — it uses a more general idea of “places.” A “place” could be a specific address, like “123 Easy Street, Chicago, IL, 12345, USA” or it could be super general, like “Chicago, IL, USA.” When passed input like “Chicago,” It could return a result like “Chicago, IL, USA” which isn’t helpful to us.

So we had to make sure we were passing Google sufficiently specific input to get results that would be considered real addresses by the postal service. This was taken care of by our front-end validation, which ensures specificity down to the street address level.

### Select a Google result to validate against

Once we have a full address from the user, we send this to Google and get a list of place results.

The Google maps API is very flexible, and will return at least one result if you pass it enough info. The task for us was to choose which of these results would be the closest match to the user’s intended address.

Why couldn’t we validate against all Google results? We didn’t for a few reasons: 1) this was time-consuming and could vary greatly based on the number of Google responses. And 2) Google returns results with a descending degree of specificity, the first being the most specific. For example, a result at the front of the list would be “123 South Easy Street, Chicago, IL, 12345 USA” while a third or fourth result would be “Chicago, IL, USA.” So only the first few results would be worth considering.

Google’s API assumes a single string of text as input, and it heavily weights the street value for determining its results. A user giving an incorrect or incomplete street could result in Google returning results for entirely different cities.

Unlike Google, we couldn’t rely on the street as the sole basis for the entire address’ accuracy. However, combining street with zip code as a secondary basis greatly reduced our false positives.

A user is likely to get either their street wrong or their zip code wrong. They’re a lot less likely to get both wrong at the same time. So whenever we got results from Google, we’d scan the results for a match on the zip code first, since Google ostensibly narrowed down the street results.

### First layer of validation: Levenshtein Distance matching

Once we have what we believe is the best Google result to validate against, we normalize both the user-inputted address and Google’s address into strings of text. We then pass both addresses through a text matching service to determine how different the two strings are from each other.

For this service, we use the [Levenshtein Distance](https://en.wikipedia.org/wiki/Levenshtein_distance){:target="_blank"} algorithm, which returns a numeric value representing how divergent two texts are from one another. The larger the number, the more the texts diverge.

Using the last few weeks of user-submitted addresses as our initial test data, we determined a numeric threshold from the Levenshtein results. Above this threshold, the user’s address is considered invalid. This was a matter of tweaking the strength of the threshold, so it could catch truly incorrect addresses but still allow for slight variations in spelling.

Spelling is where this became less of an exact science. It’s more straightforward to catch an incorrect zip code, or even misspellings of a street address. It’s more nuanced to catch alternative spellings of a street address.

Google maps give you a “short” and “long” version of each component of a street address. For example, “South Easy Street” would be the long version, “S. Easy St.” the short version. If one writes an entire address in either the “short” way or the “long” way, then the Levenshtein algorithm is quite accurate.

But an address with a combination of short and long variations in spelling may not pass the Levenshtein threshold, resulting in a false negative.

For example, “South Easy St.” uses both a long word (“South” instead of “S.”) and short word (“St.” instead of “Street”) . Even though it’s colloquially a valid address, this may be caught as invalid by our string matching because, according to the algorithm, it’s too divergent from the long form provided by Google. These edge cases depended on the address length itself. Shorter addresses were more likely to be caught by this edge case than longer addresses.

In theory, we could run the Levenshtein matching against all possible combinations of long and short name versions of the same result that Google gives us. But again, time was of the essence, and we wanted to see if this more naive implementation could suffice before we resorted to that approach.

When an address is invalidated by this layer, we return a complete address suggestion, using the long form of the address: “Did you mean: 123 South Easy Street, Chicago, IL 12345?”

### Second layer of validation: field-specific errors

Most of our invalid addresses are caught at the string-matching layer. But we still get incorrect addresses that pass under the Levenshtein threshold. These are errors like city misspellings, wrong state selection, or a zip code that’s off by a digit or two.

In these cases, we use the parsed result from Google to validate the rest of the user fields — City, State, and Zip Code — against each google field. If any of these fields don’t match their Google counterpart, we return a more specific error: “Zip Code Invalid: Did you mean 60640?”

### Frictionless handling of mistakes

It would be impossible for us to suggest the exact intended address 100% of the time, but we get pretty close with the above methods.

For any invalid address, the user is able to accept our suggestion with a single click of a button. Or, just by giving a suggestion, it becomes clear to them what might be wrong about their address, and they correct it manually.

## Results

This system sufficiently handles our needs for the overwhelming majority of cases. We get almost no feedback from the fulfillment team about wrong addresses, and our UI hasn’t prevented people from checking out. Because we save a user’s previously provided address to use on subsequent checkouts, the validation service is touched less often once initially used.

This project was an exercise in weighing probabilities. What were the most likely address inaccuracies? What were the least likely? How could we get a solution that, while not 100% effective, could confidently handle the most likely scenarios?

So much of our work as software engineers is about making tradeoffs. This project gave me good insight into how to select a third-party service to achieve a business goal, how to set realistic expectations for that service, and how to weigh tradeoffs that stretch across the stack.