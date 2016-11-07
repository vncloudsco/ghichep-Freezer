#Một số ghi chú khi tìm hiểu code

## 1. Điều chỉnh kích thước từng segment upload lên Swift (Byte), thực hiện trên node client
```
vim /usr/local/lib/python2.7/dist-packages/freezer/openstack/osclients.py

def download_image(self, image):
        """
        ...
        return utils.ReSizeStream(d, image.size, 1000000) #Sua thanh kich thuoc mong muon

 ```

## 2. Điều chỉnh kích thước chunk download từ Swift (Byte), thực hiện trên node client
```
vim /usr/local/lib/python2.7/dist-packages/freezer/openstack/restore.py

def _create_image(self, path, restore_from_timestamp):
        """
        ...

        stream = swift.get_object(self.container, "%s/%s" % (path, backup),
                                  resp_chunk_size=10000000) #Sua thanh kich thuoc mong muon
        ...

```

## 3. Backup metadata sau khi thực hiện backup VM

```
freezer-agent --nova-inst-id cfa52799-3d89-4895-92fc-73d0c39f2907 --path-to-backup /root/adminv3.sh --debug --container vm_cr2 --backup-name vm_long_snap --storage swift --log-file /root/logvmha


Backup metadata received: {"ssh_port": 22, "consistency_checksum": "", "curr_backup_level": 0, "backup_name": "vm_long_snap", "container": "vm_cr2", "compression": "gzip", "dry_run": "", "hostname": "controller", "storage": "swift", "vol_snap_path": "/root/adminv3.sh", "os_auth_version": "", "client_os": "linux2", "time_stamp": 1478349463, "container_segments": "", "ssh_username": "", "path_to_backup": "/root/adminv3.sh", "ssh_key": "", "proxy": "", "always_level": "", "max_level": "", "backup_media": "nova", "ssh_host": "", "mode": "fs", "fs_real_path": "/root/adminv3.sh", "action": "backup", "client_version": "3.0.0", "log_file": "/root/logvmha"}
```

## 4. Fix bug không xóa Snapshot Volume sau khi backup volume Snapshot (sử dụng backend Ceph và đặt `rbd_flatten_volume_from_snapshot = false`)
```
vim /usr/local/lib/python2.7/dist-packages/freezer/openstack/backup.py
def backup_cinder_by_glance(self, volume_id):
    ...
    LOG.debug("Deleting temporary volume")
    cinder.volumes.delete(copied_volume)
    LOG.debug("Deleting temporary snapshot")
    client_manager.clean_snapshot(snapshot)
    LOG.debug("Deleting temporary image")
    client_manager.get_glance().images.delete(image.id)
    ...
```

## 5. Fix bug không xóa VM Snapshot sau khi Backup lên Swift xong 
*Nguyên nhân do sử dụng Ceph làm Backend Glance, Glance api v2 không thể xóa image (Image status owner: None), do đó phải dùng Glance api v1 để xóa*

```
vim /usr/local/lib/python2.7/dist-packages/freezer/openstack/osclients.py

def __init__(self, auth_url, auth_method='password', **kwargs):
    ...
    self.glancev1 = None
    ...

def get_glance_v1(self):
    """
    Get glanceclient instance
    :return: glanceclient instance
    """
    if not self.glancev1:
        self.glancev1 = self.create_glance_v1()
    return self.glancev1
    ...

def create_glance_v1(self):
    """
    Use pre-initialized session to create an instance of glance client.
    :return: glanceclient instance
    """
    if 'endpoint_type' in self.client_kwargs.keys():
        self.client_kwargs.pop('endpoint_type')
    if 'insecure' in self.client_kwargs.keys():
        self.client_kwargs.pop('insecure')
    self.glancev1 = glance_client('1', session=self.sess,
                                **self.client_kwargs)
    return self.glancev1

```

```
vim /usr/local/lib/python2.7/dist-packages/freezer/openstack/backup.py

def backup_nova(self, instance_id):
    ...
    glancev1 = client_manager.get_glance_v1()
    glancev1.images.delete(image.id)
    ...
``` 

## 6. Fix bug không xóa Image sau khi restore VM và volume 
*Nguyên nhân do sử dụng Ceph làm Backend Glance, Glance api v2 không thể xóa image (Image status owner: None), do đó phải dùng Glance api v1 để xóa*
```
vim /usr/local/lib/python2.7/dist-packages/freezer/openstack/restore.py
    def restore_cinder_by_glance(self, volume_id, restore_from_timestamp):
        ...
        self.client_manager.get_glancev1().images.delete(image)
        ...
    def restore_nova(self, instance_id, restore_from_timestamp,
                     nova_network=None):
        glancev1 = self.client_manager.create_glancev1()
        glancev1.images.delete(image.id)
        ...

```

