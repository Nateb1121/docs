---
title: Gitlab
taxonomy:
    category: docs
---

### Scan for Vulnerabilities during Gitlab Build Pipeline

NeuVector can be configured to scan for vulnerabilities triggered in the Gitlab build pipeline. Currently, the scan requires use of the NeuVector REST API and configuring the provided script below to access the controller.

In addition, make sure there is a NeuVector scanner container deployed and configured to connect to the Allinone or Controller. In 4.0 and later, the neuvector/scanner container must be deployed separate from the allinone or controller.

#### Scan During Gitlab Build Using REST API

Use the following script, configured for your NeuVector login credentials and replacing the license with your license, to trigger the vulnerability scans.

```
########################
# Scanning Job
########################

NeuVector_Scan:
  image: docker:latest
  stage: test
  #the runner tag name is nv-scan 
  tags:
    - nv-scan
  services:
    - docker:dind
  before_script:
    - apk add curl
    - apk add jq
  variables:
    DOCKER_DAEMON_PORT: 2376
    DOCKER_HOST: "tcp://$CI_SERVER_HOST:$DOCKER_DAEMON_PORT"
    #the name of the image to be scanned
    NV_TO_BE_SCANNED_IMAGE_NAME: "nv_demo"
    #the tag of the image to be scanned
    NV_TO_BE_SCANNED_IMAGE_TAG: "latest"
    #for local, set NV_REGISTRY=""
    #for remote, set NV_REGISTRY="[registry URL]"
    NV_REGISTRY_NAME: ""
    #the credential to login to the docker registry
    NV_REGISTRY_USER: ""
    NV_REGISTRY_PASSWORD: ""
    #NeuVector image location
    NV_IMAGE: "10.1.127.3:5000/neuvector/controller"
    NV_PORT: 10443
    NV_LOGIN_USER: "admin"
    NV_LOGIN_PASSWORD: "admin"
    #NeuVector License
    NV_LICENSE: "T0ANu7J05vrFNJD031/Ua1soxcQPfpFcVq/d46B8xQGI9rJRLT1D3cZnsT105fcCQQwQE2sx7Iv/ylWdGZ/zmRPRHCPAQ8oRYdMr2Hs5iXcIq1jC6t5AKv/yKJbouEP7FhamTV2l8L10nhK3IyDUb2rcoVOIsugGiqUu7ebHVlnUHAqGcPMNt+sN2uqNaEx42OMotjJRJF929xnFnTtINyC0nNbH8mudIVnZ+xAR5bNbnp7c/V7Ptvh1N1Pigtvye9acuLerGGz3vtExqiUQ3ek2kH27mCnZvue9Q3xCr/La2l1G4MEBZ9BIhFtvia7K24FFGDp+mX9u9fBQaNiWcWqSG+DP30AJZaUsxcyOSelPVrRIrs9OMKNiyCp5cGTfJPJ5xA/jTpx7/w0NyFCuH2xmqWEpYM9aOWjMS1cVipvSvLESPBC1ylDsq1Nw8W4htpqxp8swDYkxnCtHXNqGtx+ochGzBMkwx3z/OEi9SpknEQohJlOe6iU+GEP+G7Yl9+i1LjeKC+8et/e7qV42BqzNXZCH5CuXUzR6cVs06wj02BoVjW9bGwOtMXHAZEdFj6p41Ta38auXduQFM4NchClYEVpaFEP89q/hssQWQUCGNY0/T3/D2YyZPqLoE4Bv5M46uKyJWDQM1aZQ10N/IIqULAhGuv/sE68K3SOAZa8McqZAPKwfVfjhx1aonH7H2bZTdylMNqVWMoAHEmfVIIbSkwhPkmqMi34jr+D5QosSv3q7Ty9X3gRuaK4YNNkGsdd7e6WvDYlnu6b9ee8vMA4uFfSWg4H0"
    
    NV_LOGIN_JSON: '{"password":{"username":"$NV_LOGIN_USER","password":"$NV_LOGIN_PASSWORD"}}'
    NV_LICENSE_JSON: '{"license_key":"$NV_LICENSE"}'
    NV_SCANNING_JSON: '{"request":{"registry":"$NV_REGISTRY","username":"$NV_REGISTRY_NAME","password":"$NV_REGISTRY_PASSWORD","repository":"$NV_TO_BE_SCANNED_IMAGE_NAME","tag":"$NV_TO_BE_SCANNED_IMAGE_TAG"}}'
    NV_API_AUTH_URL: "https://$CI_SERVER_HOST:$NV_PORT/v1/auth"
    NV_API_LICENSE_URL: "https://$CI_SERVER_HOST:$NV_PORT/v1/system/license/update"
    NV_API_SCANNING_URL: "https://$CI_SERVER_HOST:$NV_PORT/v1/scan/repository"
  script: 
    - echo "Start neuvector scanner"
    - docker run -itd --privileged --name neuvector.controller -e CLUSTER_JOIN_ADDR=$CI_SERVER_HOST -p 18301:18301 -p 18301:18301/udp -p 18300:18300 -p 18400:18400  -p $NV_PORT:$NV_PORT -v /var/neuvector:/var/neuvector -v /var/run/docker.sock:/var/run/docker.sock -v /proc:/host/proc:ro -v /sys/fs/cgroup/:/host/cgroup/:ro $NV_IMAGE
    - |
      _COUNTER_="0"
      while [ -z "$TOKEN" -a "$_COUNTER_" != "12" ]; do
        _COUNTER_=$((( _COUNTER_ + 1 )))
        sleep 5
        TOKEN=`(curl -s -f $NV_API_AUTH_URL -k -H "Content-Type:application/json" -d $NV_LOGIN_JSON || echo null) | jq -r '.token.token'`
        if [ "$TOKEN" = "null" ]; then
          TOKEN=""
        fi
      done
    - echo "Apply license"
    - curl $NV_API_LICENSE_URL -s -k -H "Content-Type:application/json" -H "X-Auth-Token:$TOKEN" -d $NV_LICENSE_JSON
    - echo "Scanning ..."
    - sleep 20
    - curl $NV_API_SCANNING_URL -s -k -H "Content-Type:application/json" -H "X-Auth-Token:$TOKEN" -d $NV_SCANNING_JSON | jq .
    - echo "Logout"
    - curl $NV_API_AUTH_URL -k -X 'DELETE' -H "Content-Type:application/json" -H "X-Auth-Token:$TOKEN"

  after_script:
    - docker stop neuvector.controller
    - docker rm neuvector.controller
```