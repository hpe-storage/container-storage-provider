# Container Storage Provider (CSP) spec

The Container Storage Provider (CSP) is a REST API specification that defines how provisioning, mounting, and deallocation workflows will be invoked from a host client that intends to use a storage provider.  Any storage vendor that intends to use the HPE CSI driver must implement the specification defined herein.

The HPE CSI Driver expects the CSP implementation to either run as a service with the kubernetes cluster where the driver is deployed or as a service off of the cluster (possibly on the array).  When running within the kubernetes cluster, the driver will connect to the `serviceName` and `servicePort` described in the secret over http (e.g. `http://<backend>:<servicePort>`).  When running off the cluster as a service, the driver will connect to the `backend`, `servicePort`, and `contextPath` described in the secret over https (e.g. `https://<backend>:<servicePort>/<contextPath>`)

## REST endpoints

### `/containers/v1/tokens`

This endpoint manages access to the underlying storage array.  The user must create a token that is to be used on subsequent requests made against the storage array.  All tokens are expected to expire after a certain period of time.  The HPE CSI driver will attempt to login again if the token is the client receives a `401 Unauthorized` HTTP response.

POST `/containers/v1/tokens`
 * Specifies the `username` and `password`
 * Specifies an optional `array_ip` argument used if a CSP implementation runs off-array.  The `array_ip` field points to the hostname or IP address of the storage array on which to invoke the CSP implementation API.
 * The response will report a `session_token` property.
 * All subsequent requests from clients will include the `x-auth-token` header set to the `session_token` reported from this POST request.  They will also include the optional `x-array-ip` header when specified.

#### Request
```json
{
    "data": {
        "array_ip": "10.10.10.1",
        "username": "admin",
        "password": "password"
    }
}
```

#### Response
```json
{
    "data": {
        "id": "193b5de80e54af7a6b000000000000000000002858",
        "array_ip": "10.10.10.1",
        "username": "admin",
        "creation_time": 1562787389,
        "expiry_time": 1562789189,
        "session_token": "1a91b2502ac38b3d8105c662c3f4f003"
    }
}
```

DELETE `/containers/v1/tokens/{id}`
 * Delete the token identified

### `/containers/v1/hosts`

This endpoint is used to register hosts with the CSP.

POST `/containers/v1/hosts`
 * Registers a host with the CSP
 * The body must contain a host UUID, name, initiators (either IQNs, WWPNs or both), networks (if using iSCSI), and optional CHAP information (for iSCSI)
 * The response will report a new Host.  The ID returned from this POST request be used to publish and unpublish a volume

#### Request

```json
{
    "data": {
        "name": "host-01.lab.internet.com",
        "uuid": "41302701-0196-420f-b319-834a79891db0",
        "iqns": [
            "iqn.2019-07.com.nimblestorage:cool-iqn",
            "iqn.2019-06.com.cloudvolumes:cooler-iqn"
        ],
        "networks": [
            "10.10.50.0",
            "10.234.60.0"
        ],
        "wwpns": []
    }
}
```

#### Response

``` json
{
    "data": {
        "id": "41302701-0196-420f-b319-834a79891db0",
        "name": "host-01.lab.internet.com",
        "uuid": "41302701-0196-420f-b319-834a79891db0",
        "iqns": [
            "iqn.2019-07.com.nimblestorage:cool-iqn",
            "iqn.2019-06.com.cloudvolumes:cooler-iqn"
        ],
        "networks": [
            "10.10.50.0",
            "10.234.60.0"
        ],
        "wwpns": []
    }
}
```

Note the id attribute must be used to publish/unpublish volumes.

DELETE `/containers/v1/hosts/{id}`
 * Delete the host identified
 * Expected to fail if any volumes are published to this host.

### `/containers/v1/volumes`

This endpoint is used to manage the creation, update, and deletion of volumes that are used for container environments.  The following methods will be supported against this endpoint.

GET `/containers/v1/volumes`
 * Return all of the volumes used for containers on the array.

#### Request
```
GET http://localhost:8080/csp/containers/v1/volumes
```

#### Response
```json
{
    "data": [
        {
            "id": "067b5b0c6a3d0ece0600000000000000000000001e",
            "name": "volume2",
            "size": 1073741824,
            "description": "my second volume",
            "published": false,
            "config": {
                "pool": "default",
                "folder": "",
                "encrypted": false,
                "thick": false,
                "performance_policy": "SQL Server",
                "dedupe_enabled": false,
                "limit_iops": -1,
                "limit_mbps": -1,
                "destroy_on_delete": false,
                "sync_on_detach": true,
                "clone_of": ""
            }
        },
        {
            "id": "067b5b0c6a3d0ece0600000000000000000000001d",
            "name": "volume1",
            "size": 1073741824,
            "description": "my first volume",
            "published": false,
            "config": {
                "pool": "default",
                "folder": "",
                "encrypted": false,
                "thick": false,
                "performance_policy": "SQL Server",
                "dedupe_enabled": false,
                "limit_iops": -1,
                "limit_mbps": -1,
                "destroy_on_delete": false,
                "sync_on_detach": true,
                "clone_of": ""
            }
        }
    ]
}
```

