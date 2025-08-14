# Migration instructions to the new Auth experience

:::info
TODO: This document is a work-in-progress to track the breaking changes in the new authentication experience. It will be updated to its final state once the feature is completed and this note should be removed.
:::

## Switch from `Basic` to `Bearer` default auth header

In the new authentication system, the default authentication header has changed from `Basic` to `Bearer` - which is now officially supported. This introduced a change for custom `AuthenticationHandler` implementations: the `token` parameter will now receive the token unwrapped from the `Bearer` prefix - just as it does for `Basic` tokens.

If your application relies on the default `authenticationHandler` function, no change is required. Only custom implementations that previously implemented `Bearer` authentication will need to be updated to expect the unwrapped token.
