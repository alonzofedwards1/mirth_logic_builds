# Patient Discovery Channel Logic

## PD_Request_In

### Source Transformer (JavaScript)
```javascript
// HTTP POST /pd/trigger (match your context path exactly; mismatched trailing slash can yield 405) - Source Transformer returns the HTTP response body directly (no Response tab in 4.5.x)
var rawBody = '';
try {
    rawBody = String(connectorMessage.getRawData());
} catch (e) {
    rawBody = '';
}

if (rawBody == null) {
    rawBody = '';
}

var parsed = null;
var requestId = '';
var patientId = '';
var errorText = '';

if (rawBody === '') {
    errorText = 'Empty request body';
} else {
    try {
        parsed = JSON.parse(rawBody);
    } catch (parseError) {
        errorText = 'Invalid JSON payload';
    }
}

if (parsed != null) {
    try {
        if (parsed.request_id != null) {
            requestId = String(parsed.request_id);
        }
        if (parsed.patient_id != null) {
            patientId = String(parsed.patient_id);
        }
    } catch (extractError) {
        errorText = 'Unable to read request fields';
    }
}

if (errorText === '' && (requestId === '' || patientId === '')) {
    errorText = 'Missing required fields: request_id and patient_id';
}

channelMap.put('requestId', requestId);
channelMap.put('patientId', patientId);

if (errorText !== '') {
    responseStatusCode = 400; // HTTP status set here; return value below is the HTTP body
    responseHeaders.put('Content-Type', 'text/plain');
    logger.warn('PD_Request_In: Rejected request. Reason=' + errorText);
    return errorText; // Returned string becomes the HTTP response body
}

var normalized = '{"request_id":"' + requestId + '","patient_id":"' + patientId + '"}';
msg = normalized; // Forward normalized JSON to downstream channel

try {
    globalMap.put('pd_status_' + requestId, 'PENDING');
} catch (statusError) {
    logger.warn('PD_Request_In: Unable to cache status for request ' + requestId);
}

logger.info('PD_Request_In: Accepted PD trigger requestId=' + requestId + ' patientId=' + patientId);
responseStatusCode = 202; // HTTP status set here; empty return below yields empty HTTP body
responseHeaders.put('Content-Type', 'application/json');
return ''; // Empty string -> empty HTTP response body (single response path in Source Transformer)
```

## PD_Mock_Responder

### Source Transformer (JavaScript)
```javascript
// Channel Reader: builds and returns HL7 v3 XML directly from the Source Transformer (no Response tab)
var inbound = '';
try {
    inbound = String(message);
} catch (e) {
    inbound = '';
}

var requestId = '';
var patientId = '';

if (inbound != null && inbound !== '') {
    try {
        var parsed = JSON.parse(inbound);
        if (parsed.request_id != null) {
            requestId = String(parsed.request_id);
        }
        if (parsed.patient_id != null) {
            patientId = String(parsed.patient_id);
        }
    } catch (e1) {
        logger.warn('PD_Mock_Responder: Payload not JSON; proceeding with defaults.');
    }
}

if (requestId === '') {
    requestId = UUIDGenerator.getUUID();
}
if (patientId === '') {
    patientId = 'UNKNOWN';
}

var messageId = UUIDGenerator.getUUID();
var creationTime = DateUtil.getCurrentDate('yyyyMMddHHmmss');

var xml = '';
xml += '<?xml version="1.0" encoding="UTF-8"?>';
xml += '<PRPA_IN201306UV02 xmlns="urn:hl7-org:v3" ITSVersion="XML_1.0">';
xml += '<id root="' + messageId + '"/>';
xml += '<creationTime value="' + creationTime + '"/>';
xml += '<interactionId root="2.16.840.1.113883.1.6" extension="PRPA_IN201306UV02"/>';
xml += '<processingCode code="P"/>';
xml += '<processingModeCode code="T"/>';
xml += '<acceptAckCode code="NE"/>';
xml += '<acknowledgement><typeCode code="AA"/><targetMessage><id root="' + requestId + '"/></targetMessage></acknowledgement>';
xml += '<controlActProcess classCode="CACT" moodCode="EVN">';
xml += '<code code="PRPA_TE201306UV02" codeSystem="2.16.840.1.113883.1.6"/>';
xml += '<queryAck><queryId root="' + requestId + '"/><queryResponseCode code="NF"/><resultCurrentQuantity value="0"/><resultTotalQuantity value="0"/><resultRemainingQuantity value="0"/></queryAck>';
xml += '<queryByParameter><queryId root="' + requestId + '"/><statusCode code="completed"/><parameterList>';
xml += '<patientIdentifier><value root="' + patientId + '"/></patientIdentifier>';
xml += '</parameterList></queryByParameter>';
xml += '</controlActProcess>';
xml += '</PRPA_IN201306UV02>';

try {
    globalMap.put('pd_status_' + requestId, 'COMPLETE');
} catch (e2) {
    logger.warn('PD_Mock_Responder: Unable to update status for request ' + requestId);
}

logger.info('PD_Mock_Responder: Responding with PRPA_IN201306UV02 for requestId=' + requestId + ' patientId=' + patientId);
responseStatusCode = 202; // HTTP status set here
responseHeaders.put('Content-Type', 'application/xml');
return xml; // Returned XML is the HTTP response body (single response path in Source Transformer)
```

