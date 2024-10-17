# Nested Containers in OpenShift Dev Spaces - OCP 4.17 Tech Preview

This repo implements a minimal OpenShift Dev Spaces workspace that demonstrates new support for nested containers being introduced with OpenShift 4.17.

In order to use this workspace, you need to apply some configuration changes to your OpenShift Cluster.

__Note:__ There are two things that you must take into consideration before proceeding.

1. The cluster that you apply these changes to will not be upgradable.  This must be done on a disposable cluster.

1. Your cluster needs to be on OCP v4.17.1+ 

1. You need a block storage provisioner for Dev Spaces to provision PVCs for developer workspaces.

Now, that we have that out of the way here are the changes that you need to apply to your cluster:

1. Enable `crun` as the default container runtime on the cluster compute nodes.

   ```bash
   cat << EOF | oc apply -f -
   apiVersion: machineconfiguration.openshift.io/v1
   kind: ContainerRuntimeConfig
   metadata:
     name: enable-crun-master
   spec:
     machineConfigPoolSelector:
       matchLabels:
         pools.operator.machineconfiguration.openshift.io/worker: ""
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
   cat << EOF | ${OC} apply -f -
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

1. Create an OpenShift Dev Spaces cluster

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
