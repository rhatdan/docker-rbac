# Docker Access Control
At present, anyone who has access to the Docker daemon has the ability to
perform every operation Docker can. At present, this is solved by ensuring
that Docker can only be accessed by root, and so those using it already have
full administrator access to the system. This is a very limited permissions
model, and one we would like to replace with a more granular system.

## Prerequisite: Authentication
This document will focus specifically on the Authorization side of access
control. Authentication for Docker is already being discussed in
docker/docker#13697 with a focus on providing Kerberos authentication GSSAPI.
This document assumes that some authentication solution is in place already,
and can be leveraged to identify the user performing specific operations on
the Docker daemon.

## Authorization: General Case
There is presently an issue for discussing authorization in the Docker daemon,
docker/docker#14674. In this issue, it's proposed to use the Docker plugin
framework to provide modular authentication plugins. The plugin will be
provided all pertinent information on incoming requests to the Docker remote
API, including the subject performing the request, any information on the
container or image the request is acting on, and all arguments used in the
request. The plugin will make a decision on whether the request is authorized or
not, and return this to the daemon.

## Role-Based Access Control
We propose a simple role-based access control model for Docker which can be
implemented as an authentication plugin. We define a number of permissions
which cover certain operations (on the Daemon in general, on individual
Containers, and on individual Images). These permissions are then assigned to
several roles grouped into a rough hierarchy. Finally, we discuss constraints
which can be used to further restrict the access of individual subjects or
roles.
<!---
This section may be too short. Consider expanding
-->

### Daemon Permissions
There are not many operations on the Docker Daemon that do not act on either
Containers or Images. These are usually very basic operations (retrieving basic
information on the daemon such as its version). Consequently, they are grouped
under a single permission, called Daemon Access.
<!---
Again, may be too short, consider expanding
-->

### Container Permissions
These permissions grant users permission to act on containers.

<!---
Following paragraph wording is awkward, consider revision
-->
Privileged containers (defined here as any container with reduced confinement,
including added capabilities, disabled SELinux or Apparmor labelling, lack of
certain namespaces) are affected by a different, but identical, set of
permissions - meaning that the View privileges for normal containers, and
privileged containers, are different. A user could have all privileges for
normal containers, but only a limited set of privileges on privileged
containers.

Some Container permissions act upon individual containers, giving permission to
examine or modify them. Others act on no specific container, but produce a
container as a result. We term the former Individual Container Permissions, and
the latter Global Container permissions; both are listed below.

Global Container Permissions:
* Create
* List

Individual Container Permissions:
* View - Metadata, Logs, Stats
* Delete
* Commit
* State - Start, Stop, Restart, Pause
* Access - Copy from, Exec into, Attach to, Send signals to, Diff against image

### Image Permissions
These permissions grand users permission to act on images.

Some container permissions provide overlap with image permissions. For example,
an API call to create a container must verify that the subject making the call
has the Create privilege on Containers and the Use permission on the image the
container is being created from.

As with Container permissions, some of these permissions act on an individual
image, others on a group of images. The same terminology will be used here. Both
Individual and Global permissions are listed below.

Global Image Permissions:
* Import - Includes building from Dockerfile
* List

Individual Image Permissions:
* View - Metadata, History
* Use - Create containers from
* Push - Also includes tag
* Pull
* Delete
* Export - Includes export of containers created from an image

The Import and Export permissions are treated with special care in the RBAC
hierarchy, as they are prime candidates for the infiltration of malicious images
or exfiltration of confidential data.

### Roles
The permissions listed above will be grouped together into logical roles. There
are two rough groups of roles defined, one for Developers and another for Operations.
Developer roles are granted increased privileges regarding the creation and
deletion of images, to facilitate developing images. Operations roles are focused
on managing the lifecycles of production containers. A single role atop the
hierarchy, Docker Administrator, provides global privileges on the local daemon,
identical to the access a root user would have today. All requests made by users
authenticated as root will use this role.

Operations Roles consist of Basic Operator and Advanced Operator. All
Operations roles are granted the Daemon Access permission to view basic
information on the Docker Daemon.

Basic Operator: Can Create and List containers. Can View, modify State, or
Access non-privileged containers. Can List images, and View and Use individual
images.

Advanced Operator: Can Create and List containers. Can View, modify State,
Access, Delete, and Commit containers. Can List images globally, and
Use, and Pull individual images.

There is only a single Developer role defined at present, called Image
Developer. Image Developers are granted the Daemon Access permission globally.
They have the Create and List permissions globally on containers, and View,
modify State, Access, Delete, and Commit on individual containers. Can List
and Import images globally, and View, Use, Push, Pull, Delete, and Export
individual images.

The Docker Admin role has unrestricted access to the Docker daemon.

### Additional Constraints
The permissions and roles defined above should cover most typical situations,
but some more unusual scenarios may require additional constaints be placed on
users or roles. Suppose, for example, it was desired to make subjects in a
Developer role unable to perform State operations on containers created by
subjects in an Ops role, to prevent production services from being interfered
with on a system both roles have access to. For another example, suppose that
the it is desired that the Basic Operator role is only able to create containers
if they use a non-root user. The above RBAC hierarchy cannot handle these
scenarios, but they are obviously cases which should be handled.

Here, we introduce the concept of constraints, which allow further restrictions
to be placed on roles or individual subjects. A constraint acts only on
operations for a specific class of permissions

The Docker Admin role grants unrestricted access to the Docker daemon and as
such is not affected by constraints.

Constraints on individual images and containers can be supported by access lists
granting permissions on them to subjects or roles.

A list of sample proposed constraints is below:
* Developer roles should not be able to Delete, Access, or modify State of
containers created by Operations roles.
* Operations roles should only be able to use images which have been tagged as
Production. Images created by Developer roles will not by default be tagged as
such.
* Subjects in the Basic Operator role can only Access and modify State of
containers they themselves created, unless they are explicitly granted
permission to access another container.
* Subjects in the Advanced Operator role and Developer roles cannot Access,
Delete, or modify State of containers created by subjects in the Docker Admin
role.
