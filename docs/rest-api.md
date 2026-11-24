# REST API

All requests and responses use JSON over HTTPS.

All requests require Basic authentication in the `Authorization` header. See [Authentication](authentication.md) for details.

The endpoints are relative to the API base URL. The base URL is specific to each client and is shared together with the credentials. The examples on this page use `https://transit-api.example.com/v1` as an illustrative base URL.

## Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | [`/geofences`](#get-geofences) | Retrieves the list of topologies and their bounding boxes |
| `POST` | [`/detections`](#post-detections) | Accepts detection data |
| `GET` | [`/history/{roadUserId}`](#get-historyroaduserid) | Retrieves the processing history of a road user |

## Compression

Responses (in practice `GET /geofences` and larger `GET /history/{roadUserId}` responses) are `gzip` compressed when the request includes the `Accept-Encoding: gzip` header and the response is at least 1 KB. A compressed response includes the `Content-Encoding: gzip` header.

!!! warning "Check every response"
    Whether a response is compressed differs per request. Clients that send `Accept-Encoding: gzip` must check every response for the `Content-Encoding: gzip` header.

---

## GET /geofences

Retrieves the list of topologies and their bounding boxes, allowing information providers to determine the relevance of their detection data to a specific topology.

!!! note "Bounding boxes are a coarse filter"
    The bounding box only gives a rough indication of the area of a topology. The Transit API filters the received detections more precisely, based on the bicycle lanes of the topology. Detections that do not overlap a bicycle lane are counted as `detectionsNoOverlap` in the [history](#get-historyroaduserid) of the road user.

### Request

```http
GET <API base URL>/geofences HTTP/1.1
Host: <hostname>
Authorization: Basic <base64-encoding of username:password>
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
[
  {
    "id": "string",
    "description": "string",
    "referencePoints": [
      {
        "latitude": 0,
        "longitude": 0
      }
    ],
    "boundingBox": {
      "topLeft": {
        "latitude": 0,
        "longitude": 0
      },
      "bottomRight": {
        "latitude": 0,
        "longitude": 0
      }
    }
  }
]
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Identifier for the topology |
| `description` | string | No | Description for the topology |
| `referencePoints` | array of objects | Yes | List of intersection reference points; contains at least one reference point |
| `referencePoints.latitude` | double | Yes | Latitude of a reference point |
| `referencePoints.longitude` | double | Yes | Longitude of a reference point |
| `boundingBox` | object | Yes | Bounding box for the topology |
| `boundingBox.topLeft` | object | Yes | Maximum latitude and minimum longitude (northwest corner) |
| `boundingBox.topLeft.latitude` | double | Yes | Latitude of the top-left corner of the topology |
| `boundingBox.topLeft.longitude` | double | Yes | Longitude of the top-left corner of the topology |
| `boundingBox.bottomRight` | object | Yes | Minimum latitude and maximum longitude (southeast corner) |
| `boundingBox.bottomRight.latitude` | double | Yes | Latitude of the bottom-right corner of the topology |
| `boundingBox.bottomRight.longitude` | double | Yes | Longitude of the bottom-right corner of the topology |

### Example

#### Request

```http
GET /v1/geofences HTTP/1.1
Host: transit-api.example.com
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
[
  {
    "id": "79230488",
    "description": "Geofence for subject 79230488",
    "referencePoints": [
      {
        "latitude": 52.3391458,
        "longitude": 5.2103103
      },
      {
        "latitude": 52.3414051,
        "longitude": 5.2075065
      },
      {
        "latitude": 52.340596,
        "longitude": 5.2071333
      },
      {
        "latitude": 52.3397128,
        "longitude": 5.2067699
      },
      {
        "latitude": 52.3392504,
        "longitude": 5.2088768
      }
    ],
    "boundingBox": {
      "topLeft": {
        "latitude": 52.3423745,
        "longitude": 5.2037385
      },
      "bottomRight": {
        "latitude": 52.3368683,
        "longitude": 5.2123639
      }
    }
  }
]
```

---

## POST /detections

Accepts detection data and keep-alive payloads (an empty array).

!!! info "Keep-alive"
    Send a payload with an empty array (`[]`) every 30 seconds if there are no valid detections to post. The 30-second interval applies per client connection, not per road user: any posted detections reset it.

    The keep-alive is used for monitoring only: it lets Monotch detect whether a client is still alive. A missed keep-alive has no other effect.

!!! warning "Request size limit"
    The request body must not exceed 60 KB.

!!! warning "Send detections immediately"
    A detection is only processed if it arrives within 2 seconds of its `detectionTime`. Detections that are older, or that lie in the future, are rejected. Send each detection as soon as it is made, do not buffer detections to send them later, and keep the clock of the sending system synchronised (for example with NTP).

### Request

```http
POST <API base URL>/detections HTTP/1.1
Host: <hostname>
Authorization: Basic <base64-encoding of username:password>
Content-Type: application/json
```

```json
[
  {
    "roadUserId": "string",
    "detectionTime": "2025-01-29T12:00:00Z",
    "detection": {
      "latitude": 0,
      "longitude": 0,
      "speed": 0,
      "heading": 0,
      "accuracy": 0
    },
    "route": [
      {
        "latitude": 0,
        "longitude": 0
      }
    ]
  }
]
```

| Field | Type | Required | Description |
|---|---|---|---|
| `roadUserId` | string | Yes | Unique identifier of a road user (cyclist or bike) |
| `detectionTime` | string | Yes | Time of the detection, ISO 8601 formatted in UTC (e.g. `2025-01-29T12:00:00Z`); must be within 2 seconds of the time the request is received |
| `detection` | object | Yes | Location and movement of the road user at the time of the detection |
| `detection.latitude` | double | Yes | Latitude of the detected road user's location |
| `detection.longitude` | double | Yes | Longitude of the detected road user's location |
| `detection.speed` | double | No (nullable) | Speed of the road user in meters per second; must not be negative |
| `detection.heading` | double | No (nullable) | Direction the road user is moving in (0-360°, 0° is north, clockwise) |
| `detection.accuracy` | integer | Yes | Accuracy of the location detection in meters |
| `route` | array of objects | No (nullable) | Array of waypoints. No route (`null` or an empty array) means that the road user has reached their destination |
| `route.latitude` | double | Yes | Latitude of a waypoint along the route |
| `route.longitude` | double | Yes | Longitude of a waypoint along the route |

### Validation

A detection is rejected when:

- `roadUserId` is missing or empty
- `detectionTime`, `detection`, `detection.latitude` or `detection.longitude` is missing
- `detection.speed` is negative
- `detectionTime` is more than 2 seconds before or after the time the request is received

Valid detections are matched against the bicycle lanes of the topologies. Only detections on a bicycle lane result in messages to UDAP. See [`GET /history/{roadUserId}`](#get-historyroaduserid) to check how the detections of a road user were processed.

### Response

```http
HTTP/1.1 200 OK
```

The response has no body. Every request is accepted with `200 OK`: detections are validated after the request has been accepted, so a `200 OK` does not mean that the detections are valid. Use [`GET /history/{roadUserId}`](#get-historyroaduserid) to see whether detections were rejected (`detectionsRejected`) or did not overlap a bicycle lane (`detectionsNoOverlap`).

### Example

#### Request

```http
POST /v1/detections HTTP/1.1
Host: transit-api.example.com
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
Content-Type: application/json
```

```json
[
  {
    "roadUserId": "B123",
    "detectionTime": "2025-01-29T12:00:00Z",
    "detection": {
      "latitude": 52.37757,
      "longitude": 4.89425,
      "speed": 8.0,
      "heading": 90.0,
      "accuracy": 3
    },
    "route": [
      {
        "latitude": 52.37750,
        "longitude": 4.89433
      },
      {
        "latitude": 52.37740,
        "longitude": 4.89443
      },
      {
        "latitude": 52.37732,
        "longitude": 4.89433
      }
    ]
  }
]
```

#### Keep-alive request

```json
[]
```

---

## GET /history/{roadUserId}

Retrieves the processing history of a road user: how many of its detections were received and accepted, and which traffic light controllers it passed, including the priority requests and CAMs that the Transit API published to UDAP on its behalf.

!!! info "History is built periodically"
    The history is rebuilt every minute. Changes take about 3 minutes to become visible. Until the first history of a road user has been built, the endpoint returns `404 Not Found`.

The history covers the activity of the road user in the 24 hours before the history was built.

### Request

```http
GET <API base URL>/history/{roadUserId} HTTP/1.1
Host: <hostname>
Authorization: Basic <base64-encoding of username:password>
```

| Parameter | Description |
|---|---|
| `roadUserId` | The `roadUserId` used in [`POST /detections`](#post-detections), URL-encoded |

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "asOf": "string",
  "window": {
    "from": "string",
    "to": "string"
  },
  "summary": {
    "detectionsReceived": 0,
    "detectionsRejected": 0,
    "detectionsNoOverlap": 0
  },
  "passages": [
    {
      "tlc": "string",
      "enteredAt": "string",
      "exitedAt": "string",
      "camsPublished": 0,
      "srm0": [
        {
          "sentAt": "string",
          "connections": [
            {
              "intersectionId": 0,
              "laneConnectionId": 0
            }
          ]
        }
      ]
    }
  ],
  "roadUserId": "string",
  "lastEventProcessed": "string"
}
```

All timestamps are ISO 8601 formatted in UTC.

| Field | Type | Required | Description |
|---|---|---|---|
| `asOf` | string | Yes | Time the history was built |
| `window` | object | Yes | Period covered by the history |
| `window.from` | string | Yes | Start of the period: 24 hours before `window.to` |
| `window.to` | string | Yes | End of the period |
| `summary` | object | Yes | Detection counts for the period |
| `summary.detectionsReceived` | integer | Yes | Number of detections received for the road user |
| `summary.detectionsRejected` | integer | Yes | Number of detections rejected because they failed validation (for example a missing field or an invalid format) |
| `summary.detectionsNoOverlap` | integer | Yes | Number of detections that did not overlap a bicycle lane |
| `passages` | array of objects | Yes | Passages of the road user through a topology |
| `passages.tlc` | string | Yes | Traffic light controller passed; matches the `id` from [`GET /geofences`](#get-geofences) |
| `passages.enteredAt` | string | Yes | Time the road user entered the topology |
| `passages.exitedAt` | string | No (nullable) | Time of the last detection in the topology. `null` while the passage is ongoing; a passage ends when no detection is received for 30 seconds |
| `passages.camsPublished` | integer | Yes | Number of Cooperative Awareness Messages (CAMs) published to UDAP for the road user during the passage |
| `passages.srm0` | array of objects | Yes | Soft priority requests (SRMs) published to UDAP on behalf of the road user |
| `passages.srm0.sentAt` | string | Yes | Time the priority request was sent |
| `passages.srm0.connections` | array of objects | Yes | Lane connections for which priority was requested |
| `passages.srm0.connections.intersectionId` | integer | Yes | Identifier of the intersection |
| `passages.srm0.connections.laneConnectionId` | integer | Yes | Identifier of the lane connection on the intersection |
| `roadUserId` | string | Yes | Identifier of the road user |
| `lastEventProcessed` | string | Yes | Time of the last detection received from the road user |

If there is no history for the road user (yet), the response is:

```http
HTTP/1.1 404 Not Found
Content-Type: application/json
```

### Example

#### Request

```http
GET /v1/history/dcaa2556-02b0-4344-aaef-d52a60520fb5 HTTP/1.1
Host: transit-api.example.com
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "asOf": "2026-11-24T12:41:00.001Z",
  "window": {
    "from": "2026-11-23T12:38:00.000Z",
    "to": "2026-11-24T12:38:14.241Z"
  },
  "summary": {
    "detectionsReceived": 34,
    "detectionsRejected": 0,
    "detectionsNoOverlap": 0
  },
  "passages": [
    {
      "tlc": "7963000a",
      "enteredAt": "2026-11-24T12:37:36.115Z",
      "exitedAt": "2026-11-24T12:38:13.645Z",
      "camsPublished": 34,
      "srm0": [
        {
          "sentAt": "2026-11-24T12:37:36.115Z",
          "connections": [
            { "intersectionId": 12, "laneConnectionId": 244 },
            { "intersectionId": 12, "laneConnectionId": 10 },
            { "intersectionId": 12, "laneConnectionId": 126 }
          ]
        },
        {
          "sentAt": "2026-11-24T12:37:56.581Z",
          "connections": [
            { "intersectionId": 12, "laneConnectionId": 126 }
          ]
        }
      ]
    }
  ],
  "roadUserId": "dcaa2556-02b0-4344-aaef-d52a60520fb5",
  "lastEventProcessed": "2026-11-24T12:38:14.241Z"
}
```
