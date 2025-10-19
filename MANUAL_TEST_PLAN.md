# Manual Test Plan for Issue #110 Fix

## Test Scenario 1: Username Matches Computer Name (Bug Scenario)

### Objective
Verify that a user can be created and added to the Administrators group when the username matches the computer name.

### Prerequisites
- Windows 10/11 or Windows Server 2016+ system
- cloudbase-init installed
- Computer name set to `testuser`

### Steps

1. **Set Computer Name**
   ```powershell
   # In PowerShell (Administrator)
   Rename-Computer -NewName "testuser" -Restart
   ```

2. **Prepare User Data**
   
   Create `userdata.yaml`:
   ```yaml
   #cloud-config
   users:
     - name: testuser
       passwd: TestPassword123!
       primary_group: Administrators
   ```

3. **Deploy User Data**
   - Place the userdata file in the appropriate location for your metadata service (NoCloud, ConfigDrive, etc.)
   - Or use the cloudbase-init config to point to the userdata file

4. **Run cloudbase-init**
   ```powershell
   # Stop the service if running
   Stop-Service cloudbase-init
   
   # Run manually to see output
   & "C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\cloudbase-init.exe" --config-file "C:\Program Files\Cloudbase Solutions\Cloudbase-Init\conf\cloudbase-init.conf"
   ```

5. **Verify Results**
   
   ```powershell
   # Check if user exists
   Get-LocalUser -Name testuser
   
   # Check user's group memberships
   Get-LocalGroupMember -Group "Administrators"
   
   # Should show testuser in the list
   # Expected output should include: Name: testuser, ObjectClass: User
   ```

6. **Test Login**
   - Log in as `testuser` with the password `TestPassword123!`
   - Open PowerShell and run `whoami /groups`
   - Verify that BUILTIN\Administrators is in the list with "Mandatory group" or "Enabled by default"

### Expected Results
- ✅ User `testuser` is created
- ✅ User `testuser` is a member of Administrators group
- ✅ User can login successfully
- ✅ User has administrator privileges
- ✅ No errors in cloudbase-init logs

---

## Test Scenario 2: Username Different from Computer Name (Regression Test)

### Objective
Verify that the fix doesn't break the existing functionality when username differs from computer name.

### Prerequisites
- Windows 10/11 or Windows Server 2016+ system
- cloudbase-init installed
- Computer name set to `MYCOMPUTER`

### Steps

1. **Set Computer Name**
   ```powershell
   Rename-Computer -NewName "MYCOMPUTER" -Restart
   ```

2. **Prepare User Data**
   
   Create `userdata.yaml`:
   ```yaml
   #cloud-config
   users:
     - name: adminuser
       passwd: AdminPass123!
       primary_group: Administrators
   ```

3. **Run cloudbase-init** (same as Scenario 1, step 4)

4. **Verify Results**
   ```powershell
   Get-LocalUser -Name adminuser
   Get-LocalGroupMember -Group "Administrators"
   # Should show adminuser in the Administrators group
   ```

### Expected Results
- ✅ User `adminuser` is created
- ✅ User `adminuser` is a member of Administrators group
- ✅ User can login successfully
- ✅ User has administrator privileges
- ✅ No errors in cloudbase-init logs

---

## Test Scenario 3: Multiple Groups

### Objective
Verify that users can be added to multiple groups when username matches computer name.

### Prerequisites
- Windows system with cloudbase-init
- Computer name set to `multitest`

### Steps

1. **Set Computer Name**
   ```powershell
   Rename-Computer -NewName "multitest" -Restart
   ```

2. **Create Additional Group** (if not exists)
   ```powershell
   New-LocalGroup -Name "CustomGroup" -Description "Custom test group"
   ```

3. **Prepare User Data**
   
   Create `userdata.yaml`:
   ```yaml
   #cloud-config
   users:
     - name: multitest
       passwd: MultiTest123!
       groups:
         - Administrators
         - Users
         - CustomGroup
   ```
   
   Or using cloudbase-init.conf:
   ```ini
   [DEFAULT]
   username=multitest
   groups=Administrators,Users,CustomGroup
   ```

