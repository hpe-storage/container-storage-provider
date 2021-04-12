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
    "array_ip": "10.10.10.1",
    "username": "admin",
    "password": "password"
}
```

#### Response
```json
{
    "id": "193b5de80e54af7a6b000000000000000000002858",
    "array_ip": "10.10.10.1",
    "username": "admin",
    "creation_time": 1562787389,
    "expiry_time": 1562789189,
    "session_token": "1a91b2502ac38b3d8105c662c3f4f003"
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
    "name": "host-01.lab.internet.com",
    "uuid": "41302701-0196-420f-b319-834a79891db0",
    "iqns": [
        "iqn.2019-07.com.nimblestorage:cool-iqn",
        "iqn.2019-06.com.cloudvolumes:cooler-iqn"
    ],
    "networks": [
        "172.28.12.161/20",
        "10.234.63.105/20"
    ],
    "wwpns": []
}
```

#### Response

``` json
{
    "id": "41302701-0196-420f-b319-834a79891db0",
    "name": "host-01.lab.internet.com",
    "uuid": "41302701-0196-420f-b319-834a79891db0",
    "iqns": [
        "iqn.2019-07.com.nimblestorage:cool-iqn",
        "iqn.2019-06.com.cloudvolumes:cooler-iqn"
    ],
    "networks": [
        "172.28.12.161/20",
        "10.234.63.105/20"
    ],
    "wwpns": []
}
```

Note the `uuid` attribute must be used to publish/unpublish volumes.

DELETE `/containers/v1/hosts/{id}`
 * Delete the host identified
 * Expected to fail if any volumes are published to this host.

### `/containers/v1/volumes`

This endpoint is used to manage the creation, update, and deletion of volumes that are used for container environments.  Note the `config` attribute is a map that will receive all of the parameters defined within the storage class associated with the persistent volume being provisioned through the CSI driver.  Annotations on the persistent volume claim will override those parameters from the storage class.  Each CSP implementation is responsible for handling the parameters defined within the storage class associated with the persistent volume claim.

The following methods will be supported against this endpoint.

GET `/containers/v1/volumes`
 * Return all of the volumes used for containers on the array.

#### Request
```
GET http://localhost:8080/csp/containers/v1/volumes
```

#### Response
```json
[
    {
        "id": "067b5b0c6a3d0ece0600000000000000000000001e",
        "name": "volume2",
        "size": 1073741824,
        "description": "my second volume",
        "published": false,
        "config": {
            "parameter1": "default",
            "parameter2": false
        },
        "used_bytes" : 12570624,
        "free_bytes" : 1061171200
    },
    {
        "id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "name": "volume1",
        "size": 1073741824,
        "description": "my first volume",
        "published": false,
        "config": {
            "parameter1": "default",
            "parameter2": false
        },
        "used_bytes" : 12570624,
        "free_bytes" : 1061171200
    }
]
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
    "id": "067b5b0c6a3d0ece0600000000000000000000001d",
    "name": "volume1",
    "size": 1073741824,
    "description": "my first volume",
    "published": false,
    "config": {
        "parameter1": "default",
        "parameter2": false
    },
    "used_bytes" : 12570624,
    "free_bytes" : 1061171200
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
[
    {
        "id": "067b5b0c6a3d0ece0600000000000000000000001d",
        "name": "volume1",
        "size": 1073741824,
        "description": "my first volume",
        "published": false,
        "config": {
            "parameter1": "default",
            "parameter2": false
        },
        "base_snapshot_id": "",
        "volume_group_id": "",
        "used_bytes" : 12570624,
        "free_bytes" : 1061171200
    }
]
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
    "description": "my cool new description"
}
```

#### Response
```json
{
    "id": "067b5b0c6a3d0ece0600000000000000000000001d",
    "name": "volume1",
    "size": 1073741824,
    "description": "my cool new description",
    "published": false,
    "config": {
        "parameter1": "default",
        "parameter2": false
    },
    "used_bytes" : 12570624,
    "free_bytes" : 1061171200
}
```

#### Request with unconfigurable property
```json
{
    "config": {
        "encrypted": true
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
    "name": "my-new-volume",
    "size": "1073741824",
    "description": "my first volume",
    "config": {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Create Response
```json
{
    "id": "063b5de80e54af7a6b0000000000000000000000d0",
    "name": "my-new-volume",
    "size": 1073741824,
    "description": "my first volume",
    "published": false,
    "base_snapshot_id": "",
    "volume_group_id": "073b5de80e54af7a6b000000000000000000000098",
    "config": {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Clone request
```json
{
    "name":"volume2",
    "size": 1073741824,
    "base_snapshot_id":"063b5de80e54af7a6b0000000000000000000000f0",
    "clone": true,
    "config": {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Clone response
```json
{
    "id": "063b5de80e54af7a6b0000000000000000000000d1",
    "name": "volume2",
    "size": 1073741824,
    "published": false,
    "base_snapshot_id": "063b5de80e54af7a6b0000000000000000000000f0",
    "volume_group_id": "073b5de80e54af7a6b000000000000000000000099",
    "config": {
        "parameter1": "default",
        "parameter2": false
    }
}
```

PUT `/containers/v1/volumes/{id}/actions/publish`
 * Publish the volume to the given host.  Publishing means the host can now rescan for the device and find it.  For example, the nimble implementation would add an access control record for the host to this volume.
 * The body must contain the `host_uuid` and the `access_protocol` to be used.

#### Request for iscsi access
```json
{
    "host_uuid": "41302701-0196-420f-b319-834a79891db0",
    "access_protocol": "iscsi"
}
```

#### Response for single target IQN
```json
{
    "access_protocol": "iscsi",
    "discovery_ips": [
        "172.89.82.10"
    ],
    "lun_id": 3,
    "PeerArrayDetails": [{"lun_id":1,"target_names":["iqn.2000-05.com.3pardata:20210002ac01db2c","iqn.2000-05.com.3pardata:21210002ac01db2c"],"discovery_ips":["1.1.1.1","2.2.2.2"]}],
    "serial_number": "4349bd228896f1236c9ce9006592f26f",
    "target_names": ["iqn.2007-11.com.nimblestorage:group-array1-g3b5de80e54af7a6b"]
}
```
 * Note that `chap_user` and `chap_password` must also be part of the response if CHAP details were provided as part of the Node definition.
 * `target_names`  should be returned as an array of target IQNs from CSP and converted to CSV format by CSI driver.

 #### Response for multiple target IQNs
```json
{
    "access_protocol": "iscsi",
    "discovery_ips": [
        "172.89.82.10"
    ],
    "lun_id": 3,
    "PeerArrayDetails": [{"lun_id":1,"target_names":["iqn.2000-05.com.3pardata:20210002ac01db2c","iqn.2000-05.com.3pardata:21210002ac01db2c"],"discovery_ips":["1.1.1.1","2.2.2.2"]}],
    "serial_number": "4349bd228896f1236c9ce9006592f26f",
    "target_names": ["iqn.2000-05.com.3pardata:21210002ac01db31,iqn.2000-05.com.3pardata:21220002ac01db31"]
}
```

#### Request for fc access
```json
{
    "host_uuid": "41302701-0196-420f-b319-834a79891db0",
    "access_protocol": "fc"
}
```

#### Response
```json
{
    "access_protocol": "fc",
    "lun_id": 0,
    "PeerArrayDetails": [{"lun_id":1}],
    "serial_number": "4349bd228896f1236c9ce9006592f26f"
}
```

PUT `/containers/v1/volumes/{id}/actions/unpublish`
 * Unpublish the volume from the given host.  For example, the Nimble implementation would remove an access control record for the specified host during this operation.
 * The body must contain the host UUID

#### Request
```json
{
    "host_uuid": "41302701-0196-420f-b319-834a79891db0"
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
[
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
            "parameter1": "default",
            "parameter2": false
        }
    }
]
```

GET `/containers/v1/snapshots/?volume_id=063b5de80e54af7a6b0000000000000000000000d0&name=mySnapshot`
 * Return all of the snapshots of the `volume_id` with `name` of `mySnapshot` used for containers on the array.

#### Request
```
GET http://localhost:8080/csp/containers/v1/snapshots?volume_id=067b5b0c6a3d0ece0600000000000000000000001d&name=my-first-snapshot
```

#### Response
```json
[
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
            "parameter1": "default",
            "parameter2": false
        }
    }
]
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
    "id": "047b5b0c6a3d0ece06000000000000003000000010",
    "name": "my-first-snapshot",
    "size": 1073741824,
    "description": "my first snapshot",
    "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
    "volume_name": "volume1",
    "creation_time": 1565206041,
    "ready_to_use": true,
    "config": {
        "parameter1": "default",
        "parameter2": false
    }
}
```

POST `/containers/v1/snapshots`
 * Create a new snapshot.
 * The body must contain the new snapshot definition

#### Request
```json
{
    "name": "my-first-snapshot",
    "description": "my first snapshot",
    "volume_id": "067b5b0c6a3d0ece0600000000000000000000001d",
    "config": {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Response
```json
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
        "parameter1": "default",
        "parameter2": false
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

### `/containers/v1/volume_groups`

This endpoint is used to manage the creation and deletion of volume groups that are used for container environments.

The following methods will be supported against this endpoint.

GET `/containers/v1/volume_groups`

 * Return all of the volumegroups used for containers on the array.

#### Request
```
GET http://localhost:8080/csp/containers/v1/volume_groups
```

#### Response

```json
[
    {
        "id" : "072265c9672660666b000000000000000000000005",
        "name" : "volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85",
        "description": "my-first-volumegroup",
        "volumes" :
        [
           {
            "id" : "062265c9672660666b000000000000000000000005",
            "name" : "pvc-cf76e03f-c7e6-4507-8c89-189f9ece058a"
           }
        ],
        "creation_time" : 1592249347,
        "config":
        {
            "parameter1": "default",
            "parameter2": false
        }
    },
    {
        "id" : "072265c9672660666b000000000000000000000007",
        "description": "my-second-volumegroup",
        "name" : "volumegroup-b5935dba-e01d-475c-b901-4eca0492a480",
        "volumes" : [ ],
        "creation_time" : 1592265103,
        "config":
        {
            "parameter1": "default",
            "parameter2": false
        }
    }
]
```

GET `/containers/v1/volume_groups/{id}`

 * Return the volumegroup identified by the given id.
 * Should return HTTP 404 (Not Found) when a volumegroup with that id cannot be found

#### Request

```
GET http://localhost:8080/csp/containers/v1/volume_groups/072265c9672660666b000000000000000000000005
```

#### Response

```json
{
    "id" : "072265c9672660666b000000000000000000000005",
    "description": "my-first-volumegroup",
    "name" : "volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85",
    "volumes" :
    [
        {
        "id" : "062265c9672660666b000000000000000000000005",
        "name" : "pvc-cf76e03f-c7e6-4507-8c89-189f9ece058a"
        }
    ],
    "creation_time" : 1592249347,
    "config":
    {
        "parameter1": "default",
        "parameter2": false
    }
}
```

GET `/containers/v1/volume_groups?name=volumeGroupName`

 * Return the volumegroups with the given name.
 * Should return HTTP 404 (Not Found) when a volumegroup with that name cannot be found

#### Request

```
GET http://localhost:8080/csp/containers/v1/volume_groups?name=volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85
```

#### Response

```json
{
    "id" : "072265c9672660666b000000000000000000000005",
    "name" : "volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85",
    "description": "my-first-volumegroup",
    "volumes" :
    [
        {
        "id" : "062265c9672660666b000000000000000000000005",
        "name" : "pvc-cf76e03f-c7e6-4507-8c89-189f9ece058a"
        }
    ],
    "creation_time" : 1592249347,
    "config":
    {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Request with unknown volumegroup name

```
GET http://localhost:8080/csp/containers/v1/volume_groups?name=bob
```

#### Response

```json
{
  "code" : "Not Found",
  "message" : "Volume group with name bob not found."
}
```

POST `/containers/v1/volume_groups`

 * Create a new volumegroup.
 * The body must contain the new volumegroup definition

#### Create Request

```json
{
    "name": "new-new-volumegroup",
    "description": "my first volumegroup",
    "config":
    {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Create Response

```json
 {
  "id" : "072265c9672660666b00000000000000000000000a",
  "name" : "my-new-volumegroup",
  "description" : "my first volume group",
  "volumes" : [ ],
  "creation_time" : 1592270415,
  "config":
  {
      "parameter1": "default",
      "parameter2": false
  }
}
```

DELETE `/containers/v1/volume_groups/{id}`

 * Delete the volumegroup identified

```
DELETE http://localhost:8080/csp/containers/v1/volume_groups/072265c9672660666b00000000000000000000000a
```
#### Response
```
204 No Content
```

### `/containers/v1/snapshot_groups`

This endpoint is used to manage the creation and deletion of snapshot groups that are used for container environments.

The following methods will be supported against this endpoint.

GET `/containers/v1/snapshot_groups?volume_group_id=072265c9672660666b000000000000000000000005`

 * Return all of the snapshotgroups of the `volume_group_id` used for containers on the array.
 * `volume_group_id` is mandatory

#### Request

```
GET http://localhost:8080/csp/containers/v1/snapshot_groups?volume_group_id=072265c9672660666b000000000000000000000005
```

#### Response

```json
[
    {
        "id" : "052265c9672660666b000000000000000000000001",
        "name" : "snapshotgroup-3b3062ac-3ef4-4db2-a7a8-1bad2270a1c8",
        "volume_group_id" : "072265c9672660666b000000000000000000000005",
        "volume_group_name" : "volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85",
        "creation_time" : 1592249883,
        "snapshots" :
        [
             {
                  "id" : "042265c9672660666b000000000000000900000001",
                  "name" : "snapshotgroup-3b3062ac-3ef4-4db2-a7a8-1bad2270a1c8",
                  "volume_id" : "062265c9672660666b000000000000000000000005",
                  "volume_name" : "pvc-cf76e03f-c7e6-4507-8c89-189f9ece058a"
            }
        ],
        "config":
        {
            "parameter1": "default",
            "parameter2": false
        }
    }
]
```

GET `/containers/v1/snapshot_groups/{id}`
 * Return the snapshotgroup identified by the given id.

#### Request
```
GET http://localhost:8080/csp/containers/v1/snapshot_groups/052265c9672660666b000000000000000000000001
```

#### Response

```json
{
    "id" : "052265c9672660666b000000000000000000000001",
    "name" : "snapshotgroup-3b3062ac-3ef4-4db2-a7a8-1bad2270a1c8",
    "volume_group_id" : "072265c9672660666b000000000000000000000005",
    "volume_group_name" : "volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85",
    "creation_time" : 1592249883,
    "snapshots" :
    [
        {
            "id" : "042265c9672660666b000000000000000900000001",
            "name" : "snapshotgroup-3b3062ac-3ef4-4db2-a7a8-1bad2270a1c8",
            "volume_id" : "062265c9672660666b000000000000000000000005",
            "volume_name" : "pvc-cf76e03f-c7e6-4507-8c89-189f9ece058a"
        }
    ],
    "config":
    {
        "parameter1": "default",
        "parameter2": false
    }
}
```

POST `/containers/v1/snapshot_groups`

 * Create a new snapshotgroup.
 * The body must contain the new snapshotgroup definition

#### Request

```json
{
    "name": "my-first-snapshotgroup",
    "volume_group_id": "072265c9672660666b000000000000000000000005",
    "config":
    {
        "parameter1": "default",
        "parameter2": false
    }
}
```

#### Response

```json
{
    "id" : "052265c9672660666b000000000000000000000003",
    "name" : "my-first-snapshotgroup",
    "volume_group_id" : "072265c9672660666b000000000000000000000005",
    "volume_group_name" : "volumegroup-6c58267d-b324-4b7f-88fc-6597bc62ff85",
    "creation_time" : 1592277183,
    "snapshots" :
    [
        {
            "id" : "042265c9672660666b000000000000000900000003",
            "name" : "my-first-snapshotgroup",
            "volume_id" : "062265c9672660666b000000000000000000000005",
            "volume_name" : "pvc-cf76e03f-c7e6-4507-8c89-189f9ece058a"
        }
    ],
    "config":
    {
        "parameter1": "default",
        "parameter2": false
    }
}
```

DELETE `/containers/v1/snapshot_groups/{id}`

 * Delete the snapshotgroup identified

#### Request to delete a snapshotgroup 
```
DELETE http://localhost:8080/csp/containers/v1/snapshot_groups/052265c9672660666b000000000000000000000003
```

#### Response

```
204 No Content
```

DELETE `/containers/v1/snapshot_groups/{id}?replication_partner=replPartner`

 * Delete the downstream snapshotgroup 

#### Request to delete a downstream snapshotgroup
```
DELETE http://localhost:8080/csp/containers/v1/snapshot_groups/052265c9672660666b000000000000000000000003?replication_partner=replPartner
```

```json
["040419a3f59745e5f100000000000000f50001ac9b"]
```

#### Response

```
204 No Content
```

### `/containers/v1/replication_partners`

This endpoint is used to get the details of storage system replication partners.

The following methods will be supported against this endpoint.

GET `/containers/v1/replication_partners`

 * Return all of the replication partners used for containers on the array.

#### Request

```
GET http://localhost:8080/csp/containers/v1/replication_partners
```

#### Response

```json
[
    {
        "id" : "052265c9672660666b000000000000000000000001",
        "partner_name" : "group-sjc-test",
        "repl_direction": "downstream",
        "is_alive" : "true",
    }
]
```


## Object sets

| Object set | Path | Query Param | Operations | Actions |
| ---------- | ---- | ----------- | ---------- | ------- |
| tokens | /containers/v1/tokens | | post<br>delete | |
| hosts | /containers/v1/hosts | | post<br>delete | |
| volumes | /containers/v1/volumes | name | get<br>put<br>post<br>delete | publish<br>unpublish |
| snapshots | /containers/v1/snapshots | volume_id<br>name | get<br>post<br>delete | |
| volume_groups | /containers/v1/volume_groups | name | get<br>post<br>delete | |
| snapshot_groups | /containers/v1/snapshot_groups | volume_group_id | get<br>post<br>delete | |
| replication_partners | /containers/v1/replication_partners | name | get | |


## Objects

| Objects | Attributes | Type | Mandatory | Input | Output |
| ------- | ---------- | ---- | --------- | ----- | ------ |
| Token | | | | | |
| | id | string | | | X |
| | username | string | X | X | X |
| | password | string | X | X | |
| | array_ip | string | when running on cluster | X | |
| | session_token (used for all future authentication) | string | X | | X |
| | creation_time | number | | | X |
| | expiry_time | number | | | X |
| Host | | | | | |
| | id | string | | | X |
| | uuid | string | X | X | X |
| | name | string | X | X | X |
| | iqns | list\<string\> | when wwpns are not specified | X | X |
| | wwpns | list\<string\> | when iqns are not specified | X | X |
| | networks | list\<string\> | when iqns are specified | X | X |
| | chap_user | string |  | X | X |
| | chap_password | string | | X | X |
| Volume | | | | | |
| | id | string | | | X |
| | name | string | X | X | X |
| | size | number | X | X | X |
| | description | string | | X | X |
| | base_snapshot_id | string | | X | X |
| | clone | boolean |  | X | X |
| | volume_group_id | string | | X | X |
| | published | boolean |  | | X |
| | config | map[string]interface{} | | | X |
| | free_bytes | number | | | X |
| | used_bytes | number | | | X |
| PublishOptions | | | | | |
| | host_uuid | string | X | X | |
| | access_protocol | string | X | X | |
| PublishInfo | | | | | |
| | serial_number | string | X | | X |
| | access_protocol | string | X | | X |
| | lun_id | number | X | | X |
| | PeerArrayDetails | list\<SecondaryLunInfo\> | X | | X|
| | target_names | list\<string\> | only for iscsi | | X |
| | discovery_ips | list\<string\> | only for iscsi | | X |
| | chap_user | string | only for iscsi | | X |
| | chap_password | string | only for iscsi | | X |
|  SecondaryLunInfo | | | | | |
| | lun_id | number | X | X |
| | target_names | list\<string\> | only for iscsi | | X |
| | discovery_ips | list\<string\> | only for iscsi | | X |
| UnpublishOptions | | | | | |
| | host_uuid | string | X | X | |
| Snapshot | | | | | |
| | id | string | | | X |
| | name | string | X | X | X |
| | size | number | | | X |
| | description | string | | X | X |
| | volume_id | string | X | X | X |
| | volume_name | string | | | X |
| | creation_time (seconds) | number | X | | X |
| | ready_to_use | boolean | X | | X |
| | config | map[string]interface{} | | X | X |
| VolumeGroup | | | | | |
| | id | string | | | X |
| | name | string | X | X | X |
| | description | string | |  | X |
| | creation_time (seconds) | number |  | | X |
| | volumes | list\<Volume\> |  | | X |
| | config | map[string]interface{} | | X | X |
| SnapshotGroup | | | | | |
| | id | string | | | X |
| | name | string | X | X | X |
| | volume_group_id | string | | X | X |
| | volume_group_name | string | |  | X |
| | creation_time (seconds) | number |  | | X |
| | volumes | list\<Snapshot\> |  | | X |
| | config | map[string]interface{} | | X | X |
| ReplicationPartner | | | | | |
| | id | string | | | X |
| | name | string | X |  | X |
| | is_alive | boolean | |  | X |
| | repl_direction | string | |  | X |