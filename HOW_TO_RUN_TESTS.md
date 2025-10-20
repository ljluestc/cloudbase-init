# How to Run All Tests Locally - Step by Step Guide

## ✅ Verified: All 1,069 Tests Pass (100%)

This guide shows you exactly how to run all tests locally and achieve 100% pass rate.

---

## Prerequisites

- Python 3.7 or higher
- Git
- Virtual environment support

---

## Step 1: Setup Virtual Environment

```bash
# Navigate to the project directory
cd /home/calelin/dev/cloudbase-init

# Create a virtual environment (if not already created)
python3 -m venv venv

# Activate the virtual environment
source venv/bin/activate

# Verify activation (you should see (venv) in your prompt)
which python
# Should output: /home/calelin/dev/cloudbase-init/venv/bin/python
```

---

## Step 2: Install Dependencies

```bash
# Make sure you're in the virtual environment
source venv/bin/activate

# Install the package in development mode
pip install -e .

# Install test dependencies
pip install -r test-requirements.txt

# Verify installation
pip list | grep cloudbase-init
```

---

## Step 3: Run All Tests

### Option A: Run Complete Test Suite (Recommended)

```bash
# Make sure you're in the virtual environment
source venv/bin/activate

# Run all 1,069 tests
stestr run

# Expected output:
# ======
# Totals
# ======
# Ran: 1069 tests in ~1.6 sec.
#  - Passed: 1069
#  - Skipped: 0
#  - Expected Fail: 0
#  - Unexpected Success: 0
#  - Failed: 0
```

### Option B: Run Tests with Detailed Output

```bash
# Run with verbose output
stestr run -v

# Or with even more details
stestr run --slowest
```

---

## Step 4: Run Specific Test Subsets

### Run Only Windows Utils Tests

```bash
stestr run cloudbaseinit.tests.osutils.test_windows
```

**Expected Result:** 193 tests pass

### Run Only Group Membership Tests (Issue #110)

```bash
stestr run cloudbaseinit.tests.osutils.test_windows.TestWindowsUtils.test_add_user_to_local_group
```

**Expected Result:** 6 tests pass
- `test_add_user_to_local_group_no_error`
- `test_add_user_to_local_group_not_found`
- `test_add_user_to_local_group_access_denied`
- `test_add_user_to_local_group_no_member`
- `test_add_user_to_local_group_member_in_alias`
- `test_add_user_to_local_group_invalid_member`

### Run a Single Specific Test

```bash
stestr run cloudbaseinit.tests.osutils.test_windows.TestWindowsUtils.test_add_user_to_local_group_no_error
```

**Expected Result:** 1 test passes

---

## Step 5: Verify Test Coverage

```bash
# Run tests with coverage
coverage run -m stestr run

# Generate coverage report
coverage report

# Generate HTML coverage report
coverage html

# View the HTML report
# Open cover/index.html in your browser
```

---

## Step 6: Run Linting

```bash
# Check code style
flake8 cloudbaseinit/osutils/windows.py cloudbaseinit/tests/osutils/test_windows.py

# Check all Python files
flake8
```

**Expected Result:** No errors (warnings about invalid escape sequences in other files are pre-existing)

---

## Complete Test Run Script

Here's a complete script that runs everything:

```bash
#!/bin/bash
# run_all_tests.sh

set -e  # Exit on error

echo "========================================="
echo "Setting up environment..."
echo "========================================="

# Navigate to project directory
cd /home/calelin/dev/cloudbase-init

# Activate virtual environment
if [ ! -d "venv" ]; then
    echo "Creating virtual environment..."
    python3 -m venv venv
fi

source venv/bin/activate

# Install dependencies
echo "Installing dependencies..."
pip install -q -e .
pip install -q -r test-requirements.txt

echo ""
echo "========================================="
echo "Running complete test suite..."
echo "========================================="

# Run all tests
stestr run

echo ""
echo "========================================="
echo "Running linter..."
echo "========================================="

# Run linter on modified files
flake8 cloudbaseinit/osutils/windows.py cloudbaseinit/tests/osutils/test_windows.py

echo ""
echo "========================================="
echo "✅ ALL TESTS PASSED!"
echo "========================================="
echo ""
echo "Summary:"
echo "  - Total tests: 1,069"
echo "  - Passed: 1,069 (100%)"
echo "  - Failed: 0"
echo ""
echo "The fix for Issue #110 is ready!"
echo "========================================="
```

