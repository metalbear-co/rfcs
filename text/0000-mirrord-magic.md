- Feature Name: mirrord Magic
- Start Date: 2026-02-13
- Last Updated: 2026-02-13
- RFC PR: [metalbear-co/rfcs#TBD](https://github.com/metalbear-co/rfcs/pull/TBD)

## Summary
[summary]: #summary

Add new configuration field under the root config, called "magic".
"magic" would cover features that are enabled by default that help 99% of the users,
but might annoy the other 1%. They are magic because they're not specific use of a feature,
but a combination of mirrord features to make a use case easier.

## Motivation
[motivation]: #motivation

Take the use case of using AWS CLI within mirrord. To work properly, it requires:
1. unset AWS_PROFILE (and other related env)
2. make ~/.aws not found - that specifically was broken in last AWS cli version it seems, so new implementation would be "make tmpdir, use mapping to send aws there"

I anticipate as mirrord gets more edge use cases, we'd have more "magic" logic that we can't explain into a specific feature, so we can bundle it as a "magic"

Another example would be the Santa cheats I implemented.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

User wants to run `mirrord exec -t deployment/lala -- bash` then `aws sts get-caller-identity`. We'd like that to work out of the box.
Then, for another user that does something very specific, they'd like to actually use local AWS credentials, so they'd do 

```json
{
  "magic": {
    "aws": "true"
  }
}
```

Would lead to


```json
{
  "feature": {
    "fs": {
      "mapping": {"/Users/aaa/.aws": "/tmp/aws/$0"}
    },
    "env": {
      "unset": ["AWS_PROFILE"]
    }
  }
}
```

As we add more features, we'll enable also `"magic": false` to turn off all magic.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add new field under root config called `magic`. Start with `aws` magic feature. Have a very full explanation of what the magic does and how it works.


## Drawbacks
[drawbacks]: #drawbacks

- Can be hard for users to find themselves
- Testing the magic implementations would be hard

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives


What other designs have been considered and what is the rationale for not choosing them?

- Just implement the magic with defaults for existing features. Can be complex, and overriding the logic would be hard.
 

## Prior art
[prior-art]: #prior-art



## Unresolved questions
[unresolved-questions]: #unresolved-questions


## Future possibilities
[future-possibilities]: #future-possibilities
