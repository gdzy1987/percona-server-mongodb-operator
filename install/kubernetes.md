Install Percona server for MongoDB on Kubernetes
------------------------------------------------

1. The first thing to do is to add the 'psmdb' namespace to Kubernetes, not forgetting to set the correspondent context for further steps:

   ```bash
   $ kubectl create namespace psmdb
   $ kubectl config set-context $(kubectl config current-context) --namespace=psmdb
   ```

2. Now that’s time to add the MongoDB Users secrets to Kubernetes. They should be placed in the data section of the `deploy/mongodb-users.yaml` file as base64-encoded logins and passwords for the MongoDB specific user accounts (see https://kubernetes.io/docs/concepts/configuration/secret/ for details). After editing is finished, mongodb-users secrets should be created (or updated with the new passwords) using the following command:

   ```bash
   $ kubectl create -f deploy/mongodb-users.yaml
   ```

   More details about secrets can be found in a [separate section](./psmdb-operator.install.md#more-on-required-secrets).

   **Note:** *the following command can be used to get base64-encoded password from a plain text string:* `$ echo -n 'plain-text-password' | base64`

3. Now RBAC (role-based access control) and Custom Resource Definition for PSMDB should be created from the following two files: `deploy/rbac.yaml` and `deploy/crd.yaml`. Briefly speaking, role-based access is based on specifically defined roles and actions corresponding to them, allowed to be done on specific Kubernetes resources (details about users and roles can be found in [Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)). Custom Resource Definition extends the standard set of resources which Kubernetes “knows” about with the new items (in our case ones which are the core of the percona-server-mongodb-operator). 

   ```bash
   $ kubectl create -f deploy/crd.yaml -f deploy/rbac.yaml
   ```

   **Note:** *This step requires your user to have cluster-admin role privileges. For example, those using Google Kubernetes Engine can grant user needed privileges with the following command:* `$ kubectl create clusterrolebinding cluster-admin-binding1 --clusterrole=cluster-admin --user=<myname@example.org>`

4. Finally it’s time to start the percona-server-mongodb-operator within Kubernetes:

   ```bash
   $ kubectl create -f deploy/operator.yaml
   ```
5. After the operator is started, Percona Server for MongoDB cluster can be created at any time with the following command:

   ```bash
   $ kubectl apply -f deploy/cr.yaml
   ```

   Creation process will take some time. The process is over when both operator and replica set pod have reached their Running status:

   ```bash
   $ kubectl get pods
   NAME                                               READY   STATUS    RESTARTS   AGE
   my-cluster-name-rs0-0                              1/1     Running   0          8m
   my-cluster-name-rs0-1                              1/1     Running   0          8m
   my-cluster-name-rs0-2                              1/1     Running   0          7m
   percona-server-mongodb-operator-754846f95d-sf6h6   1/1     Running   0          9m
   ```

7. Check connectivity to newly created cluster

   ```bash
   $ kubectl run -i --rm --tty percona-client --image=percona/percona-server-mongodb:3.6 --restart=Never -- bash -il percona-client:/$ mongo "mongodb+srv://userAdmin:userAdmin123456@my-cluster-name-rs0.psmdb.svc.cluster.local/admin?replicaSet=rs0&ssl=false"
   ```