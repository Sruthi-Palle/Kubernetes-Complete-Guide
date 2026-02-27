# Demo Commands and Arguments

> This article reviews a examples on Kubernetes container commands and arguments, including pod inspection, YAML modifications, and explore how Dockerfiles and Kubernetes manifests interact to determine container startup behavior.

---

## Reviewing an Existing Pod

Begin by checking the number of pods running on the system. In this example, one pod is active. Next, inspect the pod named "ubuntu-sleeper" to determine the command used to run it:

```bash theme={null}
kubectl describe pod ubuntu-sleeper
```

The output reveals:

```bash theme={null}
Command:
  $ sleep
  4800
```

This indicates that the container runs with the command `sleep 4800`. Running the describe command again confirms the same details:

```bash theme={null}
kubectl describe pod ubuntu-sleeper
```

---

## Creating and Modifying a Pod to Sleep for 5000 Seconds

The next task is to create a pod using the Ubuntu image that executes a container with a 5000-second sleep duration. Start by modifying an existing pod definition file named "ubuntu-sleeper-2". The basic YAML structure is:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-2
spec:
  containers:
    - name: ubuntu
      image: ubuntu
```

To define a startup command for sleeping 5000 seconds, you can use one of two methods:

1. Define the entire command (including its argument) as a single array:

   ```yaml theme={null}
   apiVersion: v1
   kind: Pod
   metadata:
     name: ubuntu-sleeper-2
   spec:
     containers:
       - name: ubuntu
         image: ubuntu
         command: ["sleep", "5000"]
   ```

2. Specify separate fields for the command and its arguments:

   ```yaml theme={null}
   apiVersion: v1
   kind: Pod
   metadata:
     name: ubuntu-sleeper-2
   spec:
     containers:
       - name: ubuntu
         image: ubuntu
         command: ["sleep"]
         args: ["5000"]
   ```

Both methods are valid because the first element (i.e., "sleep") specifies the command and the remaining elements act as arguments. For simplicity, the first approach is recommended. After saving the file, create the pod and verify that it runs with the proper command and argument.

---

## Correcting a YAML Error with Command Arguments

Next, consider the file "ubuntu-sleeper-3.yaml". The original file defines the pod as follows:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-3
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command:
        - "sleep"
        - 1200
```

This configuration produces an error:

```Go theme={null}
cannot unmarshal number into Go value of type string
```

<Callout icon="triangle-alert" color="#FF6B6B">
  All elements in the command array must be provided as strings.
</Callout>

To correct this, enclose the number in quotes:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-3
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command:
        - "sleep"
        - "1200"
```

After making these changes, create the pod with:

```bash theme={null}
kubectl create -f ubuntu-sleeper-3.yaml
```

---

## Attempting to Update an Immutable Field in a Pod

The next task is to update the pod "ubuntu-sleeper-3" so that it sleeps for 2000 seconds instead of 1200 seconds. Attempting a direct edit with:

```yaml theme={null}
command:
  - "sleep"
  - "2000"
```

results in an error because container commands are immutable fields. To apply the change, update the local manifest file and use a forced replace command:

```bash theme={null}
kubectl replace --force -f /tmp/kubectl-edit-2693604347.yaml
```

This command deletes the existing pod and creates a new one with the updated command. Expect a brief delay as the old pod terminates and the new pod starts.

---

## Inspecting Dockerfiles and Container Commands

### Dockerfile in /root/webapp-color

Examine the following Dockerfile, which sets up a Python web application:

```dockerfile theme={null}
FROM python:3.6-alpine

RUN pip install flask

COPY . /opt/
EXPOSE 8080
WORKDIR /opt
ENTRYPOINT ["python", "app.py"]
```

This Dockerfile configures the container to run `python app.py` on startup.

### Dockerfile2 with a CMD Override

Consider Dockerfile2, which includes both an ENTRYPOINT and a CMD instruction:

```dockerfile theme={null}
FROM python:3.6-alpine

RUN pip install flask
COPY . /opt/
EXPOSE 8080
WORKDIR /opt
ENTRYPOINT ["python", "app.py"]
CMD ["--color", "red"]
```

When the image is started without any override, it effectively executes:

```Go theme={null}
python app.py --color red
```

---

## Overriding Container Commands via Kubernetes Manifests

### webapp-color-2 Directory

The directory "webapp-color-2" contains the following files:

- Dockerfile2
- webapp-color-pod.yaml

The contents of Dockerfile2 are:

```dockerfile theme={null}
FROM python:3.6-alpine

RUN pip install flask

COPY . /opt/
EXPOSE 8080
WORKDIR /opt

ENTRYPOINT ["python", "app.py"]
CMD ["--color", "red"]
```

The pod manifest (webapp-color-pod.yaml) is defined as:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: webapp-green
  labels:
    name: webapp-green
spec:
  containers:
    - name: simple-webapp
      image: kodekloud/webapp-color
      command: ["--color", "green"]
```

In this configuration, the container’s built-in command defined in the Dockerfile is overridden by the manifest, and the container starts with `--color green` instead of `--color red`.

### webapp-color-3 Directory

In the "webapp-color-3" directory, the provided files are:

**Dockerfile2:**

```dockerfile theme={null}
FROM python:3.6-alpine

RUN pip install flask

COPY . /opt/
EXPOSE 8080
WORKDIR /opt
ENTRYPOINT ["python", "app.py"]
CMD ["--color", "red"]
```

**Pod manifest (webapp-color-pod-2.yaml):**

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: webapp-green
  labels:
    name: webapp-green
spec:
  containers:
    - name: simple-webapp
      image: kodekloud/webapp-color
      command: ["python", "app.py"]
      args: ["--color", "pink"]
```

Here, both the command and the arguments are overridden. The final command executed by the container will be:

```text theme={null}
python app.py --color pink
```

---

## Creating a Pod with Overridden Startup Arguments

The final task is to create a pod that, by default, displays a blue background but needs to be updated to display green. This is achieved by overriding the startup arguments at runtime with `kubectl run` and using a double dash ("--") to separate kubectl options from container arguments. Execute the following command:

```bash theme={null}
kubectl run webapp-green --image=kodekloud/webapp-color -- --color green
```

All options after the double dash (`--`) are passed directly to the container, ensuring that the application starts with the `--color green` parameter. Verify that the pod is running and that the application displays the green color configuration.

---

For more information on Kubernetes fundamentals, check out the [Kubernetes Basics](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) documentation.
