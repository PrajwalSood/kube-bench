kube-bench is a Go application that checks whether Kubernetes is deployed securely by running the checks documented in the CIS Kubernetes Benchmark.
Note that it is impossible to inspect the master nodes of managed clusters, e.g. GKE, EKS and AKS, using kube-bench as one does not have access to such nodes, although it is still possible to use kube-bench to check worker node configuration in these environments.
Tests are configured with YAML files, making this tool easy to update as test specifications evolve.


CIS Kubernetes Benchmark support
kube-bench supports the tests for Kubernetes as defined in the CIS Benchmarks 1.3.0 to 1.4.0 respectively.
   CIS Kubernetes Benchmark kube-bench config Kubernetes versions 
    1.3.0	 1.11	 1.11-1.12	 
  1.4.1	 1.13	 1.13-	 
  By default, kube-bench will determine the test set to run based on the Kubernetes version running on the machine.
There is also preliminary support for Red Hat's Openshift Hardening Guide for 3.10 and 3.11. Please note that kube-bench does not automatically detect Openshift - see below.

Installation
You can choose to
run kube-bench from inside a container (sharing PID namespace with the host)
run a container that installs kube-bench on the host, and then run kube-bench directly on the host
install the latest binaries from the Releases page,
compile it from the source.

Running kube-bench
If you run kube-bench directly from the command line you may need to be root / sudo to have access to all the config files.
kube-bench automatically selects which controls to use based on the detected node type and the version of kubernetes a cluster is running. This behavior can be overridden by specifying the master or node subcommand and the --version flag on the command line.
The kubernetes version can also be set with the KUBE_BENCH_VERSION environment variable. The value of --version takes precedence over the value of KUBE_BENCH_VERSION.
For example: run kube-bench against a master with version auto-detection:
kube-bench master
or run kube-bench against a node with the node controls for kubernetes version 1.13:
kube-bench node --version 1.13
controls for the various versions of kubernetes can be found in directories with the same name as the kubernetes versions under cfg/, for example cfg/1.13. controls are also organized by distribution under the cfg directory, for example, cfg/ocp-3.10.

Running inside a container
You can avoid installing kube-bench on the host by running it inside a container using the host PID namespace and mounting the /etc and /var directories where the configuration and other files are located on the host, so that kube-bench can check their existence and permissions.
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest [master|node] --version 1.13
Note: the tests require either the kubelet or kubectl binary in the path in order to auto-detect the Kubernetes version. You can pass -v $(which kubectl):/usr/bin/kubectl to resolve this. You will also need to pass in kubeconfig credentials. For example:
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -v $(which kubectl):/usr/bin/kubectl -v ~/.kube:/.kube -e KUBECONFIG=/.kube/config -t aquasec/kube-bench:latest [master|node] 
You can use your own configs by mounting them over the default ones in /opt/kube-bench/cfg/
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t -v path/to/my-config.yaml:/opt/kube-bench/cfg/config.yam -v $(which kubectl):/usr/bin/kubectl -v ~/.kube:/.kube -e KUBECONFIG=/.kube/config aquasec/kube-bench:latest [master|node]

Running in a kubernetes cluster
You can run kube-bench inside a pod, but it will need access to the host's PID namespace to check the running processes, as well as access to some directories on the host where config files and other files are stored.
Master nodes are automatically detected by kube-bench and will run master checks when possible. The detection is done by verifying that mandatory components for master, as defined in the config files, are running (see Configuration).
The supplied job.yaml file can be applied to run the tests as a job. For example:
$ kubectl apply -f job.yaml
job.batch/kube-bench created

$ kubectl get pods
NAME                      READY   STATUS              RESTARTS   AGE
kube-bench-j76s9   0/1     ContainerCreating   0          3s

# Wait for a few seconds for the job to complete
$ kubectl get pods
NAME                      READY   STATUS      RESTARTS   AGE
kube-bench-j76s9   0/1     Completed   0          11s

# The results are held in the pod's logs
kubectl logs kube-bench-j76s9
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
...
You can still force to run specific master or node checks using respectively job-master.yaml and job-node.yaml.
To run the tests on the master node, the pod needs to be scheduled on that node. This involves setting a nodeSelector and tolerations in the pod spec.
The default labels applied to master nodes have changed since Kubernetes 1.11, so if you are using an older version you may need to modify the nodeSelector and tolerations to run the job on the master node.

