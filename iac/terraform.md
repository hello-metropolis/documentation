# Metropolis
## Infrastructure-as-Code

### Terraform Plugin


#### Metropolis Provider

The Metropolis provider is used to deploy sandbox environments. The provider needs to be configured with the proper credentials before it can be used.

```terraform
provider "metropolis" {
  host        = "http://hellometropolis.com"

  public_key  = "03c69016"
  private_key = "ab86e391"
}
```

**Argument Reference**

* `host` is the URL for the metropolis endpoint to configure sandbox environments with.
* `public_key` is the public key for your Metropolis account.
* `private_key` is the secret key for your Metropolis account.  The terraform provider will use this secret to generate a JWT token, but your private key will never be shared across the Internet.



#### Resource: metropolis_workspace

A workspace is a collection of metropolis resources for the same type of environment.  For example, an instance may have a workspace for `Staging` and `Production`, but `Staging` may have many different deployment instances running inside it, using the same shared resources.

`metropolis_workspace` describes the group of assets.

**Example Usage**

```terraform
resource "metropolis_workspace" "primary" {
  name = "Metropolis Quickstart"
  note = "Workspace for the metropolis quickstart application"
}
```

**Argument Reference**

* `name` (Required) Unique name for the environment.  This name will be referenced throughout the Metropolis user interface.
* `note` (Optional) Note to describe the environment.

#### Resource: metropolis_asset

An asset is a key/value pair that can be substituted during the build process.  It can be helpful to store values that are derived from different Terraform resources as assets.

See documentation for `metropolis_component` to learn how to reference these values from within this resource.

**Example Usage**

```terraform
resource "metropolis_asset" "metropolis_asset_database_private_ip" {
  name  = "DATABASE_PRIVATE_IP"
  value = "123.45.67.8"

  workspace_id = metropolis_workspace.primary.id
}
```

**Argument Reference**

* `name` (Required) Unique name for the asset.
* `value` (Required) Value to substitute when the asset is referenced.
* `workspace_id` (Required) Id of the Metropolis Workspace to nest the asset under.


#### Resource: metropolis_component

Components let you split your infrastructure into independent, reusable pieces, and think about each piece in isolation.  They accept arbitrary inputs (called `placeholders`) and can also reference stored values (called `assets`).

Your infrastructure can be built up of many of these components and each should be fairly simple.

**Example Usage**

```terraform
resource "metropolis_component" "helm_releases" {
  name              = "helm-deployments"
  container_name    = "gcr.io/cloud-builders/gcloud"
  placeholders      = [ "SANDBOX_ID", "DOCKER_TAG" ]
  workspace_id      = metropolis_workspace.primary.id

  component_did_mount = [
    "curl -sS https://raw.githubusercontent.com/hello-metropolis/metropolis-utils/master/scripts/helm/install.sh | sh",
    ". /metropolis-utils/.metropolis-utils"
  ]

  on_create = [
    "gcloud container clusters get-credentials ${var.cluster_name} --zone=us-west1",
    "helm install frontend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID infrastructure/helm/frontend/ --set image.tag=$_METROPOLIS_PLACEHOLDER.DOCKER_TAG --set image.repository=${var.docker_repo_frontend}",
    "helm install backend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID infrastructure/helm/backend/ --set image.tag=$_METROPOLIS_PLACEHOLDER.DOCKER_TAG --set env.RAILS_ENV=production --set env.AFTER_CONTAINER_DID_MOUNT=\"sh lib/docker/mount.sh\" --set env.SANDBOX_ID=$_METROPOLIS_PLACEHOLDER.SANDBOX_ID --set image.repository=${var.docker_repo_backend}"
  ]

  on_update = [
    "gcloud container clusters get-credentials ${var.cluster_name} --zone=us-west1",
    "helm upgrade frontend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID infrastructure/helm/frontend/ --set image.tag=$_METROPOLIS_PLACEHOLDER.DOCKER_TAG --set image.repository=${var.docker_repo_frontend}",
    "helm upgrade backend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID infrastructure/helm/backend/ --set image.tag=$_METROPOLIS_PLACEHOLDER.DOCKER_TAG --set env.RAILS_ENV=production --set env.AFTER_CONTAINER_DID_MOUNT=\"sh lib/docker/mount.sh\" --set env.SANDBOX_ID=$_METROPOLIS_PLACEHOLDER.SANDBOX_ID --set image.repository=${var.docker_repo_backend}"
  ]

  on_destroy = [
    "gcloud container clusters get-credentials ${var.cluster_name} --zone=us-west1",
    "helm delete frontend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID",
    "helm delete backend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID"
  ]

  skip = []
}
```

