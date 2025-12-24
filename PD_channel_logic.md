# Patient Discovery Channel Logic

## PD_Request_In

### Source Transformer (JavaScript)
```javascript
// Defensive HTTP JSON intake for /pd/trigger/
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
var validationError = '';

try {
    parsed = JSON.parse(rawBody);
} catch (e1) {
    logger.error('PD_Request_In: Invalid JSON payload received.');
    validationError = 'Invalid JSON payload';
}

if (parsed != null) {
    try {
        if (parsed.request_id != null) {
            requestId = String(parsed.request_id);
        }
        if (parsed.patient_id != null) {
            patientId = String(parsed.patient_id);
        }
    } catch (e2) {
        logger.error('PD_Request_In: Error extracting identifiers from payload.');
        validationError = 'Unable to extract identifiers';
    }
}

if (validationError == '' && (requestId === '' || patientId === '')) {
    validationError = 'Missing required fields: request_id and patient_id';
}

channelMap.put('requestId', requestId);
channelMap.put('patientId', patientId);

if (validationError !== '') {
    responseStatusCode = 400;
    responseMessage = validationError;
    logger.warn('PD_Request_In: Rejected request. Reason=' + validationError);
    return rawBody;
}

var normalized = '{"request_id":"' + requestId + '","patient_id":"' + patientId + '"}';
msg = normalized;

try {
    globalMap.put('pd_status_' + requestId, 'PENDING');
} catch (e3) {
    logger.warn('PD_Request_In: Unable to cache status for request ' + requestId);
}

logger.info('PD_Request_In: Accepted PD trigger requestId=' + requestId + ' patientId=' + patientId);
responseStatusCode = 202;
responseMessage = '';
return msg;
```

### Response (JavaScript)
```javascript
// Ensure HTTP 202 with empty body
var status = responseStatusCode;
if (status == null) {
    status = 202;
}
responseStatusCode = status;

var body = responseMessage;
if (body == null) {
    body = '';
}
responseHeaders.put('Content-Type', 'application/json');
return body;
```

## PD_Mock_Responder

### Source Transformer (JavaScript)
```javascript
// Build HL7 v3 PRPA_IN201306UV02 response
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

msg = xml;

try {
    globalMap.put('pd_status_' + requestId, 'COMPLETE');
} catch (e2) {
    logger.warn('PD_Mock_Responder: Unable to update status for request ' + requestId);
}

logger.info('PD_Mock_Responder: Responding with PRPA_IN201306UV02 for requestId=' + requestId + ' patientId=' + patientId);
responseStatusCode = 202;
responseHeaders.put('Content-Type', 'application/xml');
responseMessage = xml;
return msg;
```

### Response (JavaScript)
```javascript
// Echo the XML response back to the HTTP caller
var status = responseStatusCode;
if (status == null) {
    status = 202;
}
responseStatusCode = status;

var body = responseMessage;
if (body == null) {
    body = '';
}
responseHeaders.put('Content-Type', 'application/xml');
return body;
```

## PD_Status_Get

### Source Transformer (JavaScript)
```javascript
// HTTP GET /pd/status/{request_id}
var requestUri = '';
var sourceMap = connectorMessage.getSourceMap();
if (sourceMap != null) {
    try {
        var httpRequest = sourceMap.get('httpRequest');
        if (httpRequest != null) {
            requestUri = String(httpRequest.getRequestURI());
        }
    } catch (e) {
        logger.warn('PD_Status_Get: Unable to read request URI from httpRequest.');
    }
}

if (requestUri == null) {
    requestUri = '';
}

var requestId = '';
try {
    var parts = requestUri.split('/');
    if (parts != null && parts.length > 0) {
        requestId = parts[parts.length - 1];
    }
} catch (e1) {
    requestId = '';
}

if (requestId === '') {
    responseStatusCode = 404;
    responseHeaders.put('Content-Type', 'application/json');
    var missingBody = '{"error":"request_id not found"}';
    responseMessage = missingBody;
    logger.warn('PD_Status_Get: Missing request_id in path.');
    return message;
}

var statusKey = 'pd_status_' + requestId;
var statusValue = null;
try {
    statusValue = globalMap.get(statusKey);
} catch (e2) {
    statusValue = null;
}

if (statusValue == null) {
    responseStatusCode = 404;
    responseHeaders.put('Content-Type', 'application/json');
    var notFound = '{"error":"request_id not found"}';
    responseMessage = notFound;
    logger.warn('PD_Status_Get: No status found for request_id=' + requestId);
    return message;
}

var isoTimestamp = DateUtil.getCurrentDate('yyyy-MM-dd\'T\'HH:mm:ssXXX');
var responseJson = '{"request_id":"' + requestId + '","status":"' + statusValue + '","timestamp":"' + isoTimestamp + '"}';

responseStatusCode = 200;
responseHeaders.put('Content-Type', 'application/json');
responseMessage = responseJson;
logger.info('PD_Status_Get: Returning status for request_id=' + requestId + ' status=' + statusValue);
return message;
```

### Response (JavaScript)
```javascript
// Return JSON body with appropriate status
var status = responseStatusCode;
if (status == null) {
    status = 500;
}
responseStatusCode = status;

var body = responseMessage;
if (body == null) {
    body = '';
}
responseHeaders.put('Content-Type', 'application/json');
return body;
```