4. **Run cloudbase-init**

5. **Verify Results**
   ```powershell
   # Check all group memberships
   Get-LocalUser -Name multitest | Get-LocalGroup
   
   # Or check each group individually
   Get-LocalGroupMember -Group "Administrators"
   Get-LocalGroupMember -Group "Users"
   Get-LocalGroupMember -Group "CustomGroup"
   ```

### Expected Results
- ✅ User `multitest` is created
- ✅ User `multitest` is a member of all specified groups
- ✅ No errors in cloudbase-init logs

---

## Test Scenario 4: Edge Cases

### 4A: Username with Special Characters
**Computer Name:** `test-user`
**Username:** `test-user`
**Expected:** User created and added to group successfully

### 4B: Case Sensitivity
**Computer Name:** `TestUser`
**Username:** `testuser` (different case)
**Expected:** User created and added to group successfully (Windows is case-insensitive for usernames)

### 4C: Unicode Characters (if supported)
**Computer Name:** `user123`
**Username:** `user123`
**Expected:** User created and added to group successfully

---

## Checking Logs

### cloudbase-init Logs Location
```
C:\Program Files\Cloudbase Solutions\Cloudbase-Init\log\cloudbase-init.log
C:\Program Files\Cloudbase Solutions\Cloudbase-Init\log\cloudbase-init-unattend.log
```

### What to Look For

**Success Indicators:**
```
Creating user "testuser" and setting password
User "testuser" created successfully
Adding user "testuser" to group "Administrators"
```

**Error Indicators (should NOT appear with the fix):**
```
Cannot add user to group "Administrators"
ERROR_NO_SUCH_MEMBER
ERROR_INVALID_MEMBER
```

---

## Rollback Plan

If issues are discovered:

1. **Remove Test User**
   ```powershell
   Remove-LocalUser -Name "testuser"
   ```

2. **Restore Computer Name**
   ```powershell
   Rename-Computer -NewName "ORIGINAL-NAME" -Restart
   ```

3. **Revert cloudbase-init**
   - Reinstall previous version of cloudbase-init

---

## Automation Script (Optional)

Here's a PowerShell script to automate some of the verification:

```powershell
# test-user-group-fix.ps1
param(
    [string]$Username = "testuser",
    [string]$GroupName = "Administrators"
)

Write-Host "Testing user creation and group membership..." -ForegroundColor Cyan

# Check if user exists
$user = Get-LocalUser -Name $Username -ErrorAction SilentlyContinue
if ($user) {
    Write-Host "✓ User '$Username' exists" -ForegroundColor Green
} else {
    Write-Host "✗ User '$Username' does NOT exist" -ForegroundColor Red
    exit 1
}

# Check group membership
$members = Get-LocalGroupMember -Group $GroupName -ErrorAction SilentlyContinue
$isMember = $members | Where-Object { $_.Name -like "*$Username" }

if ($isMember) {
    Write-Host "✓ User '$Username' is a member of '$GroupName'" -ForegroundColor Green
} else {
    Write-Host "✗ User '$Username' is NOT a member of '$GroupName'" -ForegroundColor Red
    Write-Host "Current members of ${GroupName}:" -ForegroundColor Yellow
    $members | Format-Table Name, ObjectClass
    exit 1
}

# Check if username matches computer name
$computerName = $env:COMPUTERNAME
if ($computerName -eq $Username) {
    Write-Host "✓ Username matches computer name (bug scenario)" -ForegroundColor Yellow
} else {
    Write-Host "✓ Username differs from computer name (regression test)" -ForegroundColor Yellow
}

Write-Host "`n=== All checks passed! ===" -ForegroundColor Green
```

Usage:
```powershell
.\test-user-group-fix.ps1 -Username "testuser" -GroupName "Administrators"
```

---

## Success Criteria

The fix is considered successful if:

1. ✅ All test scenarios pass
2. ✅ No errors in cloudbase-init logs
3. ✅ Users can login and have expected permissions
4. ✅ No regression in existing functionality
5. ✅ Works across different Windows versions (10, 11, Server 2016, 2019, 2022)

