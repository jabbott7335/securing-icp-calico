# Starting Up the Lab Environment

## Use these steps to power up your IBM Cloud Private cluster and prepare for the lab

Login to SkyTap.  Start the cluster by using the **Start** button in the upper left hand corner to start all the nodes

![image-20190206133536396](images/startallnodes.png)

Once all systems show **Running** status (this may take a few minutes), open the **client** system from within the SkyTap interface

![image-20190206133556674](images/clientnode.png)

> You can adjust the screen resolution of the SkyTap environment using the ![image-20190206135205001](images/fittoscreen.png) and ![image-20190206135221396](images/resize.png) buttons in the menu bar.

At the prompt, login using **sysadmin** with password of **passw0rd**

![image-20190206133755664](images/oslogin.png)

After you have logged in to the system:

- Open the **Firefox Web Browser** (link on the desktop) 
- Enter the URL https://10.10.1.10:8443 in the address bar to access the ICP dashboard
- Login with the credentials userid/password = **admin/passw0rd**

![image-20190206134448194](images/icplogin.png)

Next open another Browser tab and navigate to **ibm.biz\thinkcal**

> By opening the above link you will easily be able to copy commands from the lab into your SkyTap environment.  Copying and pasting using the SkyTap environment can become tedious.

Next open the **Terminal** application (link on the desktop)

Initialize the ICP CLI userid/password = **admin/passw0rd**

```
cloudctl login -a https://10.10.1.10:8443 --skip-ssl-validation
```

Select the **cloudcluster** account and the **default** namespace when prompted.

![image-20190206134821506](images/cliinit.png)

[Next continue on to Part 1 Securing Your Cluster with NetworkPolicy](./part-1-securing-your-cluster.md)