GET `/containers/v1/volumes/{id}`
 * Return the volume identified by the given id.
 * Should return HTTP 404 (Not Found) when a volume with that id cannot be found

#### Request
```
GET http://localhost:8080/csp/containers/v1/volumes/067b5b0c6a3d0ece0600000000000000000000001d
```

#### Response
```json
{
    "data": {
        "id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "name": "volume1",
        "size": 1073741824,
        "description": "my first volume",
        "published": false,
        "config": {
            "pool": "default",
            "folder": "",
            "encrypted": false,
            "thick": false,
            "performance_policy": "SQL Server",
            "dedupe_enabled": false,
            "limit_iops": -1,
            "limit_mbps": -1,
            "destroy_on_delete": false,
            "sync_on_detach": true,
            "clone_of": ""
        }
    }
}
```

GET `/containers/v1/volumes?name=volumeName`
 * Return the volumes with the given name.
 * Should return HTTP 404 (Not Found) when a volume with that name cannot be found

#### Request
```
GET http://localhost:8080/csp/containers/v1/volumes?name=volume1
```

#### Response
```json
{
    "data": [
        {
            "id": "067b5b0c6a3d0ece0600000000000000000000001d",
            "name": "volume1",
            "size": 1073741824,
            "description": "my first volume",
            "published": false,
            "config": {
                "pool": "default",
                "folder": "",
                "encrypted": false,
                "thick": false,
                "performance_policy": "SQL Server",
                "dedupe_enabled": false,
                "limit_iops": -1,
                "limit_mbps": -1,
                "destroy_on_delete": false,
                "sync_on_detach": true,
                "clone_of": ""
            },
            "base_snapshot_id": "",
            "volume_group_id": ""
        }
    ]
}
```

#### Request with unknown volume name
```
GET http://localhost:8080/csp/containers/v1/volumes?name=bob
```

#### Response
```json
{
    "errors": [
        {
            "code": "Not Found",
            "message": "Volume with name bob not found."
        }
    ]
}
```

PUT `/containers/v1/volumes/{id}`
 * Update/Edit settings on the volume

#### Request
```json
{
    "data": {
        "description": "my cool new description"
    }
}
```

#### Response
```json
{
    "data": {
        "id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "name": "volume1",
        "size": 1073741824,
        "description": "my cool new description",
        "published": false,
        "config": {
            "pool": "default",
            "folder": "",
            "encrypted": false,
            "thick": false,
            "performance_policy": "SQL Server",
            "dedupe_enabled": false,
            "limit_iops": -1,
            "limit_mbps": -1,
            "destroy_on_delete": false,
            "sync_on_detach": true,
            "clone_of": ""
        }
    }
}
```

#### Request with unconfigurable property
```json
{
    "data": {
        "config": {
            "encrypted": true
        }
    }
}
```

#### Response
```json
{
    "errors": [
        {
            "code": "Bad Request",
            "message": "The request could not be understood by the server. Unexpected argument 'encrypted'."
        }
    ]
}
```

POST `/containers/v1/volumes`
 * Create a new volume. Also create a clone from an existing snapshot or volume.
 * The body must contain the new volume definition

#### Create Request
```json
{
    "data": {
        "name": "my-new-volume",
        "size": "1073741824",
        "description": "my first volume",
        "config": {
            "performance_policy": "default",
            "protection_template": "Retain-30Daily",
            "sync_on_detach": true
        }
    }
}
```

#### Create Response
```json
{
    "data": {
        "id": "063b5de80e54af7a6b0000000000000000000000d0",
        "name": "my-new-volume",
        "size": 1073741824,
        "description": "my first volume",
        "published": false,
        "base_snapshot_id": "",
        "volume_group_id": "073b5de80e54af7a6b000000000000000000000098",
        "config": {
            "dedupe_enabled": false,
            "encrypted": false,
            "pool": "default",
            "folder": "",
            "limit_iops": -1,
            "limit_mbps": -1,
            "performance_policy": "default",
            "protection_template": "Retain-30Daily",
            "sync_on_detach": true,
            "thick": false
        }
    }
}
```

