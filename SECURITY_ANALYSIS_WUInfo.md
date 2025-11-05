# Security Analysis: WsWorkunits WUInfo WSDL Endpoint

## Executive Summary

This document provides a comprehensive security analysis of the `WUInfo` endpoint exposed via WSDL at `https://play.hpccsystems.com:18010/WsWorkunits/WUInfo?ver_=2.02&wsdl`. The analysis identifies potential security concerns and provides recommendations for mitigation.

**Analysis Date:** 2025-11-05  
**WSDL Version Analyzed:** 2.02  
**Repository:** dcamper/HPCC-Platform

## Overview

The `WUInfo` endpoint is part of the WsWorkunits service, which provides detailed information about workunits (execution units in the HPCC Systems platform). This endpoint returns extensive workunit metadata, including execution details, results, source code, and internal system information.

## Security Assessment

### 1. Information Disclosure Vulnerabilities

#### 1.1 ECL Source Code Exposure (HIGH RISK)
**Location:** `esp/scm/ws_workunits_struct.ecm` - ECLQuery structure  
**Field:** `Query.Text`

**Issue:**
The endpoint exposes the complete ECL (Enterprise Control Language) source code of workunits through the `Query.Text` field. While there's an option to truncate at 64KB (`TruncateEclTo64k`), the default behavior exposes the full source code.

**Security Implications:**
- Proprietary business logic exposure
- Potential exposure of algorithm implementations
- Risk of intellectual property theft
- Could reveal system architecture and data flow patterns

**Current Mitigation:**
- Access control via `ensureWsWorkunitAccess(context, wuid.str(), SecAccess_Read)`
- User must have read access to the specific workunit
- Workunit ownership model (users can only access their own workunits or shared ones)

**Code Reference:**
```cpp
// esp/services/ws_workunits/ws_workunitsHelpers.cpp:800-811
if (flags & WUINFO_IncludeECL)
{
    SCMStringBuffer queryText;
    query->getQueryShortText(queryText);
    if (queryText.length())
    {
        if((flags & WUINFO_TruncateEclTo64k) && (queryText.length() > 64000))
            queryText.setLen(queryText.str(), 64000);
        IEspECLQuery* q=&info.updateQuery();
        q->setText(queryText.str());
    }
}
```

#### 1.2 Debug Values Exposure (MEDIUM-HIGH RISK)
**Location:** `esp/scm/ws_workunits_struct.ecm` - DebugValue structure  
**Fields:** `DebugValue.Name`, `DebugValue.Value`

**Issue:**
Debug values are name-value pairs that can contain arbitrary data set during workunit execution. These can potentially contain sensitive information such as:
- Internal variable values
- Temporary credentials or tokens
- Database connection strings
- API endpoints and configuration
- Debugging information revealing system internals

**Current Mitigation:**
- Controlled by `IncludeDebugValues` flag (default: true)
- Same access control as main workunit (SecAccess_Read)

**Code Reference:**
```cpp
// esp/services/ws_workunits/ws_workunitsHelpers.cpp:987-1018
void WsWuInfo::getDebugValues(IEspECLWorkunit &info, unsigned long flags)
{
    if (!(flags & WUINFO_IncludeDebugValues))
        return;
    
    IArrayOf<IEspDebugValue> dv;
    Owned<IStringIterator> debugs(&cw->getDebugValues());
    ForEach(*debugs)
    {
        SCMStringBuffer name, val;
        debugs->str(name);
        cw->getDebugValue(name.str(),val);
        
        Owned<IEspDebugValue> t= createDebugValue("","");
        t->setName(name.str());
        t->setValue(val.str());
        dv.append(*t.getLink());
    }
    info.setDebugValues(dv);
}
```

#### 1.3 Application Values Exposure (MEDIUM RISK)
**Location:** `esp/scm/ws_workunits_struct.ecm` - ApplicationValue structure  
**Fields:** `ApplicationValue.Application`, `ApplicationValue.Name`, `ApplicationValue.Value`

