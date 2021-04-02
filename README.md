# Demo: Using `cosign` natively in Shipwright

### Install Shipwright and dependencies

1. Install dependencies:
  - latest Tekton release
  - forked Shipwright with signing support, built from [this fork](https://github.com/imjasonh/build-1/tree/sign).
  - the `kaniko` ClusterBuildStrategy

```
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f https://raw.githubusercontent.com/ImJasonH/shipwright-cosign-example/main/shipwright-fork.yaml
kubectl apply -f https://raw.githubusercontent.com/shipwright-io/build/master/samples/buildstrategy/kaniko/buildstrategy_kaniko_cr.yaml
```

2. Create the `build-bot` ServiceAccount and passphrase Secret, and registry secret to authorize image pushes:

```
kubectl apply -f https://raw.githubusercontent.com/ImJasonH/shipwright-cosign-example/main/sa.yaml
kubectl apply -f https://raw.githubusercontent.com/ImJasonH/shipwright-cosign-example/main/secret.yaml
kubectl create secret generic dockerhub-dockerconfig \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

3. Define the Build

```
kubectl apply -f https://raw.githubusercontent.com/ImJasonH/shipwright-cosign-example/main/build.yaml
```

...then edit it to push to your Dockerhub user:

```
kubectl edit build kaniko-build
```

...and modify the `image`:

```
  output:
    image: index.docker.io/imjasonh/signed  # <-- edit here
    credentials:
      name: dockerhub-dockerconfig
```

---

Up until this point, everything is the standard process for installing, setting up, and using Shipwright.
This Build, however, includes a new `.spec.sign` field, which describes how to sign the built image:

```
sign:
  keyPath: cosign.key
  passphrase:
    name: passphrase
```

This tells Shipwright to use the `cosign.key` found in the repo to sign the built image.
That key is encrypted, and the Secret named `passphrase` includes the passphrase to decrypt it.

4. Create a BuildRun to execute the Build

```
kubectl create -f https://raw.githubusercontent.com/ImJasonH/shipwright-cosign-example/main/buildrun.yaml
```

5. Finally, verify the image is signed:

```
$ cosign verify -key cosign.pub imjasonh/signed # <-- your image here
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
  - WARNING - THE CERTIFICATE EXPIRY WAS NOT CHECKED. set COSIGN_EXPERIMENTAL=1 to check!
{"Critical":{"Identity":{"docker-reference":""},"Image":{"Docker-manifest-digest":"sha256:c2e2943023baf0ccbe96db5bf21dc5e181bc597e7d5afb87750611ae10615a66"},"Type":"cosign container signature"},"Optional":null}
```

:tada:
