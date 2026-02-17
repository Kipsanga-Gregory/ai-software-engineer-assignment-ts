# Explanation

## What was the bug?

The `HttpClient` failed to refresh the authentication token when `oauth2Token` was set to a plain object (e.g., `{ accessToken: "stale", expiresAt: 0 }`) instead of an `OAuth2Token` instance. The code treated the plain object as a valid token, resulting in requests being sent without the required `Authorization` header.

## Why did it happen?

The refresh condition was:
```typescript
!this.oauth2Token || (this.oauth2Token instanceof OAuth2Token && this.oauth2Token.expired)
```

When `oauth2Token` was a plain object:
1. `!this.oauth2Token` evaluated to `false` (because the object is truthy).
2. `instanceof OAuth2Token` evaluated to `false` (it's a plain object, not a class instance).

So the refresh was skipped. The subsequent header injection also required an `instanceof` check, which also failed — leaving the `Authorization` header empty.

## Why does the fix solve it?

The condition was changed to:
```typescript
!(this.oauth2Token instanceof OAuth2Token) || this.oauth2Token.expired
```

Now, if `oauth2Token` is `null`, a plain object, or anything other than an `OAuth2Token` instance, `instanceof` returns `false`, making the expression `true`. This forces a call to `refreshOAuth2()`, ensuring a valid `OAuth2Token` is always available before the `Authorization` header is set.

## One edge case the tests don't cover

The tests don't cover the scenario where `refreshOAuth2()` throws an exception. The current implementation is a stub, but if it were extended to call an external service, a thrown error would propagate up uncaught as there is no error handling or fallback around the refresh call, potentially leaving the client in an inconsistent state.