**Argument Reference**


* `name` (Required) Unique name of the component.
* `container_name` (Required) Name of the docker container to run the component in.
* `arguments` (Optional) An array of arguments to pass the docker container.
* `placeholders` (Optional) An array of each of the placeholders that are used in the `on_create`, `on_update` and `on_destroy` actions.  If not passed, the component will not use placeholders.
* `workspace_id` (Required) Id of the Metropolis Workspace to nest under.
* `component_did_mount` (Optional) An array of commands that should be run before triggering the specific action.
* `on_create` (Optional) An array of commands that should be run when creating a new sandbox.
* `on_update` (Optional) An array of commands that should be run when updating an existing sandbox.
* `on_destroy` (Optional) An array of commands that should be run when destroying an existing sandbox.
* `skip` (Optional) An array of actions to skip.  When `create`, `update` or  `destroy` are seen the corresponding action will be skipped.

**Placeholder and Asset Substitution**.

`container_name`, `arguments`, `component_did_mount`, `on_create`, `on_updated` and `on_destroy` all allow substitution based on both placeholder values and asset value.

**Placeholders**

Each sandbox deployment will have some properties that vary instance to instance.

One example is that a workspace may have many different sandboxes, but `METROPOLIS_SANDBOX_ID` is a placeholder value you can use to reference the specific sandbox.  Substitution for `$_METROPOLIS_PLACEHOLDER.METROPOLIS_SANDBOX_ID` will be replace with the sandbox name.  

_Example:_ for a Sandbox with the id of `feature-123` the command `helm install frontend-$_METROPOLIS_PLACEHOLDER.SANDBOX_ID` will evaluate to `helm install frontend-feature-123` in the runtime engine.

Metropolis allows both built-in placeholders and custom placeholders.  Built-in placeholders are prefixed with `METROPOLIS_` and are documented [here](/data_model.md).

**Assets**

Sandboxes may wish to reference shared properties that exist within your environment.

One example is that a workspace may use a shared database server.  Rather than hardcoding these values and IP addresses in your component code, you can reference assets.

". /metropolis-utils/.metropolis-utils && PROXY_INSTANCE_NAME=\"$_METROPOLIS_ASSET.DATABASE_INSTANCE_NAME\" SANDBOX_ID=\"$_METROPOLIS_PLACEHOLDER.SANDBOX_ID\" sh backend/lib/docker/setup-cloudproxy-and-mount.sh"

_Example:_ a sandbox environment may want to connect to a database named `metropolis-quickstart:us-west1:metropolis-quickstart-66e0c17ad7ec7e421f4c8a13e62d7bc8` that is stored in the `DATABASE_INSTANCE_NAME` asset.

The command `PROXY_INSTANCE_NAME=$_METROPOLIS_ASSET.DATABASE_INSTANCE_NAME sh run_database_command.sh` will evaluate to `PROXY_INSTANCE_NAME=metropolis-quickstart:us-west1:metropolis-quickstart-66e0c17ad7ec7e421f4c8a13e62d7bc8 sh run_database_command.sh` in the runtime engine.

#### Resource: metropolis_composition

Compositions contain a collection of components that can be used together to build, upgrade, and remove sandbox environments.  A composition can be thought of like a factory that can create deployments, which are concrete sandbox instances.

