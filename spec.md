# Container Storage Provider (CSP) spec

The Container Storage Provider (CSP) is a REST API specification that defines how provisioning, mounting, and deallocation workflows will be invoked from a host client that intends to use a storage provider.  Any storage vendor that intends to use the HPE CSI driver must implement the specification defined herein.

## REST endpoints

### `/containers/v1/tokens`

This endpoint manages access to the underlying storage array.  The user must create a token that is to be used on subsequent requests made against the storage array.  All tokens are expected to expire after a certain period of time.  The HPE CSI driver will attempt to login again if the token is the client receives a `401 Unauthorized` HTTP response.

POST `/containers/v1/tokens`
 * Specifies the username and password
 * Specifies an optional `backend` argument used if a CSP implementation runs off-array.  The `backend` field points to the hostname or IP address of the storage array on which to invoke the CSP implementation API.
 * The response will report a session_token property.
 * All subsequent requests from clients will include the `x-auth-token` header set to the session_token reported from this POST request.  They will also include the optional `x-backend` header when specified.

#### Request
```json
{
  "data": {
    "backend": "10.10.10.1",
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
    "backend": "10.10.10.1",
    "username": "admin",
    "creation_time": 1562787389000,
    "session_token": "1a91b2502ac38b3d8105c662c3f4f003"
  }
}
```

### `/containers/v1/nodes`

This endpoint is used to register nodes (hosts) with the CSP.

POST `/containers/v1/nodes`
 * Registers a node with the CSP
 * The body must contain a host UUID, name, initiators (either IQNs, WWPNs or both), and networks (if using iSCSI)
 * The response will report a new Node.  The ID returned from this POST request be used to publish and unpublish a volume

#### Request

