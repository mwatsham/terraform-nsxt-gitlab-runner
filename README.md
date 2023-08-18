# terraform-nsxt-gitlab-runner
Custom GitLab runner image for Terraform NSXT IAC.

Used for to improve pipeline performance and minimise reliance on a external download of a Terraform provider/plugin. Basically, this is performing the same initialisation steps that the GitLab runner will do on execution so speeds up the initialisation step in the pipeline.

# Automated image build
WIP

# Manual steps
These are the steps for creating the custom Terraform GitLab runner image using Docker focusing on pre-installing the [NSXT Terraform provider](https://registry.terraform.io/providers/vmware/nsxt/latest/docs).

Fetch the latest GitLab Hashicorp Terraform image - https://gitlab.com/gitlab-org/terraform-images/container_registry
NB: It is assumed that Docker in available on the preparation environment.
1. Pull down required GitLab Terraform container image...
   ```
   $ docker image pull registry.gitlab.com/gitlab-org/terraform-images/releases/1.1:latest --platform linux/amd64
    latest: Pulling from gitlab-org/terraform-images/releases/1.1
    9b3977197b4f: Pull complete
    de9ce4f92ce8: Pull complete
    f8787742e1b4: Pull complete
    3b40c7d79d2d: Pull complete
    Digest: sha256:06c9e5a3877c0ed5a217f1b91aaacf49ce9332f5f0f301d8e2ef0e7ce853f3ec
    Status: Downloaded newer image for registry.gitlab.com/gitlab-org/terraform-images/releases/1.1:latest
    registry.gitlab.com/gitlab-org/terraform-images/releases/1.1:latest
   ```
2. Verify image...
   ```
   $ docker image ls registry.gitlab.com/gitlab-org/terraform-images/releases/1.1:latest
    REPOSITORY                                                     TAG       IMAGE ID       CREATED        SIZE
    registry.gitlab.com/gitlab-org/terraform-images/releases/1.1   latest    dc06d5676ca3   17 hours ago   88.1MB
   ```
3. Spin up container from image and attach to container...
   ```
   $ docker run --name test -it registry.gitlab.com/gitlab-org/terraform-images/releases/1.1 /bin/sh
   ```
4. Once attached to running container...

    a. Create Terraform provider configuration...
    ```
    # mkdir -p ~/.terraform.d/plugins
    # cd /tmp
    # vi provider.tf
    terraform {
      required_providers {
        nsxt = {
          source = "vmware/nsxt"
        }
      }
    }
     
    provider "nsxt" {
      allow_unverified_ssl  = true
      max_retries           = 10
      retry_min_delay       = 500
      retry_max_delay       = 5000
      retry_on_status_codes = [429]
    }
    ```

    b. Initialise Terraform environment...
    ```
    /tmp # terraform init  
 
    Initializing the backend...
     
    Initializing provider plugins...
    - Finding latest version of vmware/nsxt...
    - Installing vmware/nsxt v3.2.5...
    - Installed vmware/nsxt v3.2.5 (signed by a HashiCorp partner, key ID 6B6B0F38607A2264)
     
    Partner and community providers are signed by their developers.
    If you'd like to know more about provider signing, you can read about it here:
    https://www.terraform.io/docs/cli/plugins/signing.html
     
    Terraform has created a lock file .terraform.lock.hcl to record the provider
    selections it made above. Include this file in your version control repository
    so that Terraform can guarantee to make the same selections by default when
    you run "terraform init" in the future.
     
    Terraform has been successfully initialized!
     
    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.
     
    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.
    ```

    c. Move retrieved Terraform provider to runner working area and detach from container...
    ```
    # mv /tmp/.terraform/providers/registry.terraform.io ~/.terraform.d/plugins
    # rm -rf /tmp/.terraform
    # exit
    ```
5. Find container ID from previously run container...
   ```
    $ docker ps -a
    CONTAINER ID   IMAGE                                                          COMMAND     CREATED         STATUS                      PORTS     NAMES
    89c1ac38824d   registry.gitlab.com/gitlab-org/terraform-images/releases/1.1   "/bin/sh"   2 minutes ago   Exited (0) 11 seconds ago             test
   ```
6. Commit changes of the previously run container to a new image...
   ```
    $ docker commit <container ID> myregistry.example.com/my-namespace/terraform-nsxt:<terraform version>-<nsxt provider version>
 
    e.g.
     
    $ docker commit 60e74306f17b myregistry.example.com/my-namespace/terraform-nsxt:1.1.4-3.2.5 --format docker
    Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
    Getting image source signatures
    Copying blob 8d3ac3489996 skipped: already exists
    Copying blob 21476e0e54fd skipped: already exists
    Copying blob aaf800819d7a skipped: already exists
    Copying blob 9f0dcb82f796 skipped: already exists
    Copying blob 757b84e111a9 done
    Copying config c300a0052b done
    Writing manifest to image destination
    Storing signatures
    c300a0052b5cdb779ebbbb8ab54eaf041c2e7ff070f5c3ac964ea516b7c5cb1c
   ```
7. Push new image to Image Registry...
   ```
    $ docker login myregistry.example.com -u <username>
    Password:
    Login Succeeded
   
    $ docker push myregistry.example.com/my-namespace/terraform-nsxt:1.1.4-3.2.5
    Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
    Getting image source signatures
    Copying blob 757b84e111a9 skipped: already exists
    Copying blob aaf800819d7a skipped: already exists
    Copying blob 21476e0e54fd skipped: already exists
    Copying blob 8d3ac3489996 skipped: already exists
    Copying blob 9f0dcb82f796 [--------------------------------------] 0.0b / 0.0b
    Copying config a5c7d730ca done
    Writing manifest to image destination
    Storing signatures
   ```
