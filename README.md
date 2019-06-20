
## EdgeX Helm chart 

Status: pre-alpha

A Helm chart to easily deploy the EdgeX IoT project on Kubernetes.

It contains charts for the following projects:

  * EdgeX
  * EdgeX UI
  * Black Box Testing
  * API Security Gateway
  
WARNING: This project is based on the [EdgeX Docker Compose scripts](https://github.com/edgexfoundry/developer-scripts). 
This project is not secure and is not suitable for production use. See Known Security Issues below.
Note: This is based on EdgeX Foundry Delhi release.

### Usage

Download the repo and change directory

```bash
# clone repo and change to dir
git clone
cd edgex-chart
```

Use the default set of images or optionally use local dev images or nightly build images.

```bash

# install with values for current release images  
helm install edgex

# install with values for development images (requires images as built with edgex-go Makefile)
helm install edgex -f edgex/values.dev.yaml

# install with values for nightly build images from nexus3.edgexfoundry
helm install edgex -f edgex/values.nightly.yaml

```

### Known Security Issues
 
This project is not secure and should not be used in production.

The following list contains known security issues (other issues could exist):  

  * None of the services have encrypted TLS connections except Vault, Kong and EdgeX Proxy. 
  * Vault, Kong and EdgeX Proxy use self-signed TLS certificates.
  * The UI does not use TLS.
  * Improper handling of vault root token (insecure on filesystem). See vault_worker_wrapper.sh in 
    edgex-vault/templates/cm-vault.yaml.
  * All services specify credentials in clear text config files which are loaded into config maps.
  * The EdgeX Proxy init quietly fails to retrieve an actual certificate from Vault and then noticeably fails to 
      register (400) it with Vault for Kong. This prevents Kong from having a customized cert so https calls must be made
      with cert checking turned off (other cert problems may exist anyway). Plain HTTP calls function as expected.

#### Security Issue Mitigation
 
  * Securing Ingress -  The [API Security Gateway](https://docs.edgexfoundry.org/Ch-APIGateway.html) is intended to be 
  single point of entry for all EdgeX REST traffic and prevent unauthorized access. The API Security Gateway should 
  use an SSL certificate signed by a trusted authority to promote secure practices with users (to avoid having users 
  accept self-signed certificates).
    
  * Mutual TLS -  MTLS requires each end of a service connection to prove its identity with a certificate. See 
  https://istio.io/docs/tasks/security/mutual-tls/ for an example implementation.
  
  * Certificate Rotation - Certificates should expire and be renewed at predetermined intervals. 
  https://github.com/helm/charts/tree/master/stable/cert-manager
  
  * Vault - Vault should be unsealed using the prescribed practices instead of passing the root token around. See
  https://www.vaultproject.io/docs/configuration/seal/index.html More information on the Vault Helm chart can be found 
  at https://github.com/helm/charts/tree/master/incubator/vault
  
  * EdgeX UI - Until the UI employs security mechanisms it should be fronted by a TLS enabled web server such as NGINX 
  or other TLS enabled endpoint. This can be achieved in numerous ways but the use of a sidecar container is popular in
  Kubernetes. An example can be found on the [Official Kubernetes blog](https://kubernetes.io/blog/2015/07/strong-simple-ssl-for-kubernetes/)  
    
  * The plaintext passwords found in each service's configuration files should be moved to Kubernetes Secrets or 
  other secure storage. The passwords can be found in the following locations:

```
  edgex-core-data/templates/cm-core-data.yaml:48:      Password = 'password'
  edgex-support-notifications/templates/cm-support-notifications.yaml:41:      Password = ''
  edgex-support-notifications/templates/cm-support-notifications.yaml:49:    Password = 'mypassword'
  edgex-core-metadata/templates/cm-core-metadata.yaml:41:      Password = 'password'
  edgex-support-logging/templates/cm-support-logging.yaml:34:      Password = 'password'
  edgex-vault/templates/cm-edgex-proxy.yaml:36:    password = "changeme"
  edgex-export-client/templates/cm-export-client.yaml:45:      Password = ''
  edgex-support-scheduler/templates/cm-support-scheduler.yaml:161:      Password = 'password'
 ```

### Other Issues       

  * Support-scheduler fails to bootstrap without a DB connection but the error is obscured. (See [#1138](https://github.com/edgexfoundry/edgex-go/issues/1138))
  
  * Mixture of releases: The chart uses available Go based components where available and requires some dev Docker image versions to
    be built and made accessible to the cluster. User can specify additional values.yaml to customize. See Usage above.   
    
  * System is wired together by k8s service names which contain the Helm release name. While enabling multiple EdgeX 
    installs on the same cluster, it prevents building the system with a few components at a time. From a Helm point view 
    everything is a single release. See TODOs.
    
  * Most (but not all) Black Box Testing tests pass. 
   
  * The Black Box Testing container uses awk and bash to transpose the docker-compose based tests to run in k8s. This is
    fragile and is a short term solution. 
    
  * The Black Box Testing image is custom and its Dockerfile lives in the `docker` directory. The tag should be 
    `edgexfoundry/blackbox-testing:latest`.  
  
  * Using Consul for service configuration is not working. After enabling config-seed chart, you may add the `--consul` or `-c` flag to each service's
    command line parameters in the `values.yaml`. 
    
    ```yaml
    # values.yaml
    ...
    edgex-core-command:
      enabled: true
      image: nexus3.edgexfoundry.org:10004/docker-core-command-go:0.7.1
      command: ["/core-command", "-c", "-p", "docker"]
      init_wait: false  

    ```
    
    The services will connect to Consul for their config but will have problems because Consul's configs do not contain 
    proper host names for the k8s cluster. As a workaround each service's config maps can be mounted in config-seed and 
    loaded into Consul. See the example for `core-command-volume` in the config-seed deployment.
  
  * Vault Worker and EdgeX Proxy have been implemented as init containers for Vault instead of as separate charts.
   

### TODO

- [ ] Decouple service names: Replace .Release.Name references with template helper to facilitate overriding release 
      name in values.yaml to allow piecemeal installations.  

- [ ] Extract hardcoded parameters (like ports) into values.yaml

- [ ] Add standard chart features and default values (resource limits, pvc, annotations) if desired.

- [ ] Add an ingress for UI and API Gateway to enable access from outside the 
      cluster.

- [ ] Reconfigure UI or API Gateway so that requests from UI through API Gateway succeed (currently there is a mismatch 
      between requested URLs and endpoints)

- [ ] Solve remaining issues with Black Box Testing and system configuration so that all tests pass.

- [ ] Fix rules-engine registration with export-client.

- [ ] Implement secure initialization and credentials handling (upstream changes
      likely required). 
      
- [ ] Review and align all service names registered in Consul. Some services register as RELEASE NAME-service and some
      just as the service name. Decide on a convention and apply it consistently.
      
- [ ] Misc. errors in services' init. Review each service startup and eliminate init problems.      
     
           
      
      

