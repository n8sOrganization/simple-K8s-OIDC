# Configuring OIDC for Kubernetes


This follows from my [blog post located here](https://vrelevant.net/oidc-devops-and-sre-level-part-3/). There are a series of three posts covering OAAuth and OIDC that are helpful 
in understanding the steps below.

### Signup for an Okta Developer Account

https://developer.okta.com/signup/

### Configure Directory Services and OIDC

1. From the `Directory` link in the left gutter, use the `Group` and `People` links to create a group called `k8s-cluster-admins` and a user. Set the user password to not require change on first login. Place the user in the group.

_For the flow of this guide, preface your group names with `k8s-` (e.g. k8s-cluster-admins)._

![image](https://user-images.githubusercontent.com/45366367/232320134-f70a913d-5eb6-4251-a0b6-d284d245f2a7.png)

### Create an OAuth/OIDC Appllication Definition

1. From the `Applications` link in the left gutter, select `Create App Integration`

![image](https://user-images.githubusercontent.com/45366367/232320740-4d5780df-5ffa-482c-a5fd-542fefb1fcbb.png)

2. Select `OIDC - OpenID Connect` and `Native Application`, click `Next`

![image](https://user-images.githubusercontent.com/45366367/232320814-ea9b763a-71a7-485b-8b6c-d2351de850cb.png)

3. Give it an App Integration name ok K8s. In the Sign-in and Sign-out redirect URIs specify `http://localhost:8000`. Select `Allow everyone in your organization to access`. Click `Save`.

![image](https://user-images.githubusercontent.com/45366367/232321245-279a46f1-bc6d-4923-a7d5-b2ed89d9b693.png)

4. Copy the `Client ID` and save for later. Make sure `Require PKCE as additional verification` is selected.

![image](https://user-images.githubusercontent.com/45366367/232321348-b3da067e-3f58-4baf-8da6-6d573ae7c591.png)

### Create an Authorization Server

1. From the `Security` link in the left gutter, select sub menu `API`. Click `Add Authorization Server`

![image](https://user-images.githubusercontent.com/45366367/232321528-ed786930-ae62-41ae-bf6a-981fa1afbce9.png)

2. Give it the name `K8s`. Specify http://localhost:8000 for `Audience`. Select `Okta URL` for `Issuer`. Click `Save`. 
Copy the `Issuer URL` for later.

![image](https://user-images.githubusercontent.com/45366367/232322126-e3278d1b-d255-4213-8034-e3d4a39be62c.png)

3. Select the `Claims` tab and click on `Add Claim`.

![image](https://user-images.githubusercontent.com/45366367/232322328-3a2d4e3e-c145-495b-8f14-0fccf70c6b16.png)

4. Specify `groups` for `Name`. `Include in token type` ID Token Always. `Value type` Groups. `Filter` Starts with k8s. Click `Create`

![image](https://user-images.githubusercontent.com/45366367/232322614-473a1fe8-d5f9-4f4a-930c-a58fc2fc3e9c.png)

5. Click on the `Access Policy` tab and then `Add Policy`

![image](https://user-images.githubusercontent.com/45366367/232322863-365a55b3-86b9-47cd-9786-82fea6d1ac78.png)

6. Name and Description is `K8s`. `Assign to the following clients` is `K8s`. Click `Create Policy`.

![image](https://user-images.githubusercontent.com/45366367/232323076-3e218ed8-c389-4905-897c-5dcbbe5332ac.png)

7. Click `Add Rule`

![image](https://user-images.githubusercontent.com/45366367/232323133-3a820b71-c845-45c1-b7b3-4c83a9240995.png)

8. Set config to image below and click `Create Rule`

![image](https://user-images.githubusercontent.com/45366367/232323322-e35e8a69-7b6b-4965-9dde-1fa897908374.png)

That does it for setting up your OIDC enabled Authorization Server. We can now authenticate to it and receive access and ID tokens.

We've defined a group that will be used in our cluster RBAC. We've setup an Authorization Server and copied the issuer URL to configure our Client. We've defined an auth flow for OIDC with System grant type (System is a Public Cient config that allows redirect to localhost). We've said we want to generate ID tokens and include only groups from the Directory service that begin with k8s- within the token Scope fields. Next we configure kube-api server and kubectl.

These two links provide further background on the grant type we've selected to use:
https://www.oauth.com/oauth2-servers/oauth-native-apps/
https://www.oauth.com/oauth2-servers/oauth-native-apps/redirect-urls-for-native-apps/

### Configure your kube-apiserver to trust the Auth Server minted tokens.

This step will vary based on your K8s cluster. I will show the steps for a cluster deployed by Kubeadm. If you are using a different 
deployment method for K8s, refer to your docs on how to configure the kube-apiserver.

1. Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` on each control plane node. Add the following lines to the kube-apiserver commands:

```console
spec:
  containers:
  - command:
    - kube-apiserver
    - --oidc-issuer-url=<The Issuer URL we saved earlier (just the url)>
    - --oidc-client-id=<The Client ID we saved earlier>
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
```

### Add RBAC for OIDC group

```console
kubectl apply -f - <<EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: oidc-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: k8s-cluster-admins
EOF
```

### Install and configure `kubelogin / oidc-login` plugin for `kubectl`

This config must be performed on a client with a web browser. It will not work for hosts you are SSH connected to. There 
is an Authorization code flow with a keyboard that is similar to how you enter a code in a browser when you sign-in to 
smart tv streaming apps. I'm not covering it here, but it's in the kubelogin docs.


The installation steps vary based on your OS. Rather than recreate the docs for it, I'll point you to the 
repo where you can perform the task. I find the easiest way across OS platforms is to use the kubectl krew 
plugin manager.

https://github.com/int128/kubelogin

### Edit your kube config 

1. Add the following under users section (Add the correct issuer url and client id):

```console
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=<issuer url>
      - --oidc-client-id=<client id>
      - --oidc-extra-scope="email offline_access profile openid"
```

2. Use the `kubectl` command with the --user=oidc flag.

```
kubectl get nodes --user=oidc
```

This will pop you to your browser with a login page. Sign in as your user and that's it!

You can use the same Authorization server for as many clusters as you'd like. Simply configure the cluster RBAC to grant 
privileges based on group names from your IdP Directory Service. 