**Issue:**
Application values store application-specific metadata and configuration. Similar to debug values, these can contain sensitive information:
- Application configuration settings
- Integration credentials
- Feature flags
- Custom metadata that might reveal business logic

**Current Mitigation:**
- Controlled by `IncludeApplicationValues` flag (default: true)
- Same access control as main workunit (SecAccess_Read)

**Code Reference:**
```cpp
// esp/services/ws_workunits/ws_workunitsHelpers.cpp:957-985
void WsWuInfo::getApplicationValues(IEspECLWorkunit &info, unsigned long flags)
{
    if (!(flags & WUINFO_IncludeApplicationValues))
        return;
    
    IArrayOf<IEspApplicationValue> av;
    Owned<IConstWUAppValueIterator> app(&cw->getApplicationValues());
    ForEach(*app)
    {
        IConstWUAppValue& val=app->query();
        Owned<IEspApplicationValue> t= createApplicationValue("","");
        t->setApplication(val.queryApplication());
        t->setName(val.queryName());
        t->setValue(val.queryValue());
        av.append(*t.getLink());
    }
    info.setApplicationValues(av);
}
```

#### 1.4 Exception and Error Messages (MEDIUM RISK)
**Location:** `esp/scm/ws_workunits_struct.ecm` - ECLException structure

**Issue:**
Exception messages can expose:
- Stack traces revealing code paths
- File system paths
- Internal error codes
- Database schema information
- System configuration details

**Current Mitigation:**
- Controlled by `IncludeExceptions` flag (default: true)
- Standard access control applies

#### 1.5 System Topology Information (LOW-MEDIUM RISK)
**Fields Exposed:**
- `Cluster` - Cluster name
- `AllowedClusters` - List of clusters workunit can run on
- `ThorLogList` - Thor process logs information
- `ServiceNames` - Service names involved in execution

**Issue:**
Reveals internal system architecture and topology which could aid in:
- Understanding system capacity and layout
- Planning targeted attacks
- Identifying potential weak points

**Current Mitigation:**
- Access control in place
- Information only useful to authenticated users with workunit access

#### 1.6 Variables and Results (MEDIUM RISK)
**Fields:** `Variables`, `Results`

**Issue:**
Variables and results can contain:
- Actual data processed by the workunit
- Sample data or outputs
- Intermediate calculation results
- Potentially sensitive business data

**Current Mitigation:**
- Controlled by `IncludeVariables` and `IncludeResults` flags
- Standard access control applies
- Result schemas can be suppressed with `SuppressResultSchemas`

### 2. Authentication and Authorization

#### 2.1 Access Control Implementation (GOOD)
**Implementation:** `ensureWsWorkunitAccess()`

The endpoint implements proper access control:

```cpp
// esp/services/ws_workunits/ws_workunitsService.cpp:1578
ensureWsWorkunitAccess(context, wuid.str(), SecAccess_Read);
```

**Access Control Logic:**
```cpp
// esp/services/ws_workunits/ws_workunitsHelpers.cpp:144-151
void ensureWsWorkunitAccess(IEspContext& context, const char* wuid, SecAccessFlags minAccess)
{
    Owned<IWorkUnitFactory> wf = getWorkUnitFactory(context.querySecManager(), context.queryUser());
    Owned<IConstWorkUnit> cw = wf->openWorkUnit(wuid);
    if (!cw)
        throw MakeStringException(ECLWATCH_CANNOT_OPEN_WORKUNIT, 
            "Failed to open workunit %s when ensuring workunit access", wuid);
    ensureWsWorkunitAccess(context, *cw, minAccess);
}
```

**Assessment:**
✅ Proper authentication required  
✅ Authorization checks based on workunit ownership  
✅ Uses security manager integration  
✅ Throws appropriate exceptions on access denial  

#### 2.2 Deferred Authorization (GOOD)
**Configuration:** `auth_feature("DEFERRED")`

