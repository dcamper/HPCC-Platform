# Security Test Plan: WUInfo Sensitive Value Redaction

## Overview
This document describes the test plan for validating the sensitive value redaction feature implemented in the WUInfo endpoint.

## Feature Description
The WUInfo endpoint now automatically detects and redacts potentially sensitive values in debug and application value fields based on field name keywords.

## Test Scenarios

### Test 1: Debug Value Redaction - Password Field
**Objective:** Verify that debug values with password-related names are redacted

**Prerequisites:**
- Access to HPCC Systems environment
- Ability to create and submit workunits

**Test Steps:**
1. Create a workunit with a debug value containing "password" in the name
   ```ecl
   #WORKUNIT('debug', 'my_password', 'secretvalue123');
   OUTPUT('test');
   ```

2. Submit the workunit and note the WUID

3. Query WUInfo for the workunit:
   ```
   GET /WsWorkunits/WUInfo?Wuid=<WUID>&IncludeDebugValues=1
   ```

4. Examine the response DebugValues section

**Expected Result:**
- The debug value named "my_password" should have value "***REDACTED***"
- Security log should contain: "Security: Redacted potentially sensitive value 'my_password' in workunit <WUID>"

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 2: Application Value Redaction - Token Field
**Objective:** Verify that application values with token-related names are redacted

**Prerequisites:**
- Access to HPCC Systems environment
- Ability to create workunits with application values

**Test Steps:**
1. Create a workunit and set an application value with "token" in the name:
   ```ecl
   #WORKUNIT('applicationValue', 'app', 'api_token', 'sk_live_123456789');
   OUTPUT('test');
   ```

2. Submit the workunit and note the WUID

3. Query WUInfo for the workunit:
   ```
   GET /WsWorkunits/WUInfo?Wuid=<WUID>&IncludeApplicationValues=1
   ```

4. Examine the response ApplicationValues section

**Expected Result:**
- The application value named "api_token" should have value "***REDACTED***"
- Security log should contain redaction warning

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 3: Non-Sensitive Values Unchanged
**Objective:** Verify that values without sensitive names are not redacted

**Prerequisites:**
- Access to HPCC Systems environment

**Test Steps:**
1. Create a workunit with normal debug values:
   ```ecl
   #WORKUNIT('debug', 'counter', '42');
   #WORKUNIT('debug', 'description', 'This is a test workunit');
   OUTPUT('test');
   ```

2. Submit the workunit and note the WUID

3. Query WUInfo for the workunit:
   ```
   GET /WsWorkunits/WUInfo?Wuid=<WUID>&IncludeDebugValues=1
   ```

4. Examine the response DebugValues section

**Expected Result:**
- All debug values should have their original values
- No redaction warnings in logs

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 4: Case Insensitivity
**Objective:** Verify that keyword detection is case-insensitive

**Prerequisites:**
- Access to HPCC Systems environment

**Test Steps:**
1. Create workunits with various case variations:
   ```ecl
   #WORKUNIT('debug', 'Password', 'test1');  // Capital P
   #WORKUNIT('debug', 'PASSWORD', 'test2');  // All caps
   #WORKUNIT('debug', 'pAsSwOrD', 'test3');  // Mixed case
   ```

2. Query WUInfo for each workunit

**Expected Result:**
- All variations should be detected and redacted
- Values should all show "***REDACTED***"

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 5: Multiple Sensitive Keywords
**Objective:** Verify all sensitive keywords are detected

**Prerequisites:**
- Access to HPCC Systems environment

**Test Steps:**
1. Create a workunit with debug values for each sensitive keyword:
   - password, passwd, pwd
   - secret, token
   - apikey, api_key, api-key
   - credential, auth, authorization
   - privatekey, private_key, privkey
   - secretkey, secret_key
   - encryptionkey, encryption_key
   - connection, connectionstring, conn_str
   - bearer, oauth

2. Query WUInfo with IncludeDebugValues=1

**Expected Result:**
- All fields with sensitive keywords should be redacted
- Approximately 20+ redaction warnings in security log

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 6: Substring Matching
**Objective:** Verify that keywords are detected within field names

**Prerequisites:**
- Access to HPCC Systems environment

