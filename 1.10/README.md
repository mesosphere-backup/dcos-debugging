## Debug common issues

**Exercise 1 - Horizontal Scaling App - Not enough resources**

1. Deploy an application and scale the instances
    ```
    dcos marathon app update /app-scaling instances=100
    dcos marathon app list
    dcos marathon deployment list
    ```

2. Look into the Marathon logs. Can you find the reason why the deployment is in waiting state?
    ```
    dcos node ssh --master-proxy --leader
    sudo journalctl -flu dcos-marathon | grep app-scaling
    ```

3. Reset the deployment by forcing the instances to be 1
    ```
    dcos marathon app update /app-scaling --force instances=1
    ```

**Exercise 2 - Vertical scaling - No matching resources**

1. Now increase the CPU allocation of the app
    ```
    dcos marathon app update /app-scaling cpus=100
    dcos marathon app list
    dcos marathon deployment list
    ```

2. Look into the Marathon logs. Can you find the reason why the deployment is in waiting state?
    ```
    dcos node ssh --master-proxy --leader
    sudo journalctl -flu dcos-marathon | grep app-scaling
    ```

3. Reset the deployment by forcing the cpu allocation to be 1
    ```
    dcos marathon app update /app-scaling --force cpus=1
    ```

**Exercise 3 - OOM Situations**

1. Deploy the file `app-oom.json`
    ```
    dcos marathon app add app-oom.json
    ```

2. Look into the Marathon logs. Can you find the reason why the application is failing?
    ```
    dcos node ssh --master-proxy --leader
    sudo journalctl -flu dcos-marathon | grep app-oom
    ```

3. Login to one of the agents the task failed on. You can see in the journal that the process got killed because of the cgroup memory violation.
    ```
    dcos node ssh --master-proxy --mesos-id=$(dcos task app-oom --json | jq -r '.[] | .slave_id')
    journalctl -f _TRANSPORT=kernel
    ```
4. Remove the application
    ```
    dcos marathon app remove /app-oom
    ```

**Exercise 4 - Docker Images**

1. Deploy the file `dockerimage.json`
    ```
    dcos marathon app add dockerimage.json
    ```

    The app will fail over and over again without any logs.

2. Can you find the reason why the application is failing?
     ```
     docker pull noimage:idonotexist
     ```

**Exercise 5 - Debugging Web Applications**

Prerequisite:

* [Marathon-LB](https://dcos.io/docs/1.8/usage/service-discovery/marathon-lb/) installed on all public nodes
* [IP of your public agent](https://docs.mesosphere.com/1.10/administering-clusters/locate-public-agent/)
* [DC/OS CLI](https://docs.mesosphere.com/1.10/cli/) installed


We want to deploy a webserver listening on port `3030`, and even set the service port to `10105` so we can reach it via marathon-lb.

1. Deploy the file `webserver1.json`
    ```
    dcos marathon app add webserver1.json
    ```
    This will start the webserver on Port `3030` and also defines a label that marathon-lb uses to bind to the service port `10105` on the public agent.

2. Try to reach it via marathon-lb

    We try to go to `http://<public_agent>:10105/` in our browser, but the webapp does not show up. So let us try to figure out why this is not working.

3. Check marathon-lb/HAProxy
    In order to see if/how marathon-lb has picked up the app, we look at it the HAProxy stats.  These are available on all nodes where marathon-lb is installed: `http://<public_ip>:9090/haproxy?stats`

    a. The page should look similar to the screenshot below. So we can see that marathon-lb has picked up the app. And `10.0.0.212:7727` is used as backend for the service port frontend `10105`.

    ![HAProxy stats](img/HAProxy-stats.png "HAProxy stats")

4. Check whether we can reach that backend from within the cluster
    ```
    dcos node ssh --leader --master-proxy
    curl 10.0.0.212:7727
    ```
    We should see something similar to:
   `curl: (7) Failed to connect to 10.0.0.212 port 7727: Connection refused`

    So seemingly the app isn't serving on `10.0.0.212:7727`, which is the address used by marathon-lb.

5. Check on which port our app is listening

    Let us revisit our application definition:
    ```
    "cmd": "echo 'Hello DC/OS' > index.html && python -m http.server 3030"
    ```
    So let us check whether we can reach our application on port 3030

    ```
    dcos node ssh --leader --master-proxy
    curl 10.0.0.212:3030
    Hello DC/OS
    ```

    So we have identified the problem: marathon-lb tries to redirect to the random assigned port `7727`, while our app is listening on port `3030`. Note that, port `3030` is not allocated (as it is not in the app specification), so it might actually happen that another app is already using that port. Also that implies we can only run a single instance per node.

6. Fixing the problem
    We have two options for fixing this problem

    a) Have the application listen to the random port
    DC/OS will give you the `PORT0` environment variable holding the first random port assigned to your app. So we could change our webserver to listen to that port:
    ```
    "cmd": "echo 'Hello DC/OS' > index.html && python -m http.server PORT0"
    ```

    b) Sometimes using a random port is not possible, as the application needs to listen to a fixed port (e.g., `3030`). In that case we can run our docker container in bridge mode (see [here](https://dcos.io/docs/1.8/usage/marathon/ports/) for details on host versus bridge mode). That would mean inside the container network the application can use port `3030`, which is mapped to a random port on the host.

    See `webserver2.json` for more details.


**Exercise 6 - Debugging nginx using `dcos task exec`**

Prerequisite:

* [IP of your public agent](https://docs.mesosphere.com/1.10/administering-clusters/locate-public-agent/)
* [DC/OS CLI](https://docs.mesosphere.com/1.10/cli/) installed

We want to deploy a nginx webserver, but cannot reach nginx (depite it running on a public agent).

1. Deploy the file [`nginx.json`](https://github.com/dcos-labs/dcos-debugging/blob/master/1.10/nginx.json).
    ```
    dcos marathon app add https://raw.githubusercontent.com/dcos-labs/dcos-debugging/master/1.10/nginx.json
    ```
    This will start the nginx on the public agent.

2. Try to reach via the public agent

    We try to go to `http://<public_agent>/` (default port is 80) in our browser, but the webapp does not show up. So let us try to figure out why this is not working.

3. Check that app is running

    Let us first check whether app is really running, this can be done either via UI or the CLI:

    ```
    $ dcos task
    NAME   HOST        USER  STATE  ID                                          MESOS ID
    nginx  10.0.6.146  root    R    nginx.cd3b7ef1-0e9e-11e8-b971-92f543ad2ed3  6a2b8b96-82bb-4ee8-b26a-39d3bb553a73-S0
    ```

    Looks like everything is ok here (State equal *R*unning). So let us continue..

4. Check whether we can reach the instance from within the cluster
    ```
    dcos node ssh --leader --master-proxy
    curl <public_agent>
    ```
    We should see something similar to:
   `curl: (7) Failed to connect to ...: Connection refused`

    If we are on AWS it might be wort trying to curl both the interal and external IP of the the public agent.
    
5. Check nginx logs

    ```
    $ dcos task log nginx
    Executing pre-exec command '{"arguments":["mesos-containerizer","mount","--help=false","--operation=make-rslave","--path=\/"],"shell":false,"value":"\/opt\/mesosphere\/active\/mesos\/libexec\/mesos\/mesos-containerizer"}'
    Executing pre-exec command '{"arguments":["mount","-n","--rbind","\/var\/lib\/mesos\/slave\/slaves\/6a2b8b96-82bb-4ee8-b26a-39d3bb553a73-S0\/frameworks\/6a2b8b96-82bb-4ee8-b26a-39d3bb553a73-0001\/executors\/nginx.cd3b7ef1-0e9e-11e8-b971-92f543ad2ed3\/runs\/3f0846e3-9c40-430f-82a6-b217de90ae15","\/var\/lib\/mesos\/slave\/provisioner\/containers\/3f0846e3-9c40-430f-82a6-b217de90ae15\/backends\/overlay\/rootfses\/996094c4-9a28-47d7-9678-ccd3c2458bd1\/mnt\/mesos\/sandbox"],"shell":false,"value":"mount"}'
    Executing pre-exec command '{"shell":true,"value":"mount -n -t proc proc \/proc -o nosuid,noexec,nodev"}'
    Executing pre-exec command '{"arguments":["mount","-n","-t","ramfs","ramfs","\/var\/lib\/mesos\/slave\/slaves\/6a2b8b96-82bb-4ee8-b26a-39d3bb553a73-S0\/frameworks\/6a2b8b96-82bb-4ee8-b26a-39d3bb553a73-0001\/executors\/nginx.cd3b7ef1-0e9e-11e8-b971-92f543ad2ed3\/runs\/3f0846e3-9c40-430f-82a6-b217de90ae15\/.secret-ae0b7c8c-19bb-436e-9bb5-09036ff3bdee"],"shell":false,"value":"mount"}'
    Changing root to /var/lib/mesos/slave/provisioner/containers/3f0846e3-9c40-430f-82a6-b217de90ae15/backends/overlay/rootfses/996094c4-9a28-47d7-9678-ccd3c2458bd1
    ```

    Unfortunatly, still nothing helpful...

6. Exec into task

    Next, let us interactivly debug the task by launching a shell into the container environment:
    Note, `dcos task exec currently only works with UCR containers. If you are using the docker runtime,
    you will need too ssh to that node where the task is running and then use `docker exec`.

    ```
    dcos task exec -it nginx bash
    ```

    Then from inside the container environment, we would check the running processes.


    ```
    # ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root         1  0.1  0.1 106468 31368 ?        Ss   21:19   0:00 /opt/mesosphere/active/mesos/libexec/mesos/mesos-containerizer la
    root         6  0.2  0.2 770436 34092 ?        Sl   21:19   0:00 mesos-executor --launcher_dir=/opt/mesosphere/active/mesos/libexe
    root        16  0.0  0.0   4336   728 ?        Ss   21:19   0:00 sh -c sleep 100000
    root        17  0.0  0.0   4236   712 ?        S    21:19   0:00 sleep 100000
    root        19  0.2  0.1 106488 31200 ?        Ss   21:19   0:00 /opt/mesosphere/active/mesos/libexec/mesos/mesos-containerizer la
    root        20  0.0  0.0  20240  3292 ?        S    21:19   0:00 bash
    root        23  0.0  0.0  17500  2128 ?        R+   21:19   0:00 ps aux
    ```

    It seems nginx is not running (but instead the sleep command we supplied in nginx.json).

    Let us double check:

    ```
    # /usr/sbin/service nginx status
    nginx is not running ... failed!

    # /usr/sbin/service nginx start
    ```

    If we now check again `http://<public_agent>/`, we should see `Welcome to nginx!`.
    
7. Update the app definition
   
   Even though it seems that everything is fine now, the next crucial step would be to update your `nginx.json` by either removing the cmd override or explicitly starting nginx yourself. Otherwise, the failure will occure again as soon as the app is redeployed or scaled.

