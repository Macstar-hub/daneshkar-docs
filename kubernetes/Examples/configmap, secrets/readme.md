- To make cm from cli: 
    kubectl create configmap example-cm --from-literal=sleep-interval=25 -n daneshkar

- To make cm from file: 
    kubectl create configmap my-config --from-file=config-file.conf -n daneshkar

- To make configmap from Directory: 
    kubectl create configmap my-config --from-file=/path/to/dir

- To make secret from cli:
    kubectl create secret generic fortune-https --from-file=https.key ➥ --from-file=https.cert --from-file=foo