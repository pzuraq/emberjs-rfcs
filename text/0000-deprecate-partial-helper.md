- Start Date: 2017-02-28
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Deprecate `{{partial}}` helper.

# Motivation

Rendering cleanup to focus on component-based mental model.

# Transition Path

Codemod to transform into component?
Continue making Components lighter?

# How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

`{{partial}}` is already not present in the Guides, and we very much push the component mental model throughout.

> Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?

> How should this deprecation be introduced and explained to existing Ember
users?

Delineate general strategies. Point to codemod.

# Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

> What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