#### Clone request
```json
{
  "data": {  
    "name":"volume2",
    "size": 1073741824,
    "base_snapshot_id":"063b5de80e54af7a6b0000000000000000000000f0",
    "clone": true,
    "config": {
      "performance_policy": "default",
      "protection_template": "Retain-30Daily",
      "sync_on_detach": true
    }
  }
}
```

#### Clone response
```json
{
    "data": {
        "id": "063b5de80e54af7a6b0000000000000000000000d1",
        "name": "volume2",
        "size": 1073741824,
        "published": false,
        "base_snapshot_id": "063b5de80e54af7a6b0000000000000000000000f0",
        "volume_group_id": "073b5de80e54af7a6b000000000000000000000099",
        "config": {
            "clone_of": "my-new-volume",
            "dedupe_enabled": false,
            "description": "",
            "encrypted": false,
            "ephemeral": false,
            "folder": "",
            "limit_iops": -1,
            "limit_mbps": -1,
            "performance_policy": "default",
            "pool": "default",
            "protection_template": "Retain-30Daily",
            "sync_on_detach": true,
            "thick": false
        }
    }
}
```

PUT `/containers/v1/volumes/{id}/actions/publish`
 * Publish the volume to the given host.  Publishing means the host can now rescan for the device and find it.  For example, the nimble implementation would add an access control record for the host to this volume.
 * The body must contain the `host_id` and the `access_protocol` to be used.

#### Request for iscsi access
```json
{
    "data": {
        "host_id": "41302701-0196-420f-b319-834a79891db0",
        "access_protocol": "iscsi"
    }
}
```

#### Response
```json
{
    "data": {
        "access_protocol": "iscsi",
        "discovery_ip": "172.89.82.10",
        "lun_id": 0,
        "serial_number": "4349bd228896f1236c9ce9006592f26f",
        "target_name": "iqn.2007-11.com.nimblestorage:group-array1-g3b5de80e54af7a6b"
    }
}
```
 * Note that `chap_user` and `chap_password` must also be part of the response if CHAP details were provided as part of the Node definition.

#### Request for fc access
```json
{
    "data": {
        "host_id": "41302701-0196-420f-b319-834a79891db0",
        "access_protocol": "fc"
    }
}
```

#### Response
```json
{
    "data": {
        "access_protocol": "fc",
        "lun_id": 0,
        "serial_number": "4349bd228896f1236c9ce9006592f26f"
    }
}
```

PUT `/containers/v1/volumes/{id}/actions/unpublish`
 * Unpublish the volume from the given host.  For example, the Nimble implementation would remove an access control record for the specified host during this operation.
 * The body must contain the host ID

#### Request
```json
{
    "data": {
        "host_id": "41302701-0196-420f-b319-834a79891db0"
    }
}
```

#### Response
```
204 No Content
```

DELETE `/containers/v1/volumes/{id}`
 * Delete the volume identified
 * Should fail if the volume is published

#### Request while volume is published
```
DELETE http://localhost:8080/csp/containers/v1/volumes/067b5b0c6a3d0ece0600000000000000000000001d
```
#### Response
```json
{
    "errors": [
        {
            "code": "Bad Request",
            "message": "Cannot delete a published volume"
        }
    ]
}
```

### `/containers/v1/snapshots`
This endpoint is used to manage the creation and deletion of snapshots that are used for container environments.  The following methods will be supported against this endpoint.

GET `/containers/v1/snapshots?volume_id=413027010196420fb319834a79891db0`
 * Return all of the snapshots of the `volume_id` used for containers on the array.
 * `volume_id` is mandatory

#### Request
```
GET http://localhost:8080/csp/containers/v1/snapshots?volume_id=067b5b0c6a3d0ece0600000000000000000000001d
```

#### Response
```json
{
    "data": [
        {
            "id": "047b5b0c6a3d0ece06000000000000003000000010",
            "name": "my-first-snapshot",
            "size": 1073741824,
            "description": "my first snapshot",
            "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
            "volume_name": "volume1",
            "creation_time": 1565206041,
            "ready_to_use": true
            "config": {
                "online": false,
                "writable": false
            }
        }
    ]
}
```

GET `/containers/v1/snapshots/?volume_id=063b5de80e54af7a6b0000000000000000000000d0&name=mySnapshot`
 * Return all of the snapshots of the `volume_id` with `name` of `mySnapshot` used for containers on the array.

#### Request
```
GET http://localhost:8080/csp/containers/v1/snapshots?volume_id=067b5b0c6a3d0ece0600000000000000000000001d&name=my-first-snapshot
```

