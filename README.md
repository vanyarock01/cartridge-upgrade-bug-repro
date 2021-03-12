# Cartridge upgrade bug reproducer

Used
- Tarantool **v2.6.2**
- Cartridge upgrade from **v2.0.1** -> **v2.4.0**
- Cartridge CLI **v2.6.0**

## Steps

1. Clone this repo
    ```
    https://github.com/vanyarock01/cartridge-upgrade-bug-repro.git
    ```

2. Go to application directory

    ```
    $ cd repro-app/
    ```

3. Build app
    ```
    $ cartridge build
    ```
    
4. Start instances
    ```
    $ cartridge start -d
    ---
    • repro-app.s1-master... OK
    • repro-app.s2-replica... OK
    • repro-app.s1-replica... OK
    • repro-app.s2-master... OK
    ```
    
5. Setup replicasets
    ```
    $ cartridge replicasets setup --bootstrap-vshard
    ---
    • Set up replicasets described in /home/user/Workspace/Tarantool/cartridge-upgrade-bug-repro/repro-app/replicasets.yml
    • s-1... CREATED  
    • s-2... CREATED  
    • Replicasets are set up successfully
    • Vshard is bootstrapped successfully
    ```
    
We now have a cluster on cartridge v2.0.1. [localhost:8081](localhost:8081)

![](https://i.imgur.com/0EeYHBs.png)


6. Update cartridge version in rockspec file
    ```
    $ sed -i 's/cartridge\ ==\ 2.0.1-1/cartridge\ ==\ 2.4.0-1/' repro-app-scm-1.rockspec
    ```

7. Build app again
    ```
    $ cartridge build
    ```
    
8. Restart replicas
    ```
    $ cartridge stop s1-replica && cartridge stop s2-replica
    --
    • repro-app.s1-replica... OK
    • repro-app.s2-replica... OK
    $ cartridge start -d s1-replica && cartridge start -d s2-replica
    --
    • repro-app.s1-replica... OK
    • repro-app.s2-replica... OK
    ```
    
9. Switch master of replicaset s2
    ![](https://i.imgur.com/0WUkyZo.png)

10. Try to switch master of replicaset s1
    ![](https://i.imgur.com/abVhCX4.png)


Something is broken. Let's check the status of the members.

```
$ tarantoolctl connect admin:repro-app-cluster-cookie@localhost:3301
connected to localhost:3301
localhost:3301> require('membership').members()
---
- localhost:3302:
    payload: {'state': 'RolesConfigured', 'uuid': '202cd36d-7ae6-4e86-a6bd-32746631d8ff',
      'alias': 's1-replica'}
    uri: localhost:3302
    clock_delta: -89.5
    status: alive
    incarnation: 26
    timestamp: 1615552827964552
  localhost:3304:
    payload: {'state': 'RolesConfigured', 'uuid': 'e329ddf4-a94c-4840-bdd0-3f3c63d37725',
      'alias': 's2-replica'}
    uri: localhost:3304
    clock_delta: -13
    status: alive
    incarnation: 26
    timestamp: 1615552824962713
  localhost:3303:
    payload: {'state': 'ConfiguringRoles', 'uuid': '997ad11d-8ca1-4aeb-9e0b-2391aa86a23f',
      'alias': 's2-master'}
    uri: localhost:3303
    clock_delta: -24
    status: alive
    incarnation: 16
    timestamp: 1615552828965202
  localhost:3301:
    payload:
      state: RolesConfigured
      uuid: 50789568-a1a0-4dad-9acf-78e104423eea
      alias: s1-master
    uri: localhost:3301
    clock_delta: 34.5
    status: alive
    incarnation: 17
    timestamp: 1615552825962790
...

localhost:3301> 
```

Instance `s2-master` is stuck in a state `ConfiguringRoles`.

After restarting the `s2-master` instance, it returns to the `RolesConfigured` state, but if you try to switch the master again, it will stick again in the `ConfiguringRoles`.