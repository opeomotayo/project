kubectl apply -f jenkins-volume.yaml
kubectl -n jenkins apply -f jenkins-sa.yaml
helm repo add jenkinsci https://charts.jenkins.io
helm upgrade --install jenkins -n jenkins -f jenkins-values.yaml jenkinsci/jenkins

printf $(kubectl get secret --namespace jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

avxJwpTQB3U27VZ0cBlL2r

https://github.com/jenkinsci/configuration-as-code-plugin
https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos



https://hub.kubeapps.com/charts/jenkinsci/jenkins/3.10.3
https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/README.md
https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos
https://github.com/jenkinsci/helm-charts




kubernetes_plugin: |
  jenkins:
    location:
      url: "http://jenkins-master:8080"
    clouds:
      - kubernetes:
           name: "kubernetes"
           serverUrl: "https://kubernetes.default"
           skipTlsVerify: true
           namespace: "jenkins"
           jenkinsUrl: "http://jenkins-master:8080"
           jenkinsTunnel: "jenkins-master-agent:50000"
           connectTimeout: 0
           readTimeout: 0
           containerCap: 10
           maxRequestsPerHost: 32
           waitForPodSec: 600
           templates:
             - name: "default"
                namespace: "default"
                label: "jenkins-pod"
                nodeUsageMode: "EXCLUSIVE"
                containers:
                  - name: "jnlp"
                     image: "jenkins/jnlp-slave:3.35-3-alpine"
                     alwaysPullImage: false
                     workingDir: "/home/jenkins/agent"
                     ttyEnabled: false
                     args: "${computer.jnlpmac}${computer.name}"
                     resourceRequestCpu: "500m"
                     resourceLimitCpu: "1000m"
                     resourceRequestMemory: "1Gi"
                     resourceLimitMemory: "2Gi"
                imagePullSecrets: "artifactory-registry"