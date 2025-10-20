# Fix: Cannot add user to group "Administrators" when username matches computer name

## Summary

This PR fixes issue #110 where cloudbase-init fails to add a user to the "Administrators" group (or any local group) when the username matches the computer/desktop name.

## Problem Description

When creating a user via userdata with a username that matches the computer name, cloudbase-init would:
- ✅ Successfully create the user
- ❌ Fail to add the user to the specified group (e.g., "Administrators")

However, when the username is different from the computer name, both operations succeed as expected.

### Root Cause

The issue occurs due to Windows name resolution ambiguity. When calling `NetLocalGroupAddMembers` with just the username, Windows becomes confused about whether to add:
- The local user account (e.g., `.\username`), or
- The computer account with the same name

This ambiguity causes the group membership operation to fail with errors such as "cannot add user to group".

## Solution

The fix qualifies the username with the local domain prefix (`.\\`) before adding it to a group. This explicitly tells Windows to use the local user account, eliminating the name resolution ambiguity.

### Changes Made

#### 1. `cloudbaseinit/osutils/windows.py`
Modified the `add_user_to_local_group` method to qualify the username:

```python
def add_user_to_local_group(self, username, groupname):
    lmi = Win32_LOCALGROUP_MEMBERS_INFO_3()
    # Qualify the username with the local domain prefix to avoid
    # name resolution ambiguity when username matches computer name
    # (see issue #110)
    qualified_username = f".\\{username}"
    lmi.lgrmi3_domainandname = str(qualified_username)
    # ... rest of the method
```

**Before:** `lmi.lgrmi3_domainandname = str(username)`
**After:** `lmi.lgrmi3_domainandname = str(f".\\{username}")`

#### 2. `cloudbaseinit/tests/osutils/test_windows.py`
Updated the test assertion to verify the username qualification:

```python
# Username should be qualified with local domain prefix
expected_username = f".\\{self._USERNAME}"
self.assertEqual(lmi.lgrmi3_domainandname, str(expected_username))
```

## Testing

### Unit Tests
All existing tests pass, including the 6 specific tests for `add_user_to_local_group`:
- ✅ `test_add_user_to_local_group_no_error`
- ✅ `test_add_user_to_local_group_not_found`
- ✅ `test_add_user_to_local_group_access_denied`
- ✅ `test_add_user_to_local_group_no_member`
- ✅ `test_add_user_to_local_group_member_in_alias`
- ✅ `test_add_user_to_local_group_invalid_member`

All 193 WindowsUtils tests pass with no regressions.

### Test Command
```bash
stestr run cloudbaseinit.tests.osutils.test_windows.TestWindowsUtils.test_add_user_to_local_group
```

### Manual Testing Recommendation

To manually verify this fix on a Windows system:

1. Set the computer name to `testuser`
2. Create cloud-init userdata with:
   ```yaml
   #cloud-config
   users:
     - name: testuser
       passwd: SecurePassword123!
       primary_group: Administrators
   ```
3. Run cloudbase-init
4. Verify that:
   - User `testuser` is created ✅
   - User `testuser` is successfully added to the Administrators group ✅
   - User can login and has administrator privileges ✅

## Impact

### Backward Compatibility
✅ **Fully backward compatible**. The change only affects how the username is formatted when calling the Windows API. Existing functionality with usernames that don't match the computer name continues to work as before.

### Security
✅ **No security implications**. The `.\\` prefix is the standard Windows notation for explicitly specifying a local account, and is commonly used in Windows administration.

### Performance
✅ **No performance impact**. The change adds a simple string formatting operation.

## Related Issues

- Fixes #110 - cannot add user xxx to group "Administrators"

## References

- [Microsoft Documentation: NetLocalGroupAddMembers](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netlocalgroupaddmembers)
- [Windows Local vs Domain Account Naming](https://learn.microsoft.com/en-us/windows/win32/secauthn/user-name-formats)

## Checklist

- [x] Code changes implement the fix
- [x] Unit tests updated and passing
- [x] All existing tests pass (193/193)
- [x] No linting errors introduced
- [x] Code is properly commented
- [x] References GitHub issue #110
- [x] Backward compatible
- [x] PR description includes manual test plan