**Test Steps:**
1. Create workunit with compound field names:
   ```ecl
   #WORKUNIT('debug', 'database_password', 'test1');
   #WORKUNIT('debug', 'user_api_token', 'test2');
   #WORKUNIT('debug', 'oauth_bearer_token', 'test3');
   ```

2. Query WUInfo

**Expected Result:**
- All fields should be redacted (keywords found within names)

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 7: Security Logging
**Objective:** Verify that redaction events are properly logged

**Prerequisites:**
- Access to HPCC Systems logs
- Log level set to include warnings

**Test Steps:**
1. Create a workunit with a sensitive debug value
2. Query WUInfo for the workunit
3. Check the ESP server logs

**Expected Result:**
- Log entry contains: "Security: Redacted potentially sensitive value"
- Log entry includes the field name
- Log entry includes the workunit ID

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 8: Performance Impact
**Objective:** Verify minimal performance impact

**Prerequisites:**
- Access to HPCC Systems environment
- Workunit with 100+ debug values

**Test Steps:**
1. Create a workunit with many debug values (mix of sensitive and non-sensitive)
2. Measure WUInfo response time without the fix (if possible)
3. Measure WUInfo response time with the fix
4. Compare response times

**Expected Result:**
- Response time difference should be negligible (< 5%)
- String matching is O(n*m) where n=keywords, m=field_name_length, both small

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 9: Backward Compatibility
**Objective:** Verify that existing clients continue to work

**Prerequisites:**
- Existing WUInfo client applications

**Test Steps:**
1. Test with various WUInfo API versions (1.x, 2.x)
2. Test with different IncludeDebugValues settings (true/false)
3. Test with different IncludeApplicationValues settings (true/false)

**Expected Result:**
- All API versions work correctly
- When flags are false, no values are returned (existing behavior)
- When flags are true, values are returned with sanitization applied
- XML/JSON structure remains unchanged

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Test 10: Integration Test
**Objective:** End-to-end test with real-world scenario

**Prerequisites:**
- Access to HPCC Systems environment

**Test Steps:**
1. Create a realistic ECL query that might use debug values for tracking
2. Accidentally set a debug value that looks like a credential:
   ```ecl
   #WORKUNIT('debug', 'connection_string', 'Server=myserver;Password=mypassword');
   ```
3. Submit and query via WUInfo

**Expected Result:**
- The connection_string value is redacted
- Other query metadata is intact
- User can still see workunit status and results

**Actual Result:**
_To be filled during testing_

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

## Security Test Scenarios

### Security Test 1: Credential Leak Prevention
**Objective:** Verify that actual credentials cannot be extracted

**Test Steps:**
1. Create a workunit with a debug value containing a real-looking credential:
   ```
   name: "database_password"
   value: "P@ssw0rd123!@#"
   ```
2. Attempt to retrieve via various methods:
   - WUInfo SOAP request
   - WUInfo REST request
   - Direct WSDL query
   - XML/JSON response parsing

**Expected Result:**
- All methods should return "***REDACTED***"
- No method should expose the actual credential

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Security Test 2: Bypass Attempt - Case Manipulation
**Objective:** Verify that case manipulation cannot bypass detection

**Test Steps:**
1. Try various case patterns:
   - pAsSwOrD, PaSsWoRd, PASSWORD, password
   - ToKeN, TOKEN, token
   - etc.

**Expected Result:**
- All variations are detected and redacted

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Security Test 3: Bypass Attempt - Unicode/Special Characters
**Objective:** Verify that special characters don't break detection

**Test Steps:**
1. Create field names with special characters:
   - "my-password"
   - "my_password"
   - "my.password"
   - "mypassword123"

**Expected Result:**
- All variations containing "password" are detected

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

## Regression Tests

### Regression Test 1: Non-Sensitive Workunit Still Works
**Objective:** Verify that normal workunits are unaffected

**Test Steps:**
1. Query WUInfo for a workunit without any debug/application values
2. Query WUInfo for a workunit with only non-sensitive debug values

**Expected Result:**
- All responses work as before
- No redaction occurs
- No warnings in logs

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

### Regression Test 2: WUInfo Without Include Flags
**Objective:** Verify behavior when debug/application values are not requested

