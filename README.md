# whenami
Zanzibar-like authorization engine extended to use time intervals object#relation@subject+from~to

Re-implementing Zanzibar properly is not a trival process. See for example [Ory Keto](https://www.ory.sh/docs/keto).

Nonetheless, the practable.io remote laboratory ecosystem requires permissions with strict time limits. The time limits need to work reliabily, even when quite short, even as short as a few tens of seconds, with perhaps one-second granularity.

So perhaps an appropriate solution is to hybridise an existing Zanzibar solution like keto, by wrapping it with a layer that adds and removes relationship tuples as required by the interval specification. Then users can query their permissions at the current time with the standard zanzibar interface.

Each relationship tuple can be represented as a row in a table 

| ns    | id   | iat      | nbf      | exp      | obj  | rel  | subid  | subns  | subobj | subrel |
|-------|------|----------|----------|----------|------|------|--------|--------|--------|--------| 
| str   | uuid | datetime | datetime | datetime | str  | str  | str    | str    | str    | str    |

where ns is namespace, and the subject is identified either by id (subid), or a subject set (subns, subobj, subrel) (see [proto for tuple](https://github.com/ory/keto/blob/master/proto/ory/keto/relation_tuples/v1alpha2/relation_tuples.proto)).

It's possible that there is no id needed, same as there is no id for zanzibar relationship tuples. However, it might be easier to track intervals by id rather than the combination of nbf,exp?

For other aspects of the implementation, similar considerations will apply as for [github.com/timdrysdale/bookable](https://github.com/timdrysdale/bookable) except that more than one relationship tuple can exist at a time for a given object and subject.


## The Future

TL;DR: add relations that represent time based policies, e.g. `engineering:spinnnersv2#policy:cie3-2022@(engineering:cie3#member)` and interpret policies in proxy layer running in front of keto.


Rules about when you can make a booking are easy to evaluate, because they rely only on rules that are currently loaded into keto, and this will be as fast and efficient as any other keto decision.

Where things get challenging, is evaluating future permissions about using the equipment. Can you book at this date and time? Will the same rules apply throughout the booking? 

It's impractical to answer a question about an interval with a system that can only categorically provide an answer for a given instant. How many instants within the proposed booking interval do you need to sample in order to be reasonably sure, let alone certain, that your booking request is ok? Even if you fast-forwarded keto by adding rules, you would have to play it forward maybe a second or a minute at a time through the whole interval. E.g. user wants to book a system for a day, but there is a 10min exclusive window for maintenance. So you are going to have to fast forward keto, or reason about which rules need to be added/considered in order to fully understand the interval. While keto is being fast-forwarded to the future, it cannot be used for queries at other times. All of sudden we need an instance of keto for each query. Even some clever caching won't really help with the fundamental lack of scalability of this approach.

Instead, we need to convert all points in the future into the present instant.

How? We add relations that are in fact time-based access policies representing the future, that are known to apply to requests made at the current time about the future.

Time-based policies are not understood by keto, so must be defined, stored, and interpreted externally to keto. 

We need to put a proxy in front of keto that routes "now" queries direct to keto, and "future" queries to our future permissions engine (which interacts with keto as required) 

The future permissions engine will query keto to find all relations that apply to a given object for a given subject.  For relations that represent a future time based policy, these can be loaded from the policy store, and evaluated one by one against the time interval that is requested in the query.

There are some challenges here around scope. For example, we can't just ask for policies which apply to the tuple of object-subject, because of exlusive access rules.

An exclusive access policy  disallows everyone except for those it explicitly applies to. If we have to check for exclusive access policies every time we add a new policy, to make a specific disallow policy for any new competing permitted access period, we end up with a lot of overhead, and the possibility that we get things wrong if we are interrupted part way through a rule update by failure or affect performance e.g. we change an exclusive access policy, and then need to update a large number of other policiess.

So our rule system must look for all policies that apply to an object, and understand if any exclusive access policies will prevent this subject from being allowed to do what they want, then look to see if any of their policies allow them to do it (or the other way around).

Concrete example

Engineering Design 1 laboratories run on a specific set of times and days of the week, during afternoons, but not otherwise.
Controls 3 laboratories run at any time over a period of weeks.

We want a shared usage window of n-weeks for controls 3.
We want a set of exclusive-use windows of 1 hour, or even 20min, for Engineering Design 1 students.

In the first instance we could just allow the whole class to make bookings, but then we might get people booking during a lab session that is not theirs, preventing a student from accessing what they need during the scheduled session.

So the rule system would need to be informed of the group allocations from engineering design 1, and any exceptions, e.g. students attending on another day, and set up exclusive use for them over the specified groups of kit.

As we tune those rule sets, we don't want to have to tune every other rule set for every other user in the system.


Being able to cancel bookings made by the "wrong" students and/or make bookings on their behalf would be useful, but might be something to avoid at other times ... similarly "dropping in" to a session virtually might be acceptable in a lab, but not when working from home (or only in some cases when working from home), so other permissions like joining in with student sessions would be similarly tied to specific groups at specific times.











## Appendices

### Proto for tuple

[proto for tuple](https://github.com/ory/keto/blob/master/proto/ory/keto/relation_tuples/v1alpha2/relation_tuples.proto)


```
syntax = "proto3";

package ory.keto.relation_tuples.v1alpha2;

option go_package = "github.com/ory/keto/proto/ory/keto/relation_tuples/v1alpha2;rts";
option csharp_namespace = "Ory.Keto.RelationTuples.v1alpha2";
option java_multiple_files = true;
option java_outer_classname = "RelationTuplesProto";
option java_package = "sh.ory.keto.relation_tuples.v1alpha2";
option php_namespace = "Ory\\Keto\\RelationTuples\\v1alpha2";

// RelationTuple defines a relation between an Object and a Subject.
message RelationTuple {
  // The namespace this relation tuple lives in.
  string namespace = 1;
  // The object related by this tuple.
  // It is an object in the namespace of the tuple.
  string object = 2;
  // The relation between an Object and a Subject.
  string relation = 3;
  // The subject related by this tuple.
  // A Subject either represents a concrete subject id or
  // a `SubjectSet` that expands to more Subjects.
  Subject subject = 4;
}

// Subject is either a concrete subject id or
// a `SubjectSet` expanding to more Subjects.
message Subject {
  // The reference of this abstract subject.
  oneof ref {
    // A concrete id of the subject.
    string id = 1;
    // A subject set that expands to more Subjects.
    // More information are available under [concepts](../concepts/subjects.mdx).
    SubjectSet set = 2;
  }
}

// SubjectSet refers to all subjects who have
// the same `relation` on an `object`.
message SubjectSet {
  // The namespace of the object and relation
  // referenced in this subject set.
  string namespace = 1;
  // The object related by this subject set.
  string object = 2;
  // The relation between the object and the subjects.
  string relation = 3;
}
```