The service declaration includes:
```cpp
// esp/scm/ws_workunits.ecm:27
ESPservice [
    auth_feature("DEFERRED"), //This declares that the method logic handles feature level authorization
    version("2.02"), default_client_version("2.02"), cache_group("ESPWsWUs"),
    noforms,exceptions_inline("./smc_xslt/exceptions.xslt"),use_method_name] WsWorkunits
```

This indicates that authorization is handled within the method logic rather than at the service level, allowing for fine-grained control.

### 3. Caching Considerations

**Configuration:** `cache_seconds(30)`

```cpp
// esp/scm/ws_workunits.ecm:33
ESPmethod [cache_seconds(30), resp_xsl_default("/esp/xslt/wuid.xslt")] WUInfo(WUInfoRequest, WUInfoResponse);
```

**Issue:**
Responses are cached for 30 seconds, which means:
- Sensitive data remains in cache temporarily
- Multiple requests within 30 seconds return cached data
- Cache invalidation might not occur immediately after access revocation

**Risk Level:** LOW (acceptable for most use cases)

**Recommendation:**
- Consider reducing cache time for high-security environments
- Implement cache invalidation on permission changes
- Ensure cache keys include user identity to prevent cross-user cache hits

### 4. Transport Security

**WSDL URL:** `https://play.hpccsystems.com:18010/WsWorkunits/WUInfo?ver_=2.02&wsdl`

**Assessment:**
✅ Uses HTTPS (port 18010)  
✅ Encrypted transport layer  

**Recommendation:**
- Ensure strong TLS configuration (TLS 1.2+)
- Verify certificate validation
- Implement HSTS headers

### 5. Rate Limiting and Abuse Prevention

**Issue:**
No apparent rate limiting mechanism in the WSDL or service definition.

**Risk:**
- Potential for information harvesting attacks
- Denial of service through excessive requests
- Brute force enumeration of workunit IDs

**Recommendation:**
- Implement rate limiting per user/IP
- Add request throttling for expensive operations
- Monitor for suspicious access patterns
- Consider implementing CAPTCHA for repeated failures

### 6. WSDL Information Disclosure

**Issue:**
The WSDL itself reveals:
- Complete API structure
- All available methods and parameters
- Data types and structures
- Version information
- Internal naming conventions

**Risk Level:** LOW (expected behavior for SOAP services)

**Note:** This is standard WSDL behavior and necessary for client integration. However, consider:
- Limiting WSDL access to authenticated users only
- Providing minimal WSDL for public consumption
- Documenting API externally with sanitized examples

## Implemented Security Enhancements

### 1. Sensitive Value Redaction (IMPLEMENTED)
**Date:** 2025-11-05  
**Files Modified:** `esp/services/ws_workunits/ws_workunitsHelpers.cpp`

**Description:**
Implemented automatic detection and redaction of potentially sensitive values in debug and application value fields. The system now scans field names for common credential-related keywords and redacts their values before exposing them via the WUInfo endpoint.

**Implementation Details:**
- Added `isSensitiveValueName()` function that detects field names containing sensitive keywords
- Added `sanitizeSensitiveValue()` function that redacts values when sensitive names are detected
- Modified `getDebugValues()` to sanitize debug values before returning them
- Modified `getApplicationValues()` to sanitize application values before returning them
- Added security logging when values are redacted for audit purposes

**Keywords Detected:**
- password, passwd, pwd
- secret, token, apikey, api_key, api-key
- credential, auth, authorization
- private, priv, key
- connection, connectionstring, conn_str
- bearer, oauth

**Redaction Behavior:**
When a field name contains any of the above keywords (case-insensitive), its value is replaced with `***REDACTED***` and a warning is logged with the workunit ID.

**Example:**
```
Before:
  DebugValue: name="api_token", value="sk_live_51abc123..."
  
After:
  DebugValue: name="api_token", value="***REDACTED***"
  Log: "Security: Redacted potentially sensitive value 'api_token' in workunit W20231105-123456"
```