**Test Steps:**
1. Query WUInfo with IncludeDebugValues=false
2. Query WUInfo with IncludeApplicationValues=false

**Expected Result:**
- No debug/application values in response (as before)
- No redaction performed (optimization)
- No warnings in logs

**Status:** ⬜ Not Run / ✅ Pass / ❌ Fail

---

## Test Environment Requirements

### Software Requirements
- HPCC Systems Platform (version with fix applied)
- ECL IDE or command-line tools
- Access to ESP server logs

### Access Requirements
- User account with workunit creation permissions
- User account with WUInfo query permissions
- Access to system logs (for verification)

### Data Requirements
- No sensitive data should be used in actual tests
- Use mock/dummy credentials for testing

---

## Test Execution Checklist

- [ ] Test environment prepared
- [ ] Test data created
- [ ] Security logs configured for monitoring
- [ ] Test 1: Password field redaction
- [ ] Test 2: Token field redaction
- [ ] Test 3: Non-sensitive values unchanged
- [ ] Test 4: Case insensitivity
- [ ] Test 5: Multiple keywords
- [ ] Test 6: Substring matching
- [ ] Test 7: Security logging
- [ ] Test 8: Performance impact
- [ ] Test 9: Backward compatibility
- [ ] Test 10: Integration test
- [ ] Security Test 1: Credential leak prevention
- [ ] Security Test 2: Case manipulation bypass
- [ ] Security Test 3: Special characters bypass
- [ ] Regression Test 1: Non-sensitive workunit
- [ ] Regression Test 2: Without include flags
- [ ] All test results documented
- [ ] Issues filed for any failures
- [ ] Test report generated

---

## Test Results Summary

**Test Date:** _______________

**Tester:** _______________

**Environment:** _______________

**Total Tests:** 15

**Passed:** _____

**Failed:** _____

**Not Run:** _____

**Critical Issues Found:** _____

**Notes:**
_________________________________________________________________
_________________________________________________________________
_________________________________________________________________

---

## Appendix A: Sample ECL Test Scripts

### Script 1: Comprehensive Test Workunit
```ecl
// Test workunit for sensitive value redaction
#WORKUNIT('name', 'Security Redaction Test');

// Sensitive values that should be redacted
#WORKUNIT('debug', 'database_password', 'my_secret_password_123');
#WORKUNIT('debug', 'api_token', 'sk_live_1234567890abcdef');
#WORKUNIT('debug', 'secret_key', 'very_secret_value');
#WORKUNIT('debug', 'oauth_bearer', 'bearer_token_value');
#WORKUNIT('debug', 'connection_string', 'Server=srv;Password=pwd');

// Non-sensitive values that should NOT be redacted
#WORKUNIT('debug', 'counter', '42');
#WORKUNIT('debug', 'description', 'Test workunit for security');
#WORKUNIT('debug', 'timestamp', '2025-11-05T12:00:00');
#WORKUNIT('debug', 'user_name', 'testuser');  // Note: not sensitive
#WORKUNIT('debug', 'iteration', '5');

OUTPUT('Security redaction test completed');
```

### Script 2: Application Values Test
```ecl
// Test application values redaction
#WORKUNIT('name', 'Application Values Test');
#WORKUNIT('applicationValue', 'myapp', 'api_key', 'sensitive_api_key_value');
#WORKUNIT('applicationValue', 'myapp', 'version', '1.0.0');
#WORKUNIT('applicationValue', 'myapp', 'credentials', 'user:password');
#WORKUNIT('applicationValue', 'myapp', 'description', 'My application');

OUTPUT('Application values test completed');
```

---

## Appendix B: Sample SOAP Request
```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:ws="urn:hpccsystems:ws:wsworkunits">
  <soap:Body>
    <ws:WUInfoRequest>
      <ws:Wuid>W20231105-123456</ws:Wuid>
      <ws:IncludeDebugValues>true</ws:IncludeDebugValues>
      <ws:IncludeApplicationValues>true</ws:IncludeApplicationValues>
    </ws:WUInfoRequest>
  </soap:Body>
</soap:Envelope>
```

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-05  
**Status:** Draft
