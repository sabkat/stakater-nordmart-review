# Prerequisites:
# 1. oc login --token=<TOKEN> --server=<SERVER>
# 2. HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
# 3. docker login -u <USERNAME> -p $(oc whoami -t) $HOST
# 4. cd deploy; helm dependency update

# For more on Extensions, see: https://docs.tilt.dev/extensions.html
load('ext://restart_process', 'docker_build_with_restart')

settings = read_json('tilt_options.json', default={})

if settings.get("namespace"):
  namespace =  settings.get("namespace")

if settings.get("default_registry"):
  default_registry(settings.get("default_registry").format(namespace), host_from_cluster='image-registry.openshift-image-registry.svc:5000/{}'.format(namespace))

if settings.get("allow_k8s_contexts"):
  allow_k8s_contexts(settings.get("allow_k8s_contexts"))

# Watch source code and on change rebuild artifacts and place in target folder
local_resource(
  'compile',
  'mvn package && ' +
  'java -Djarmode=layertools -jar target/review-v0.0.0-SNAPSHOT.jar extract --destination target/jar-extracted && ' +
  'rsync --delete --inplace --checksum -r target/jar-extracted/ target/jar',
  deps=['src', 'pom.xml'])

# Keeps the docker image updated
docker_build_with_restart(
  'stakater/stakater-nordmart-review',
  '.',
  entrypoint=['java', 'org.springframework.boot.loader.JarLauncher'],
  dockerfile='./Dockerfile',
  live_update=[
    sync('./target/jar/dependencies', '/opt/app'),
    sync('./target/jar/spring-boot-loader', '/opt/app'),
    sync('./target/jar/snapshot-dependencies', '/opt/app'),
    sync('./target/jar/application', '/opt/app'),
  ])

yaml = helm('./deploy/', namespace=namespace, values=['./tilt/values-local.yaml'])

k8s_yaml(yaml)

k8s_resource('review-mongodb', port_forwards=['27017:27017'])
k8s_resource('review', port_forwards=['9000:8080'], resource_deps=['compile'])