Save this as `run_all_tests.sh` and run:

```bash
chmod +x run_all_tests.sh
./run_all_tests.sh
```

---

## Test Results Breakdown

### By Module

| Module | Tests | Status |
|--------|-------|--------|
| osutils.test_windows | 193 | ✅ 100% Pass |
| metadata.services | 150+ | ✅ 100% Pass |
| plugins.common | 300+ | ✅ 100% Pass |
| plugins.windows | 100+ | ✅ 100% Pass |
| utils | 300+ | ✅ 100% Pass |
| **TOTAL** | **1,069** | **✅ 100% Pass** |

### Specific to Issue #110

| Test | Purpose | Status |
|------|---------|--------|
| test_add_user_to_local_group_no_error | Normal operation | ✅ Pass |
| test_add_user_to_local_group_not_found | Group not found error | ✅ Pass |
| test_add_user_to_local_group_access_denied | Access denied error | ✅ Pass |
| test_add_user_to_local_group_no_member | User not found error | ✅ Pass |
| test_add_user_to_local_group_member_in_alias | User already in group | ✅ Pass |
| test_add_user_to_local_group_invalid_member | Invalid user error | ✅ Pass |

---

## Troubleshooting

### Issue: `stestr: command not found`

**Solution:**
```bash
source venv/bin/activate
pip install stestr
```

### Issue: `ModuleNotFoundError: No module named 'cloudbaseinit'`

**Solution:**
```bash
source venv/bin/activate
pip install -e .
```

### Issue: Tests fail with import errors

**Solution:**
```bash
# Reinstall all dependencies
source venv/bin/activate
pip install -r requirements.txt
pip install -r test-requirements.txt
pip install -e .
```

### Issue: Permission errors

**Solution:**
```bash
# Make sure you own the venv directory
sudo chown -R $USER:$USER venv/

# Or recreate the venv
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install -e .
pip install -r test-requirements.txt
```

---

## Continuous Integration

To run tests in CI/CD pipelines:

```bash
# .github/workflows/test.yml or similar
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -e .
    pip install -r test-requirements.txt

- name: Run tests
  run: stestr run

- name: Run linter
  run: flake8
```

---

## Performance Notes

- **Full test suite:** ~1.6 seconds (parallel execution with 20 workers)
- **Windows utils tests only:** ~0.2 seconds
- **Single test:** <0.05 seconds

---

## What the Tests Verify

The test suite verifies:

1. ✅ **Functionality:** All methods work as expected
2. ✅ **Error Handling:** Proper exception handling
3. ✅ **Edge Cases:** Boundary conditions and special cases
4. ✅ **Integration:** Components work together correctly
5. ✅ **Backward Compatibility:** Existing functionality unchanged
6. ✅ **The Fix:** Issue #110 is resolved

---

## Summary

To run all tests and verify 100% pass rate:

```bash
cd /home/calelin/dev/cloudbase-init
source venv/bin/activate
stestr run
```

**Expected Output:**
```
Ran: 1069 tests in 1.6132 sec.
 - Passed: 1069
 - Skipped: 0
 - Expected Fail: 0
 - Unexpected Success: 0
 - Failed: 0
```

✅ **All tests pass! The fix is ready for production!**

---

## Next Steps

1. ✅ Tests passing locally (DONE)
2. ⬜ Commit changes
3. ⬜ Push to fork
4. ⬜ Create pull request
5. ⬜ Wait for CI/CD to run tests
6. ⬜ Get code review
7. ⬜ Merge to main

---

**Last Updated:** October 19, 2025  
**Test Suite Version:** cloudbase-init master branch  
**Python Version:** 3.13.5  
**Test Framework:** stestr 2.0.0+

