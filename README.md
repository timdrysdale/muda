# muda
Zanzibar-like authorization engine extended to use time intervals object#relation@subject+from~to

Re-implementing Zanzibar properly is not a trival process. See for example [Ory Keto](https://www.ory.sh/docs/keto).

Nonetheless, the practable.io remote laboratory ecosystem requires permissions with strict time limits. The time limits need to work reliabily, even when quite short, even as short as a few tens of seconds, with perhaps one-second granularity.

So perhaps an appropriate solution is to hybridise an existing Zanzibar solution like keto, by wrapping it with a layer that adds and removes relationship tuples as required by the interval.

Each relationship tuple can be represented as a row in a table 

| ns    | id   | iat      | nbf      | exp      | obj  | rel  | sub  |
|-------|------|----------|----------|----------|------|------|------| 
| str   | uuid | datetime | datetime | datetime | str  | str  | str  |

where ns is namespace.

It's possible that there is no id needed, same as there is no id for zanzibar relationship tuples. There should probably also be columns for the constituent parts of a subject that is not just an id, but is a relationship tuple itself, or just a 

For other aspects of the implementation, similar considerations will apply as for [github.com/timdrysdale/bookable](https://github.com/timdrysdale/bookable) except that more than one relationship tuple can exist at a time for a given object and subject.

This project is a placeholder for this concept.

 

