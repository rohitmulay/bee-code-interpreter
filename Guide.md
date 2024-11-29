### Bee Code Interpreter Deployment Guide  

---

#### **Repository and Image Setup**  

1. Clone the repository with submodules:  
   ```bash
   git clone https://github.com/i-am-bee/bee-code-interpreter.git --recurse-submodules
   ```

2. Build the Docker image:  
   ```bash
   docker build --platform "linux/amd64" -t us.icr.io/watson-orchestrate/bee-code-interpreter:rohit .
   ```

3. If using Podman (which does not support high UIDs):  
   Modify the Dockerfile to manually include the special UID required by the cluster. Below is a sample modification to the `executor` Dockerfile: 

   ```dockerfile
   ARG UID="1000860000"
   ...
   RUN echo "UID_MIN=1" >> /etc/login.defs && \
       echo "UID_MAX=1000860000" >> /etc/login.defs && \
       groupadd --gid 1000860000 executor && \
       useradd -u 1000860000 -g 1000860000 -c 'executor' -s /bin/bash executor
   ...
   ```

4. Build the updated image:  
   ```bash
   docker build --platform "linux/amd64" -t us.icr.io/watson-orchestrate/bee-code-interpreter:rohit executor
   ```

5. Push the image to the registry:  
   ```bash
   docker push us.icr.io/watson-orchestrate/bee-code-interpreter:rohit
   ```

---

#### **Kubernetes Setup**

1. **Create a new project/namespace**:
   ```bash
   oc new-project bee-code-interpreter
   ```

2. **Create necessary resources**:
   ```yaml
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: code-interpreter-sa
     namespace: bee-code-interpreter
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: pod-manager-role
     namespace: bee-code-interpreter
   rules:
   - apiGroups: [""]
     resources: ["pods", "pods/exec"]
     verbs: ["*"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: pod-manager-binding
     namespace: bee-code-interpreter
   subjects:
   - kind: ServiceAccount
     name: code-interpreter-sa
   roleRef:
     kind: Role
     name: pod-manager-role
     apiGroup: rbac.authorization.k8s.io
   ```
   
3. **Apply Deployment, Service, and Route resources**:  
   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: code-interpreter-deployment
     namespace: bee-code-interpreter
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: code-interpreter
     template:
       metadata:
         labels:
           app: code-interpreter
       spec:
         serviceAccountName: code-interpreter-sa
         securityContext:
           fsGroup: 1000860000
         containers:
         - name: code-interpreter-service
           image: us.icr.io/watson-orchestrate/bee-code-interpreter:rohit
           ports:
           - containerPort: 50051
           - containerPort: 8000
           env:
           - name: APP_FILE_STORAGE_PATH
             value: /storage
           volumeMounts:
           - name: storage-volume
             mountPath: /storage
           securityContext:
             runAsUser: 1000860000
             allowPrivilegeEscalation: false
             capabilities:
               drop:
               - ALL
             seccompProfile:
               type: RuntimeDefault
         volumes:
         - name: storage-volume
           emptyDir: {}
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: code-interpreter-service
     namespace: bee-code-interpreter
   spec:
     selector:
       app: code-interpreter
     ports:
       - name: http-port
         protocol: TCP
         port: 80
         targetPort: 8000
       - name: grpc-port
         protocol: TCP
         port: 443
         targetPort: 50051
   ---
   apiVersion: route.openshift.io/v1
   kind: Route
   metadata:
     name: code-interpreter-route
     namespace: bee-code-interpreter
   spec:
     host: code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com
     to:
       kind: Service
       name: code-interpreter-service
       weight: 100
     port:
       targetPort: http-port
     wildcardPolicy: None
   ```

4. Add required security context constraint (SCC):
   ```bash
   oc adm policy add-scc-to-user anyuid -z code-interpreter-sa -n bee-code-interpreter
   ```

---

#### **API Usage**

1. **Direct Code Execution**:
   ```bash
   curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"source_code": "print(\"hello world\")"}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
   ```

2. **Execution via a File** (JSON-only):  
   ```bash
   curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"source_code\": \"$(cat test.py | sed 's/\"/\\\"/g')\"}" \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
   ```

3. **Execution via a Remote Python File** (Workaround):  
   ```bash
   curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"source_code\": \"$(curl -sL 'https://example.com/sample_script.py' | awk '{printf \"%s\\n\", $0}' | sed 's/\"/\\\"/g')\"}" \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
   ```

4. **Passing Arguments to the Code**:  
   Arguments must be embedded in the source code as this setup does not support separate argument passing. Here's an example workaround which would not work:  
   ```bash
   curl -X POST \
     -H "Content-Type: application/json" \
     -d "$(jq -n \
       --arg source_code "$(curl -sL 'https://example.com/add.py' | perl -pE 's/\\/\\\\/g; s/\"/\\\\\"/g')" \
       --argjson input '[5,10]' \
       '{source_code: $source_code, input: $input}')" \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
   ```

   **Response**:  
   ```json
   {
     "stdout": "",
     "stderr": "Error: Please provide two numbers.",
     "exit_code": 1,
     "files": []
   }
   ```

---

**Note:** Direct execution does not support separate argument handling, so arguments must be part of the source code itself.