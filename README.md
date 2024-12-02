# Nested Containers in OpenShift Dev Spaces - OCP 4.17 Tech Preview

This repo implements a minimal OpenShift Dev Spaces workspace that demonstrates new support for nested containers being introduced with OpenShift 4.17.

See this Red Hat Developers Blog - [Enable nested containers in OpenShift Dev Spaces with user namespaces](https://developers.redhat.com/articles/2024/12/02/enable-nested-containers-openshift-dev-spaces-user-namespaces)

In order to use this workspace, you need to apply some configuration changes to your OpenShift Cluster.

__Note:__ There are three things that you must take into consideration before proceeding.

1. The cluster that you apply these changes to will not be upgradable.  This must be done on a disposable cluster.

1. Your cluster needs to be on OCP v4.17.1+ 

1. You need a block storage provisioner for Dev Spaces to provision PVCs for developer workspaces.

Now, that we have that out of the way here are the changes that you need to apply to your cluster:

1. Enable `crun` as the default container runtime on the cluster compute nodes.

   For a cluster with both control-plane ("master"), and compute ("worker") nodes, apply the following MachineConfig to enable `crun` as the default container runtime on the compute nodes.

   ```bash
   cat << EOF | oc apply -f -
   apiVersion: machineconfiguration.openshift.io/v1
   kind: ContainerRuntimeConfig
   metadata:
     name: enable-crun-worker
   spec:
     machineConfigPoolSelector:
       matchLabels:
         pools.operator.machineconfiguration.openshift.io/worker: ""
     containerRuntimeConfig:
       defaultRuntime: crun
   EOF
   ```

   For a Single Node cluster, or if you want to enable usernamespaces support on the control-plane too, then apply the following MachineConfig to enable `crun` as the default container runtime on the control-plane nodes.

   ```bash
   cat << EOF | oc apply -f -
   apiVersion: machineconfiguration.openshift.io/v1
   kind: ContainerRuntimeConfig
   metadata:
     name: enable-crun-master
   spec:
     machineConfigPoolSelector:
       matchLabels:
         pools.operator.machineconfiguration.openshift.io/master: ""
     containerRuntimeConfig:
       defaultRuntime: crun
   EOF
   ```

   __Note:__ Your compute nodes will perform a rolling reboot to apply this change.

1. Enable the feature gates for `UserNamespacesSupport` and `ProcMountType`

   ```bash
   oc patch FeatureGate cluster --type merge --patch '{"spec":{"featureSet":"CustomNoUpgrade","customNoUpgrade":{"enabled":["ProcMountType","UserNamespacesSupport"]}}}'
   ```

   __Note:__ Your cluster will perform a rolling reboot to apply this change.

1. Create a SecurityContextConstraint for OpenShift Dev Spaces to support nested containers

   ```bash
   cat << EOF | oc apply -f -
   apiVersion: security.openshift.io/v1
   kind: SecurityContextConstraints
   metadata:
     name: nested-podman-scc
   priority: null
   allowPrivilegeEscalation: true
   allowedCapabilities:
   - SETUID
   - SETGID
   fsGroup:
     type: MustRunAs
     ranges:
     - min: 1000
       max: 65534
   runAsUser:
     type: MustRunAs
     uid: 1000
   seLinuxContext:
     type: MustRunAs
     seLinuxOptions:
       type: container_engine_t
   supplementalGroups:
     type: MustRunAs
     ranges:
     - min: 1000
       max: 65534
   EOF
   ```

1. Install OpenShift Dev Spaces:

   ```bash
   cat << EOF | oc apply -f -
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: devspaces
     namespace: openshift-operators
   spec:
     channel: stable 
     installPlanApproval: Automatic
     name: devspaces 
     source: redhat-operators 
     sourceNamespace: openshift-marketplace 
   EOF
   ```

1. Create an OpenShift Dev Spaces cluster that uses the new SCC

   ```bash
   cat << EOF | oc apply -f -
   apiVersion: v1                      
   kind: Namespace                 
   metadata:
     name: devspaces
   ---           
   apiVersion: org.eclipse.che/v2 
   kind: CheCluster   
   metadata:              
     name: devspaces  
     namespace: devspaces
   spec:                         
     components:                  
       cheServer:      
         debug: false
         logLevel: INFO
       metrics:                
         enable: true
       pluginRegistry:
         openVSXURL: https://open-vsx.org
     containerRegistry: {}      
     devEnvironments:       
       startTimeoutSeconds: 600
       secondsOfRunBeforeIdling: -1
       maxNumberOfWorkspacesPerUser: -1
       maxNumberOfRunningWorkspacesPerUser: 5
       containerBuildConfiguration:
         openShiftSecurityContextConstraint: nested-podman-scc
       disableContainerBuildCapabilities: false
       defaultComponents:
       - name: dev-tools
         container:
           image: quay.io/cgruver0/che/dev-tools:latest
           memoryLimit: 6Gi
           mountSources: true
       defaultEditor: che-incubator/che-code/latest
       defaultNamespace:
         autoProvision: true
         template: <username>-devspaces
       secondsOfInactivityBeforeIdling: 1800
       storage:
         pvcStrategy: per-workspace
     gitServices: {}
     networking: {}   
   EOF
   ```

1. Log into Dev Spaces as a non-admin user and create a new workspace from this git repo:

   `https://github.com/cgruver/ocp-4-17-nested-container-tech-preview.git`

   __Note:__ The container image for this workspace is built from the files in the `./workspace-image` folder of this project.

1. Demonstrate running a container with podman:

   Open a new terminal in the workspace and run -

   ```bash
   podman run -d --rm --name webserver -p 8080:80 quay.io/libpod/banner
   curl http://localhost:8080
   ```

   You should observer the following output:

   ```bash
   Trying to pull quay.io/libpod/banner:latest...
   Getting image source signatures
   Copying blob 64dc81575282 done   | 
   Copying blob 2408cc74d12b done   | 
   Copying blob 92ec11331c38 done   | 
   Copying blob ef4966331ce5 done   | 
   Copying config 5ba9aec95f done   | 
   Writing manifest to image destination
   22fcc41e7fca27f37841aafab535db2dc836d94aa513594f440b5a4824c4bef7
      ___          __              
     / _ \___  ___/ /_ _  ___ ____ 
    / ___/ _ \/ _  /  ' \/ _ `/ _ \
   /_/   \___/\_,_/_/_/_/\_,_/_//_/
   ```

1. Stop the running container:

   ```bash
   podman kill webserver
   ```

Now, go have fun with podman in OpenShift Dev Spaces...