## PD_Status_Get

### Source Transformer (JavaScript)
```javascript
// HTTP GET /pd/status/{request_id} - Source Transformer returns JSON directly (no Response tab)
var requestUri = '';
var queryString = '';
var sourceMap = connectorMessage.getSourceMap();
if (sourceMap != null) {
    try {
        var httpRequest = sourceMap.get('httpRequest');
        if (httpRequest != null) {
            try {
                requestUri = String(httpRequest.getRequestURI());
            } catch (e1) {
                requestUri = '';
            }
            try {
                queryString = String(httpRequest.getQueryString());
            } catch (e2) {
                queryString = '';
            }
        }
    } catch (e) {
        logger.warn('PD_Status_Get: Unable to read httpRequest details.');
    }
}

if (requestUri == null) {
    requestUri = '';
}
if (queryString == null) {
    queryString = '';
}

var requestId = '';

// Prefer explicit query parameter request_id if present
if (queryString !== '') {
    try {
        var pairs = queryString.split('&');
        for (var i = 0; i < pairs.length; i++) {
            var kv = pairs[i].split('=');
            if (kv != null && kv.length >= 2) {
                var key = kv[0];
                var val = kv[1];
                if (key === 'request_id') {
                    requestId = val;
                    break;
                }
            }
        }
    } catch (qpError) {
        requestId = '';
    }
}

// Fallback to last path segment if query param not found
if (requestId === '') {
    try {
        var parts = requestUri.split('/');
        if (parts != null && parts.length > 0) {
            requestId = parts[parts.length - 1];
        }
    } catch (e3) {
        requestId = '';
    }
}

if (requestId === '') {
    responseStatusCode = 404; // HTTP status set here; return value below is the HTTP body
    responseHeaders.put('Content-Type', 'application/json');
    var missingBody = '{"error":"request_id not found"}';
    logger.warn('PD_Status_Get: Missing request_id in request.');
    return missingBody; // Returned string becomes the HTTP response body
}

var statusKey = 'pd_status_' + requestId;
var statusValue = null;
try {
    statusValue = globalMap.get(statusKey);
} catch (e4) {
    statusValue = null;
}

if (statusValue == null) {
    responseStatusCode = 404; // HTTP status set here; return value below is the HTTP body
    responseHeaders.put('Content-Type', 'application/json');
    var notFound = '{"error":"request_id not found"}';
    logger.warn('PD_Status_Get: No status found for request_id=' + requestId);
    return notFound; // Returned string becomes the HTTP response body
}

var isoTimestamp = DateUtil.getCurrentDate('yyyy-MM-dd\'T\'HH:mm:ssXXX');
var responseJson = '{"request_id":"' + requestId + '","status":"' + statusValue + '","timestamp":"' + isoTimestamp + '"}';

responseStatusCode = 200; // HTTP status set here
responseHeaders.put('Content-Type', 'application/json');
logger.info('PD_Status_Get: Returning status for request_id=' + requestId + ' status=' + statusValue);
return responseJson; // Returned JSON is the HTTP response body (single response path in Source Transformer)
```
