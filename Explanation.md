# Bug Fix Explanation: OAuth2 Token Refresh Failure

<img width="8192" height="1736" alt="Flow chart" src="https://github.com/user-attachments/assets/99148f5c-2c90-44c3-8c72-6460824c8911" />

## Overview
This document details a critical bug fix in the `Client` class within `app/http_client.py`. The issue prevented the application from refreshing expired or invalid OAuth2 tokens when they were stored as dictionaries (e.g., loaded from a config file) instead of `OAuth2Token` objects.

## The Bug
**File:** `app/http_client.py`
**Affected Method:** `Client.request`

The original implementation contained a logical flaw in how it determined whether to refresh the OAuth2 token. The problematic code was:

```python
# app/http_client.py (Original Logic)
if not self.oauth2_token or (
    isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired
):
    self.refresh_oauth2()
```

### Why it failed
This condition failed to trigger a refresh in a specific scenario: **when `self.oauth2_token` was a non-empty dictionary.**

1.  `if not self.oauth2_token`: Evaluates to `False` because a non-empty dictionary is truthy.
2.  `isinstance(..., OAuth2Token)`: Evaluates to `False` because the token is a `dict`, not an instance of `OAuth2Token`.
3.  **Result:** Both sides of the `OR` condition were false, so `self.refresh_oauth2()` was skipped.

Consequently, the code proceeded to set the Authorization header using:
```python
if isinstance(self.oauth2_token, OAuth2Token):
    headers["Authorization"] = self.oauth2_token.as_header()
```
Since `self.oauth2_token` was a dict, this check also failed, resulting in the request being sent **without any Authorization header**, leading to authentication errors (e.g., 401 Unauthorized).

## The Fix
We simplified the conditional check to strictly enforce that we must have a valid `OAuth2Token` object. If we have *anything else* (None, dict, expired token), we must refresh.

**Corrected Code:**
```python
# app/http_client.py (Fixed Logic)
if not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired:
    self.refresh_oauth2()
```

### Verification
This fix ensures correct behavior in all states:
- **None**: `not isinstance` is True -> **Refresh Triggered**.
- **Dictionary**: `not isinstance` is True -> **Refresh Triggered** (Fixes the bug).
- **Expired Token**: `expired` is True -> **Refresh Triggered**.
- **Valid Token**: Both checks False -> **No Refresh** (Correct behavior).

A regression test was added in `tests/test_http_client.py` (`test_api_request_refreshes_when_token_is_dict`) to confirm that dictionary tokens now correctly trigger a refresh and produce a valid Authorization header.

## Uncovered Edge Case: Refresh Failure Handling

The current implementation assumes `self.refresh_oauth2()` always succeeds.
```python
def refresh_oauth2(self) -> None:
    # Simulates fetching a new token
    self.oauth2_token = OAuth2Token(access_token="fresh-token", expires_at=10**10)
```
In a production environment, this method would perform a network request which could fail due to:
- Network connectivity issues.
- Invalid refresh token (e.g., revoked or expired).
- Server-side errors (5xx).

**Risk:** If `refresh_oauth2()` raises an exception, the application may crash or propagate the error up the stack without context. If it fails silently (e.g., returns `None` or keeps the old token), the subsequent API request will fail with 401.

**Recommendation:**
Wrap the refresh logic in a try-except block to handle potential failures gracefully (e.g., retry logic, user re-authentication prompt, or structured error logging).