#### Response
```json
{
    "data": [
        {
            "id": "047b5b0c6a3d0ece06000000000000003000000010",
            "name": "my-first-snapshot",
            "size": 1073741824,
            "description": "my first snapshot",
            "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
            "volume_name": "volume1",
            "creation_time": 1565206041,
            "ready_to_use": true,
            "config": {
                "online": false,
                "writable": false
            }
        }
    ]
}
```

GET `/containers/v1/snapshots/{id}`
 * Return the snapshot identified by the given id.

#### Request
```
GET http://localhost:8080/csp/containers/v1/snapshots/047b5b0c6a3d0ece06000000000000003000000010
```

#### Response
```json
{
    "data": {
        "id": "047b5b0c6a3d0ece06000000000000003000000010",
        "name": "my-first-snapshot",
        "size": 1073741824,
        "description": "my first snapshot",
        "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "volume_name": "volume1",
        "creation_time": 1565206041,
        "ready_to_use": true,
        "config": {
            "online": false,
            "writable": false
        }
    }
}
```

POST `/containers/v1/snapshots`
 * Create a new snapshot.
 * The body must contain the new snapshot definition

#### Request
```json
{
    "data": {
        "name": "my-first-snapshot",
        "description": "my first snapshot",
        "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "config": {
            "online": false,
            "writable": false
        }
    }
}
```

#### Response
```json
{
    "data": {
        "id": "047b5b0c6a3d0ece06000000000000003000000010",
        "name": "my-first-snapshot",
        "size": 1073741824,
        "description": "my first snapshot",
        "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "volume_name": "volume1",
        "creation_time": 1565206041,
        "ready_to_use": true,
        "config": {
            "online": false,
            "writable": false
        }
    }
}
```

DELETE `/containers/v1/snapshots`
 * Delete the snapshot identified
 * Should fail if there is an existing clone created from this snapshot.

#### Request to delete a snapshot with a clone
```
DELETE http://localhost:8080/csp/containers/v1/snapshots/047b5b0c6a3d0ece06000000000000003000000010
```

#### Response
```json
{
    "errors": [
        {
            "code": "Conflict",
            "message": "The request could not be completed due to a conflict. Snapshot my-first-snapshot for volume volume1 has a clone."
        }
    ]
}
```

## Object sets

| Object set | Path | Query Param | Operations | Actions |
| ---------- | ---- | ----------- | ---------- | ------- |
| tokens | /containers/v1/tokens | | post<br>delete | |
| hosts | /containers/v1/hosts | | post<br>delete | |
| volumes | /containers/v1/volumes | name | get<br>put<br>post<br>delete | publish<br>unpublish |
| snapshots | /containers/v1/snapshots | volume_id<br>name | get<br>post<br>delete | |

## Objects

| Objects | Attributes |
| ------- | ---------- |
| Token | id (string)<br>username (string) - mandatory<br>password (string) - mandatory<br>session_token (string) - used for all future authentication<br>expiry_time (number) |
| Host | id (string)<br>uuid (string) - mandatory<br>name (string) - mandatory<br>iqns (list\<string\>) - mandatory if wwpns are not specified<br>wwpns (list\<string\>) - mandatory if iqns are not specified<br>networks (list\<string\>) - mandatory if iqns are specified<br>chap_user (string)<br>chap_password (string) |
| Volume | id (string)<br>name (string) - mandatory<br>size (number in bytes) - mandatory<br>base_snapshot_id (string) - used for cloning<br>volume_group_id (string)<br>clone (boolean) - true to create a clone<br>published (boolean) - true if the volume has been published; false otherwise (mandatory)<br>config (map[string]interface{}) |
| VolumeConfig | map[string]interface{} - vendor specific volume properties |
| PublishOptions | host_id (string) - mandatory<br>access_protocol ("fc" or "iscsi") - mandatory |
| PublishInfo | <ul><li>Block<ul><li>serial_number (string) - mandatory</li><li>access_protocol ("fc" or "iscsi") - mandatory</li><li>lun_id (number) - mandatory</li><li>target_name (string) - mandatory for iscsi</li><li>discovery_ip (string) - mandatory for iscsi</li><li>chap_user (string) - optional for iscsi</li><li>chap_password (string) - optional for iscsi</li></ul></li><li>VMDK<ul><li>pci_id (number)</li><li>unit_number (number)</li></ul></li><li>NFS<ul><li>TBD</li></ul></li></ul> |
| UnpublishOptions | host_id (string) - mandatory |
| Snapshot | id (string)<br>name (string) - mandatory<br>size (number) - in bytes<br>volume_id (string) - mandatory<br>volume_name (string)<br>creation_time (number) - in seconds (mandatory)<br>ready_to_use (boolean) - mandatory<br>config (map[string]interface{}) |
| SnapshotConfig | map[string]interface{} - vendor specific snapshot properties |


