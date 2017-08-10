# vagrant-drbd

## Installation
- please install virtul box first
- `git clone git@github.com:imyoungyang/vagrant-drbd.git`
- `cd vagrant-drbd`
- `vagrant up`

### After installation

[](https://gyazo.com/8e37c5b83e0052baa3d2c88b3126de9d)

[![https://gyazo.com/8e37c5b83e0052baa3d2c88b3126de9d](https://i.gyazo.com/8e37c5b83e0052baa3d2c88b3126de9d.png)](https://gyazo.com/8e37c5b83e0052baa3d2c88b3126de9d)

The status is 

    `offline/quorum vote ignored, pending actions: adjust connections`

It needs to fix it.

### How to fix the status?

- `vagrant ssh drbd2`
- `sudo su`
- `drbdmanage howto-join drbd0`

It will output the following message:

<blockquote>

        IMPORTANT: Execute the following command only on node drbd0!
        drbdmanage join -p 6999 192.168.100.60 1 drbd2 192.168.100.62 0 4/3tow9h/Qw9YWQapWOK
        Operation completed successfully

</blockquote>

- `ssh drbd0 "drbdmanage join -p 6999 192.168.100.60 1 drbd2 192.168.100.62 0 4/3tow9h/Qw9YWQapWOK"`

Repeat the command `drbdmanage howto-join drbd1` and execute the command as the message:

- `ssh drbd1 "drbdmanage join -p 6999 192.168.100.61 2 drbd2 192.168.100.62 0 4/3tow9h/Qw9YWQapWOK"`

### End result of cluster

[![https://gyazo.com/ed73fcd555c510e9f18e93f2ba15ed6f](https://i.gyazo.com/ed73fcd555c510e9f18e93f2ba15ed6f.png)](https://gyazo.com/ed73fcd555c510e9f18e93f2ba15ed6f)

# Create the block resource

- `drbdmanage add-volume esdata 1GB --deploy 3`
- `drbdmanage list-volumes`  you will see the volume created
- `lsblk` can see the new block

[![https://gyazo.com/88eea58e772d9db9adf57e9f257e18af](https://i.gyazo.com/88eea58e772d9db9adf57e9f257e18af.png)](https://gyazo.com/88eea58e772d9db9adf57e9f257e18af)

- Also you can check other machines. The volume is replicated to drbd0 and drbd1.

[![https://gyazo.com/aaed368cd08aecd4a26c4d3c6e153742](https://i.gyazo.com/aaed368cd08aecd4a26c4d3c6e153742.png)](https://gyazo.com/aaed368cd08aecd4a26c4d3c6e153742)

# Format disk and turn on the NFS service

- `mkfs.xfs /dev/drbd100`
- `mkdir -p /mnt/esdata`
- `mount /dev/drbd100 /mnt/esdata`

## turn on NFS and change the user group for elasticsearch

- `chown -R elasticsearch:elasticsearch /mnt/esdata`
- `chmod 755 /mnt/esdata`
- `echo "/mnt/esdata 192.168.100.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)" >> /etc/exports`
- `exportfs -a`

## Test NFS Client

- `vagrant ssh drbd1`

[![https://gyazo.com/e8b3f2f4880654d3194c5d970eac3054](https://i.gyazo.com/e8b3f2f4880654d3194c5d970eac3054.png)](https://gyazo.com/e8b3f2f4880654d3194c5d970eac3054)

# Unmount and mount in different machines

## umount in drbd1

- [root@drbd1 vagrant]# umount /mnt/data
- [root@drbd1 vagrant]# ls /mnt/data