It `depends_on` all assets that are used within the components.  It can contain `event_link` objects that can trigger creation of deployments based on events that are registered in the system.

**Example Usage**

```terraform
resource "metropolis_composition" "primary" {
  name              = "Sandbox"
  workspace_id      = metropolis_workspace.primary.id

  component {
    id = metropolis_component.clone_source.id
  }

  component {
    id = metropolis_component.docker_build_frontend.id
  }

  component {
    id = metropolis_component.helm_releases.id
  }

  event_link {
    repo                    = "hello-metropolis/quickstart"
    event_name              = "pull_request"
    branch                  = "*"
    trigger_action          = "build"
    run_notification_engine = true
    spawn_sync              = true
  }

  depends_on = [
    metropolis_asset.metropolis_asset_database_instance_name
  ]
}
```

**Argument Reference**

* `name` (Required) Unique name for the composition.
* `workspace_id` (Required) Id of the Metropolis Workspace to nest under.
* `component` (Required) A composition requires an array of components, each with an id that references the components that have been created in the system.  
* `event_link` (Optional) An event link to cause triggering of the composition to build deployment instances of the sandboxes.  Event links are optional, but if you provide one it needs to specify.
  * `repo` (Required) Identifier for a reference to a source code repository
  * `event_name` (Required) Indicates the event name you're setting a trigger for.  Includes `pull_request`, `push` or `delete_branch`.
  * `trigger_action` (Required) Indicates the action that should be triggered on the composition when the event happens.  Options are `build`, `upgrade` or `remove`.
  * `run_notification_engine` (Optional) When the action is triggered should the notification engine be triggered?
  * `spawn_sync` (Optional) When the event link is triggered should it spawn additional synchronization events?  If enabled, when a `build` is triggered, new event links will automatically be generated to update and remove the instance (`upgrade`s will happen on `push`es and `remove`s will happen on `branch_delete`d).

#### Resource: metropolis_deployment

A deployment is an instance of a sandbox that is created from a composition.  It requires each of a compositions placeholders are populated with values and can also trigger updates based on events that are triggered.

**Example Usage**

```terraform
resource "metropolis_deployment" "master" {
  name           = "master"
  composition_id = metropolis_composition.primary.id
  state          = "build"

  placeholder {
    name  = "DOCKER_TAG"
    value = "latest"
  }

  placeholder {
    name  = "SANDBOX_ID"
    value = "master"
  }

  placeholder {
    name  = "METROPOLIS_REF"
    value = "master"
  }

  placeholder {
    name  = "METROPOLIS_REPO"
    value = "hello-metropolis/quickstart"
  }

  event_link {
    repo           = var.github_repo
    event_name     = "push"
    branch         = "master"
    trigger_action = "upgrade"
  }

}
```

**Argument Reference**

* `name` (Required) Unique name for the deployment.
* `composition_id` (Required) Id for the composition to build a sandbox with.
* `state` (Optional) If set to `build` on creation the `build` event will be triggered.
* `placeholder` (Required) Each of the compositions placeholders need to have specific values set.  Each instance of a placeholder requires
  * `name` (Required) Name of the placeholder attribute.
  * `value` (Required) Value to use when evaluating composition in the runtime engine.
* `event_link` (Optional) An event link to cause triggering of the composition to build deployment instances of the sandboxes.  Event links are optional, but if you provide one it needs to specify.
  * `repo` (Required) Identifier for a reference to a source code repository
  * `event_name` (Required) Indicates the event name you're setting a trigger for.  Includes `pull_request`, `push` or `delete_branch`.
  * `trigger_action` (Required) Indicates the action that should be triggered on the composition when the event happens.  Options are `build`, `upgrade` or `remove`.
  * `run_notification_engine` (Optional) When the action is triggered should the notification engine be triggered?
  * `spawn_sync` (Optional) When the event link is triggered should it spawn additional synchronization events?  If enabled, when a `build` is triggered, new event links will automatically be generated to update and remove the instance (`upgrade`s will happen on `push`es and `remove`s will happen on `branch_delete`d).
