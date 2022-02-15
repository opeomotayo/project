
helm repo add oteemocharts https://oteemo.github.io/charts
helm search repo oteemocharts
helm upgrade --install nexus -n nexus oteemocharts/sonatype-nexus -f nexus/nexus-values.yaml


nexusDockerHost: 192.168.56.50
nexusHttpHost: 192.168.56.50