**Impact:**
- ✅ Prevents accidental credential exposure via debug/application values
- ✅ Provides audit trail of redacted values
- ✅ Backward compatible - only affects values with sensitive-sounding names
- ✅ No performance impact - simple string matching operation

**Testing:**
To test the implementation:
1. Create a workunit with debug values containing sensitive keywords
2. Query WUInfo for that workunit
3. Verify that sensitive values are redacted
4. Check logs for security warnings

**Future Enhancements:**
- Add configuration option to customize sensitive keywords list
- Add option to completely exclude sensitive fields instead of redacting
- Implement pattern-based detection (regex) for credential values
- Add metrics for tracking redaction frequency

---

## Recommendations

### High Priority

1. **✅ IMPLEMENTED - Sensitive Data Filtering**
   - ✅ Implemented keyword-based detection for credentials, tokens, and passwords
   - ✅ Added automatic redaction for potentially sensitive debug and application values
   - ⚠️ Future: Add configuration option to customize sensitive keywords list
   - ⚠️ Future: Implement pattern-based (regex) detection for credential values

2. **Enhance Access Logging**
   - Log all WUInfo access requests with user identity and WUID
   - Implement anomaly detection for unusual access patterns
   - Alert on bulk data access attempts

3. **Add Rate Limiting**
   - Implement per-user rate limits (e.g., 100 requests per minute)
   - Add IP-based rate limiting for additional protection
   - Provide admin controls to adjust limits

### Medium Priority

4. **Review Default Include Flags**
   - Consider changing defaults for sensitive fields to `false`:
     - `IncludeDebugValues` → false
     - `IncludeApplicationValues` → false
     - `IncludeECL` → false (or require explicit permission)
   - Document security implications of each flag

5. **Implement Field-Level Access Control**
   - Add granular permissions for sensitive fields
   - Allow administrators to configure which fields are accessible
   - Separate read permissions for code vs. execution data

6. **Reduce Cache Duration**
   - Reduce from 30 seconds to 10 seconds for sensitive endpoints
   - Implement cache invalidation on permission changes
   - Add no-cache option for high-security requests

### Low Priority

7. **Enhanced Documentation**
   - Document security model in API documentation
   - Provide security best practices for API consumers
   - Create security guidelines for ECL developers

8. **Add Response Filtering Options**
   - Allow clients to specify exactly which fields they need
   - Implement field whitelisting instead of blacklisting
   - Reduce over-fetching of data

9. **Implement Audit Trail**
   - Create comprehensive audit logs for sensitive operations
   - Include data access audit trail
   - Provide audit report generation for compliance

## Compliance Considerations

### GDPR/Data Privacy
- **Personal Data:** Workunit owner information is exposed
- **Right to Access:** Supported via the endpoint
- **Right to Erasure:** Verify workunit deletion removes all cached data
- **Data Minimization:** Consider implementing minimal data response mode

### SOC 2 / Security Standards
- **Access Controls:** ✅ Implemented
- **Audit Logging:** ⚠️ Needs enhancement
- **Encryption:** ✅ HTTPS in use
- **Monitoring:** ⚠️ Limited visibility

## Testing Recommendations

1. **Security Testing**
   - Penetration testing focused on information disclosure
   - Access control testing (vertical and horizontal privilege escalation)
   - Input validation testing for WUID parameter
   - Rate limiting verification

2. **Automated Security Scanning**
   - SAST scanning of service implementation
   - Dependency vulnerability scanning
   - WSDL security analysis tools

3. **Access Control Tests**
   - Verify users cannot access other users' workunits
   - Test permission inheritance and delegation
   - Validate access revocation effectiveness

## Conclusion

The WUInfo endpoint implements **adequate base-level security** with proper authentication and authorization controls. However, it exposes a **significant amount of potentially sensitive information** that could be leveraged by malicious actors.

**Overall Risk Assessment:** MEDIUM

**Key Risks:**
1. ECL source code exposure (business logic)
2. Debug/application values (potential credentials)
3. System topology information
4. Lack of rate limiting

