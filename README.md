# ACM Configuration with Maintanance Groups

## High level Folder Structure
```
├── app-of-app
├── base
├── cluster-configurations
├── components
├── init
├── maintanance-groups
└── user-roles
```

### Folder *components*
Folder to manage Individual application manifests. These are basic manifests for individual applications that will be overlayed with configurations for different clusters and maintanance groups. 

#### Sample Contents
```
components/
├── AC-Access-Control
│   ├── kustomization.yaml
│   ├── placementBinding.yaml
│   └── policy.yaml
├── hello-world
│   ├── hello-world-deployment.yaml
│   ├── hello-world-service.yaml
│   └── kustomization.yaml
├── mysql
│   ├── deployment-frontend.yaml
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── service-frontend.yaml
│   └── service.yaml
```
In the above folder structure, mysql and hello-world are two different applications and AC-Access-Control contain a governance policy

### Folder *cluster-configurations*
Kustomization for different clusters grouped by region and environment 

```
cluster-configurations/
├── east
│   ├── dev
│   │   ├── AC-Access-Control
│   │   │   └── kustomization.yaml
│   │   ├── hello-world
│   │   │   ├── kustomization.yaml
│   │   │   └── route.yaml
│   │   └── mysql
│   │       ├── dbclaim-pvc.yaml
│   │       ├── kustomization.yaml
│   │       └── route.yaml
│   ├── lab
│   ├── prod
│   └── stage
└── west
    ├── dev
    │   ├── AC-Access-Control
    │   │   └── kustomization.yaml
    │   ├── hello-world
    │   │   ├── kustomization.yaml
    │   │   └── route.yaml
    │   ├── mysql
    │   │   ├── dbclaim-pvc.yaml
    │   │   ├── kustomization.yaml
    │   │   └── route.yaml
    ├── lab
    ├── prod
```
This directory serves as the repository for Kustomizations organized by region and environment for various clusters.

Within this directory, overlays should be created for specific applications, extending the resources from the components folder.

For instance:

The kustomization located at cluster-configuration/east/dev/hello-world includes all resources from components/hello-world and incorporates a new route.yaml for the east region development cluster.

The kustomization at cluster-configuration/east/dev/mysql encompasses all resources from components/mysql and integrates custom resources for the addition of a PVC.

### Folder *maintance group*
This directory serves as the repository for ACM application subscriptions associated with various maintenance groups. These subscriptions are configured to reference the cluster-configuration folder.

For example:

Within the 'dev-primary' folder, you will find subscriptions, applications, and placement groups related to a specific set of applications within this maintenance group.

When you run 'kustomize build' within this folder, it will generate the necessary 'application.yaml' and 'subscription.yaml' files for each application in this group. These subscriptions are set to target folders in 'cluster-configurations/east/dev/< appname >'.

Furthermore, the subscriptions not only reference the folder name but also a specific GitTag, which will be explained in more detail later below.

### Folder *base/gitTag*
Kustomization.yaml located in this folder contains a map of litrals which lists the Git Tags pointing to different maintanance groups. These litrals being referred by kustomize while generating the subscriptions for different maintanance-groups

`dev-primary/kustomize.yaml` has following lines that assigns the value of `dev-primary-tag` to the subscription for each application in the dev-primary

```yaml
- source:
    kind: ConfigMap
    name: git-config-map
    fieldPath: data.dev-primary-tag
  targets:
  - select:
      kind: Subscription
    fieldPaths:
      - metadata.annotations.[apps.open-cluster-management.io/git-tag]
```

### Folder *app-of-app*
This directory includes ACM manifests for configuring Git subscriptions within the 'maintenance-group' folder. It essentially acts as a subscription for subscriptions, as it subscribes to the 'maintenance-group' folder, which, in turn, subscribes to individual cluster configurations.

### Folder *base/common*
This folder contains shared manifests that are referenced across multiple 'kustomize.yaml' files in the repository to prevent duplication

### Folder *init*
Initial setup folder for the repository. This creates the channel and a initiates app-of-app subscription

## How to make a Change
1. Make Code changes and commit to git
2. Tag and create a release
3. Update base/gitTag/kustomization.yaml with the appropriate release number for the maintenance group
4. Commit the changes to git
5. ACM will synchronize the changes for the maintenance group

### Example

Let us say we need to add a new route to `mysql` application in `west/dev` environment. Here are the sequence of steps

1. Add the new route.yaml to `cluster-config/west/dev/mysql`folder
2. Update kustomize.yaml in the `cluster-config/west/dev/mysql` folder to reference the new resource (`route.yaml`)
3. Commit and push the changes to the repository
4. In Git create a new release for the branch (Example: `V2.0`).
5. Edit `base/gitTag/kustomize.yaml` to change the value of `dev-secondary-tag` to `V2.0`
6. Commit and Push this change to git
7. ACM will sync the `dev-secondary` application with the git tag `V2.0`

## Adding a new application
1. Add the new application to the `components` folder with the necessary manifests
2. Create kustomization overlays in cluster-configurations folder for each environment that the application needs to be deployed. 
    1. At the bare minimum, there should be a kustomize.yaml that references the resources from the `components/<new-application>` folder
3. Make an entry in each maintenance group with a customization.yaml cloning an existing application
4. Make Code changes and commit to git
5. Tag and create a release
6. Update base/gitTag/kustomization.yaml with the appropriate release number for the maintenance group
7. Commit the changes to git
8. ACM will synchronize the changes for the maintenance group


