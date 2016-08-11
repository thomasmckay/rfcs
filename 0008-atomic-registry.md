- Feature Name: Katello and Atomic Registry Integration
- Start Date: 2016-08-11
- RFC PR:
- Issue:

Table of Contents
 * [Summary](#summary)
 * [Benefits](#benefits)
 * [Example](#example)
 * [User Authentication and Role Based Access Control](#userauthentication)
 * [Installation](#installation)
 * [Capsules](#capsules)

# Summary
[summary]: #summary

To facilitate a more natural feel for working with containers and [Katello](https://katello.org), [Atomic Registry](http://www.projectatomic.io/registry/) will become an optional integration point. This true registry will provide the full set of features required by users, while allowing Katello to continue to act as a container repository. Integration between the two will allow images in the Katello content management lifecycle to be directly linked to images in Atomic Registry projects, and vice versa.

# Benefits
[benefits]: #benefits

The docker registry API provided by Katello does not support 'docker push' commands so it is effectively a read-only source of images. With the inclusion of Atomic Registry, the full set of docker commands will be available.

When Katello imports a docker image into a product repository, it is given a name that includes the organization and the product. In addition, when this product repository is added to a content view and promoted, the image name will include this information as well. While this is necessary to uniquely identify these images within Katello it can be confusing to users expecting to access images by the original name. With Atomic Registry these images can be provided as their original names, if desired.


# Example
[example]: #example

In this example, Katello's registry is at katello.example.com:5001 and Atomic Registry is at katello.example.com:5000.

1. An image is synced into a Katello product repository. The synced image is available directly from Katello via its contextual name which includes the organization, product, and source registry.

   ```shell
   katello.example.com:5001/examplecorp-docker_io-thomasmckay_hammer
   ```
   :sparkles: The project and imagestream names may be set at repository creation time. If this name is later changed, the existing project in Atomic Registry will remain.   
   :question: If this name is later changed, the existing project in Atomic Registry will remain.  
   :question: If a project/imagestream already exists in Atomic Registry, it will have all (or some?) tags overwritten.  
   :question: Deleting a repository in Katello will leave the project/imagestream in Atomic Registry.
   
1. At repository creation, a project is created in Atomic Registry to match the namespace and an image stream is added with the original image name. The created imagestream will reference Katello's contextual image name.

   ```
   Name: hammer
   Project: thomasmckay
   Pull from: katello.example.com:5001/examplecorp-docker_io-thomasmckay_hammer
   ```
   :sparkles: Pushing to this project will be disabled to normal users as changes synced into Katello that are then automatically pushed to Atomic Registry has the potential to overwrite changes made there.

1. Users may then pull images from Atomic Registry to work with. Changes should be pushed back to their own projects. This user project may then be synced into Katello for use there. In turn, the product repository in Katello may itself push back to Atomic Registry.
   ```
   Katello                                Atomic Registry
   -------                                ---------------
   *Image synced from docker.io*
                                          *Image created with original name*
   examplecorp-docker_io-
        thomasmckay_hammer:latest     ->  thomasmckay/hammer:latest
                                          *Modify, tag, then push image*
                                          # docker tag thomasmckay/hammer:latest sallydev/hammer:latest
                                          # docker push sallydev/hammer
   *Image synced from Atomic Registry*
   examplecorp-registry_example_com-  <-  
        sallydev_hammer:latest
                                          *Image created with new tag*
                                      ->  thomasmckay/hammer:v1
   ```

   In this way, a user may pull an image (eg. thomasmckay/hammer:latest), manipulate it, then make it available with a new tag (eg. thomasmckay/hammer:v1). More than one Katello repo may push to a single project/imagestream, each supplying their own tags.  
   :question: Is this too complicated and unnecessary? Atomic Registry doesn't support this.
   :sparkles: Specify which tags are pushed to Atomic Registry and also which tags are synced into Katello.  
   :sparkles: When creating a repo, tags should be configurable for sync or not, push or not, automatic or manual push.  
   :sparkles: Defaults for above should be set per repo to handle new tags as they appear from a sync.  
   :sparkles: Repositories in content views should also be configurable in this way.  
   :question: Should tags be configurable? Could Sally's hammer:latest be synced as hammer:sally? Could a Katello tag hammer:13-test be pushed to Atomic Registry as hammer:latest-test?
   :sparkles: Roles should be granular to the tag level to determine who may sync into or push out of a Katello repository.  
   :sparkles: A central location should be available listing all Atomic Registry project/imagestream and their dependent repositories and settings. All should be searchable and configurations changed from there.  

# User Authentication and Role Based Access Control (RBAC)
[userauthentication]: #userauthentication

Both Katello and Atomic Registry have the concept of a user, as well as the ability to assign roles and permissions to control access to actions and resources. At this time they are separate systems and as such there is no model for a "single sign on" (SSO) that would allow a user in the Katello UI to also be logged into the Atomic Registry UI.

### User Authentication

Katello uses either its own builtin authentication or LDAP ([Foreman LDAP Authentication](https://theforeman.org/manuals/1.12/index.html#4.1.1LDAPAuthentication)). Atomic Registry leverages openshift/origin authentication and is able to run in several modes, including LDAP ([Configuring Authentication and User Agent](http://docs.projectatomic.io/registry/latest/install_config/configuring_authentication.html)). Using LDAP would allow a closer feel to SSO but only because the user names would be consistent. Each tool would need those users configured individually.

### Role Based Access Control (RBAC)


Katello has a rich, fine-grained RBAC model ([Roles and Permissions](https://theforeman.org/manuals/1.12/index.html#4.1.2RolesandPermissions)) that ties resources to actions. For example, a permission to view and sync products could be assigned to a user to allow them to bring in updated content but that would prevent the creation or deletion of products. Further refinement is possible by adding a search filter.

The following use cases will be necessary:
 * As a Katello admin, I wish to limit which users may update all docker repositories in a product.
 * As a Katello admin, I wish to limit which users may update specific repositories in a product.
 * As a Katello admin, I wish to limit which users may publish a new version of a contentview.
 * As a Katello admin, I wish to limit which users may promote a contentview to a lifecycle environment.
 * As a Katello admin, I wish to put restrictions on who may sync new tags to Atomic Registry.
 * As a Katello admin, I wish to limit which uesrs may edit the settings on a docker repository.

# Installation
[installation]: #installation

### All-in-one Server

```shell
# hostname -f
katello.example.com

# ...katello install instructions...
  ... configures Katello registry at katello.example.com:5001
  ... configures Atomic Registry at katello.example.com:5000
  ... configures Atomic Registry UI at katello.example.com:9091

# yum install -y atomic
...
Installed:
  atomic.x86_64 1:1.10.5-7.el7 
  
# atomic install projectatomic/atomic-registry-install katello.example.com
Using default tag: latest
Trying to pull repository docker.io/projectatomic/atomic-registry-install ...
...

# systemctl enable --now atomic-registry-master.service
# docker ps
CONTAINER ID        IMAGE                COMMAND
f58273e3fedc        cockpit/kubernetes   "/usr/libexec/cockpit"
8cdb51b35239        openshift/origin    "/usr/bin/openshift s"

# /var/run/setup-atomic-registry.sh katello.example.com
Launch web console in browser at https://katello.example.com:9090
By default, ANY username and ANY password will successfully authenticate.

```
 
### Separate Atomic Registry Server
```
...some good stuff...
```

# Capsules
[capsules]: #capsule