**Strengths:**
1. Proper access control implementation
2. Deferred authorization for fine-grained control
3. Encrypted transport (HTTPS)
4. Comprehensive permission model

**Recommendation:** Implement high-priority recommendations before deploying in production environments handling sensitive data. Regular security audits should be conducted to ensure ongoing protection.

## References

- ECM File: `esp/scm/ws_workunits.ecm`
- Implementation: `esp/services/ws_workunits/ws_workunitsService.cpp`
- Helper Functions: `esp/services/ws_workunits/ws_workunitsHelpers.cpp`
- Structure Definitions: `esp/scm/ws_workunits_struct.ecm`
- Request/Response: `esp/scm/ws_workunits_req_resp.ecm`

## Appendix A: Exposed Data Fields

### ECLWorkunit Structure (Complete Field List)
```
- Wuid (Workunit ID)
- Owner
- Cluster, RoxieCluster
- Jobname
- Queue
- StateID, State, StateEx
- Description
- Protected, Active
- Action, ActionEx
- DateTimeScheduled
- PriorityClass, PriorityLevel
- Scope, Snapshot
- ResultLimit
- Archived
- IsPausing, ThorLCR
- EventSchedule
- TotalClusterTime
- AbortBy, AbortTime
- Query (ECLQuery) ⚠️
  - Text ⚠️ (ECL source code)
  - Cpp ⚠️ (Generated C++ code)
  - ResTxt ⚠️
  - Dll ⚠️
  - ThorLog ⚠️
  - QueryMainDefinition ⚠️
- Helpers (ECLHelpFile array) ⚠️
- Exceptions (ECLException array) ⚠️
- Graphs (ECLGraph array)
- SourceFiles (ECLSourceFile array) ⚠️
- Results (ECLResult array) ⚠️
- Variables (ECLResult array) ⚠️
- Timers (ECLTimer array)
- DebugValues (DebugValue array) ⚠️ HIGH RISK
- ApplicationValues (ApplicationValue array) ⚠️ HIGH RISK
- Workflows (ECLWorkflow array)
- TimingData (ECLTimingData array)
- AllowedClusters ⚠️
- Error/Warning/Info/Alert Counts
- Graph/SourceFile/Result/Variable/Timer Counts
- XmlParams ⚠️
- AccessFlag
- ClusterFlag
- Various description fields
- HasArchiveQuery
- ThorLogList ⚠️
- ResourceURLs
- ServiceNames ⚠️
- ExecuteCost, FileAccessCost, CompileCost
- NoAccess
- ECLWUProcessList ⚠️
```

⚠️ = Potentially sensitive information

## Appendix B: Attack Scenarios

### Scenario 1: Intellectual Property Theft
**Attacker Goal:** Steal proprietary ECL code  
**Attack Vector:** 
1. Compromised user account with workunit access
2. Enumerate workunits via WUQuery
3. Fetch full ECL source via WUInfo for each WUID
4. Exfiltrate business logic and algorithms

**Mitigation:** 
- Enhanced access logging and anomaly detection
- Rate limiting
- Alert on bulk downloads

### Scenario 2: Credential Harvesting
**Attacker Goal:** Extract credentials from debug values  
**Attack Vector:**
1. Developer accidentally logs credentials to debug values
2. Attacker with read access retrieves WUInfo
3. Extracts credentials from DebugValues or ApplicationValues
4. Uses credentials for lateral movement

**Mitigation:**
- Implement credential detection and redaction
- Developer training on secure debugging
- Automated scanning for credential patterns

### Scenario 3: System Reconnaissance
**Attacker Goal:** Map internal infrastructure  
**Attack Vector:**
1. Query multiple workunits
2. Aggregate cluster names, service names, topology info
3. Identify potential targets for further attacks
4. Understand system capacity and weaknesses

**Mitigation:**
- Limit topology information exposure
- Implement honeypot workunits
- Monitor for reconnaissance patterns

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-05  
**Author:** Security Analysis - AI Assistant  
**Classification:** Internal Use
