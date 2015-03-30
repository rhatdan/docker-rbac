# Definining Privilege operation on Docker

We have begun an effort to describe Role Based Access Control in Docker.  The goal is to build 
an authorization/authentication database into docker for people who can communicate with the docker daemon.  Currently if you can communicate with the docker daemon you can gain full control over docker and the server machine that is running docker.  

I see that we can break this into three different realms of control to start.

1 Define existing containers and allow a user "admin" control over those containers only, preventing the user for escallating controls and touching other existing containers.

For containers that a user is allowed to "manage"
docker start/stop/exec/attach/inspect/export/kill/logs/pause/unpause/restart//save/stats/top/wait

2. Allow a user "admin" the ability to create new "unprivileged" containers.  We have to define what it means to create an unprivileged container.  This admin would not be allowed to interact with other existing containers.  Any container he creates will automatically get added to his list.

A second level would be to allow a user to run/create a container, without using any of the privileged operations.
run/create
--privilege
--cap-add (I don't see cap-drop as being a privileged operation)
--security-opt
--ipc=host/container
--net=host/ns/container
--pid=host
--user= /Unless it is his own UID.
Volume Mount? Maybe volume mount volumes owned by his UID?
docker exec/attach into a container running with one of the --privileged operations
Created containers automatically get added to the list of containers that this user/admin owns, he then gets all of the access defined in section 1.
Remove containers that the user created/pulled
docker rmi 
docker rm

3.Full access Admin.




