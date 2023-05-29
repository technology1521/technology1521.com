

```
export SKYWALKING_RELEASE_VERSION=4.3.0  # change the release version according to your need
export SKYWALKING_RELEASE_NAME=skywalking  # change the release name according to your scenario
export SKYWALKING_RELEASE_NAMESPACE=default  # change the namespace to where you want to install SkyWalking

helm install "skywalking" ${REPO}/skywalking -n "default" \
  --set oap.image.tag=9.2.0 \
  --set oap.storageType=elasticsearch \
  --set ui.image.tag=9.2.0
```



















[root@master chart]# helm install skywalking skywalking -n logging  -f ./skywalking/values-my-es.yaml
NAME: skywalking
LAST DEPLOYED: Sat May 27 05:49:58 2023
NAMESPACE: logging
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:

************************************************************************
*                                                                      *
*                 SkyWalking Helm Chart by SkyWalking Team             *
*                                                                      *
************************************************************************

Thank you for installing skywalking.

Your release is named skywalking.

Learn more, please visit https://skywalking.apache.org/

Get the UI URL by running these commands:
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward svc/skywalking-ui 8080:80 --namespace logging