Running in an EKS cluster
There is a job-eks.yaml file for running the kube-bench node checks on an EKS cluster. Note that you must update the image reference in job-eks.yaml. Typically you will push the container image for kube-bench to ECR and refer to it there in the YAML file.
There are two significant differences on EKS:
It uses config files in JSON format
It's not possible to schedule jobs onto the master node, so master checks can't be performed

Installing from a container
This command copies the kube-bench binary and configuration files to your host from the Docker container: ** binaries compiled for linux-x86-64 only (so they won't run on OSX or Windows) **
docker run --rm -v `pwd`:/host aquasec/kube-bench:latest install
You can then run ./kube-bench [master|node].

Installing from sources
If Go is installed on the target machines, you can simply clone this repository and run as follows (assuming your $GOPATH is set):
go get github.com/aquasecurity/kube-bench
go get github.com/golang/dep/cmd/dep
cd $GOPATH/src/github.com/aquasecurity/kube-bench
$GOPATH/bin/dep ensure -vendor-only
go build -o kube-bench .

# See all supported options
./kube-bench --help

# Run all checks
./kube-bench

Running on OpenShift
kube-bench includes a set of test files for Red Hat's OpenShift hardening guide for OCP 3.10 and 3.11. To run this you will need to specify --version ocp-3.10 when you run the kube-bench command (either directly or through YAML). This config version is valid for OCP 3.10 and 3.11.

Output
There are three output states
[PASS] and [FAIL] indicate that a test was run successfully, and it either passed or failed
[WARN] means this test needs further attention, for example, it is a test that needs to be run manually
[INFO] is informational output that needs no further action.
Note:
If the test is Manual, this always generates WARN (because the user has to run it manually)
If the test is Scored, and kube-bench was unable to run the test, this generates FAIL (because the test has not been passed, and as a Scored test, if it doesn't pass then it must be considered a failure).
If the test is Not Scored, and kube-bench was unable to run the test, this generates WARN.
If the test is Scored, type is empty, and there are no test_items present, it generates a WARN.

Configuration
Kubernetes configuration and binary file locations and names can vary from installation to installation, so these are configurable in the cfg/config.yaml file.
Any settings in the version-specific config file cfg/<version>/config.yaml take precedence over settings in the main cfg/config.yaml file.
You can read more about kube-bench configuration in our documentation.

Test config YAML representation
The tests (or "controls") are represented as YAML documents (installed by default into ./cfg). There are different versions of these test YAML files reflecting different versions of the CIS Kubernetes Benchmark. You will find more information about the test file YAML definitions in our documentation.

Omitting checks
If you decide that a recommendation is not appropriate for your environment, you can choose to omit it by editing the test YAML file to give it the check type skip as in this example:
  checks:
  - id: 2.1.1
    text: "Ensure that the --allow-privileged argument is set to false (Scored)"
    type: "skip"
    scored: true
No tests will be run for this check and the output will be marked [INFO].

Roadmap
Going forward we plan to release updates to kube-bench to add support for new releases of the Benchmark, which in turn we can anticipate being made for each new Kubernetes release.
We welcome PRs and issue reports.

Testing locally with kind
Our makefile contains targets to test your current version of kube-bench inside a Kind cluster. This can be very handy if you don't want to run a real kubernetes cluster for development purpose.
First you'll need to create the cluster using make kind-test-cluster this will create a new cluster if it cannot be found on your machine. By default the cluster is named kube-bench but you can change the name by using the environment variable KIND_PROFILE.
If kind cannot be found on your system the target will try to install it using go get
Next, you'll have to build the kube-bench docker image using make build-docker, then we will be able to push the docker image to the cluster using make kind-push.
Finally, we can use the make kind-run target to run the current version of kube-bench in the cluster and follow the logs of pods created. (Ctrl+C to exit)
Every time you want to test a change, you'll need to rebuild the docker image and push it to cluster before running it again. ( make build-docker kind-push kind-run )