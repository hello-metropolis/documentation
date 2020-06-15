# Metropolis
## Data Model: metropolis_deployment

### Placeholder Values

Metropolis Deployments expect and can use certain built in placeholder values, in addition to being able to specify custom placeholder values.

* `METROPOLIS_SANDBOX_ID` needs to be provided to pass a unique identifier for the environment.  This sandbox id is used across notification engines and can (and should) be used within components when referencing a unique identifier.
* `METROPOLIS_REPO` when specified this will be used to configure the GitHub notification engine and the Deployment API.
* `METROPOLIS_REF` is a REF that can be used to checkout the relevant commit using git.  This can be either a branch name or a `SHA` commit hash.  Although this will often be set with a default value, for a branch for instance, when overridden from it's default value it can have a different value (for instance, the specific SHA commit hash).


## Data Model: trigger_event

### Placeholder Values

* `ref` that is passed in a trigger_event will be passed as `METROPOLIS_REF` to the deployment.
* `repo` that is passed in a trigger_event will be passed as a `METROPOLIS_RPEO` to the deployment.
* **placeholder values**
  * `METROPOLIS_DEPLOYMENT_NAME` will be used to provide the `name` of the deployment if a new deployment is spawned from the action.  If a new deployment is spawned and this is omitted, a unique hash will be set as the name of the value.
  * _all other values_ will be passed through as placeholder_values for the deployment.

Each of the placeholder values that an event link