```json
{
  "data": {
    "name": "node-01.lab.internet.com",
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
    "name": "gcostea-k8s1.rtpvlab.nimblestorage.com",
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

DELETE `/containers/v1/nodes/{id}`
 * Delete the node identified


### `/containers/v1/volumes`

This endpoint is used to manage the creation, update, and deletion of volumes that are used for container environments.  The following methods will be supported against this endpoint.

GET `/containers/v1/volumes`
 * Return all of the volumes used for containers on the array.  
 * This short form will only return volume id and name.

GET `/containers/v1/volumes/detail`
 * Return all of the volumes used for containers on the array.  
 * This detailed form returns all volume properties

GET `/containers/v1/volumes/detail?name=volumeName`
 * Return the volumes with the given name.  
 * This detailed form returns all volume properties

GET `/containers/v1/volumes/{id}`
 * Return the volume identified by the given id.
 * This will return all volume properties.

PUT `/containers/v1/volumes/{id}`
 * Update/Edit settings on the volume
 * The body must contain the new volume settings

POST `/containers/v1/volumes`
 * Create a new volume. Also create a clone from an existing snapshot or volume.
 * The body must contain the new volume definition

#### Create Request
```json
{
  "data": {
    "name": "my-new-volume",
    "size": "1073741824",
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
        "in_use": false,
        "base_snapshot_id": "",
        "consistency_set_id": "073b5de80e54af7a6b000000000000000000000098",
        "config": {
            "dedupe_enabled": false,
            "description": "",
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
        "in_use": false,
        "base_snapshot_id": "063b5de80e54af7a6b0000000000000000000000f0",
        "consistency_set_id": "073b5de80e54af7a6b000000000000000000000099",
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

POST `/containers/v1/volumes/{id}/actions/publish`
 * Publish the volume (Add an Access Control Record) to the given node
 * The body must contain the node ID
 * `access_protocol` is an optional field.  The Nimble CSP will favor FC if not specified.  Other CSPs can do as they see fit.

#### Request
```json
{
    "data": {
        "node_id": "41302701-0196-420f-b319-834a79891db0"
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
        "target_name": "iqn.2007-11.com.nimblestorage:group-gcostea-array1-g3b5de80e54af7a6b",
        "target_scope": "group"
    }
}
```

POST `/containers/v1/volumes/{id}/actions/unpublish`
 * Unpublish the volume (Remove an Access Control Record) from the given node
 * The body must contain the node ID

#### Request
```json
{
    "data": {
        "node_id": "41302701-0196-420f-b319-834a79891db0"
    }
}
```

#### Response
```json
{}
```

DELETE `/containers/v1/volumes/{id}`
 * Delete the volume identified
 * Should fail if the volume is in use (has an Access Control Record assigned)

### `/containers/v1/snapshots`
This endpoint is used to manage the creation and deletion of snapshots that are used for container environments.  The following methods will be supported against this endpoint.

GET `/containers/v1/snapshots?volume_id=413027010196420fb319834a79891db0`
 * Return all of the snapshots of the `volume_id` used for containers on the array.
 * This short form will only return snapshot id and name.
 * `volume_id` is mandatory

GET `/containers/v1/snapshots/detail?volume_id=413027010196420fb319834a79891db0`
 * Return all of the snapshots of the `volume_id` used for containers on the array.  
 * This detailed form returns all snapshot properties
 * `volume_id` is mandatory

GET `/containers/v1/snapshots/detail?volume_id=063b5de80e54af7a6b0000000000000000000000d0&name=mySnapshot`
 * Return all of the snapshots of the `volume_id` with `name` of `mySnapshot` used for containers on the array.  
 * This detailed form returns all snapshot properties

GET `/containers/v1/snapshots/{id}`
 * Return the snapshot identified by the given id.
 * This will return all snapshot properties.

POST `/containers/v1/snapshots`
 * Create a new snapshot.
 * The body must contain the new snapshot definition

DELETE `/containers/v1/snapshots`
 * Delete the snapshot identified
 * Should fail if there is an existing clone created from this snapshot.

## Object sets

| Object set | Path | Query Param | Operations | Actions |
| ---------- | ---- | ----------- | ---------- | ------- |
| tokens | /containers/v1/tokens | | post<br>delete | |
| nodes | /containers/v1/nodes | | post<br>delete | |
| volumes | /containers/v1/volumes | name | get<br>put<br>post<br>delete | publish<br>unpublish |
| snapshots | /containers/v1/snapshots | volume_id<br>name | get<br>post<br>delete | |

## Objects

| Objects | Attributes |
| ------- | ---------- |
| Token | id (string)<br>username (string) - mandatory<br>password (string) - mandatory<br>session_token (string) - used for all future authentication |
| Node | id (string)<br>uuid (string) - mandatory<br>name (string) - mandatory<br>iqns (list\<string\>) - mandatory if wwpns are not specified<br>wwpns (list\<string\>) - mandatory if iqns are not specified<br>networks (list\<string\>) - mandatory if iqns are specified |
| Volume | id (string)<br>name (string) - mandatory<br>size (number in bytes) - mandatory<br>base_snapshot_id (string) - used for cloning<br>volume_group_id (string)<br>clone (boolean) - true to create a clone<br>in_use (boolean) - true if the volume has an access control record; false otherwise (mandatory)<br>config (map[string]interface{}) |
| VolumeConfig | map[string]interface{} - vendor specific volume properties |
| PublishOptions | node_id (string) - mandatory<br>access_protocol ("fc" or "iscsi") |
| PublishInfo | <ul><li>Block<ul><li>serial_number (string) - mandatory</li><li>access_protocol ("fc" or "iscsi") - mandatory</li><li>target_name (string) - mandatory</li><li>target_scope (string) - mandatory</li><li>lun_id (number) - mandatory</li><li>discovery_ip (string) - mandatory</li><li>chap_user (string)</li><li>chap_password (string)</li></ul></li><li>VMDK<ul><li>pci_id (number)</li><li>unit_number (number)</li></ul></li><li>NFS<ul></ul><li>TBD</li></li></ul> |
| UnpublishOptions | node_id (string) - mandatory |
| Snapshot | id (string)<br>name (string) - mandatory<br>size (number) - in bytes<br>volume_id (string) - mandatory<br>volume_name (string)<br>creation_time (number) - in seconds (mandatory)<br>ready_to_use (boolean) - mandatory |

