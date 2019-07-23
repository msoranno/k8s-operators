# Kubernetes operators (Ansible)

## Operator SDK by redhat

https://github.com/operator-framework/operator-sdk

The SDK provides workflows to develop operators in Go, Ansible, or Helm.

The following workflow is for a new Ansible operator:

- Create a new operator project using the SDK Command Line Interface(CLI)
- Write the reconciling logic for your object using ansible playbooks and roles
- Use the SDK CLI to build and generate the operator deployment manifests
- Optionally add additional CRD's using the SDK CLI and repeat steps 2 and 3


### Install the Operator SDK CLI

https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md

```
RELEASE_VERSION=v0.9.0

 mkdir operator-sdk
 cd operator-sdk/
 curl -OJL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
 ls -alrt
 curl -OJL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu.asc
 ls -alrt
 gpg --verify operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu.asc # (this ewill return a keyID)
 gpg --recv-key "9AF46519"
 gpg --keyserver keyserver.ubuntu.com --recv-key "9AF46519"
 chmod +x operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu && sudo cp operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu /usr/local/bin/operator-sdk && rm operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
 operator-sdk version

```

### Ansible project sample

https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md


- Creamos el proyecto
```
operator-sdk new memcached-operator --api-version=cache.example.com/v1alpha1 --kind=Memcached --type=ansible

Esto genera esta estructura de directorio:

├── memcached-operator
│   ├── build
│   │   ├── Dockerfile
│   │   └── test-framewo rk
│   │       ├── ansible-test.sh
│   │       └── Dockerfile
│   ├── deploy
│   │   ├── crds
│   │   │   ├── cache_v1alpha1_memcached_crd.yaml
│   │   │   └── cache_v1alpha1_memcached_cr.yaml
│   │   ├── operator.yaml
│   │   ├── role_binding.yaml
│   │   ├── role.yaml
│   │   └── service_account.yaml
│   ├── molecule
│   │   ├── default
│   │   │   ├── asserts.yml
│   │   │   ├── molecule.yml
│   │   │   ├── playbook.yml
│   │   │   └── prepare.yml
│   │   ├── test-cluster
│   │   │   ├── molecule.yml
│   │   │   └── playbook.yml
│   │   └── test-local
│   │       ├── molecule.yml
│   │       ├── playbook.yml
│   │       └── prepare.yml
│   ├── roles
│   │   └── memcached
│   │       ├── defaults
│   │       │   └── main.yml
│   │       ├── files
│   │       ├── handlers
│   │       │   └── main.yml
│   │       ├── meta
│   │       │   └── main.yml
│   │       ├── README.md
│   │       ├── tasks
│   │       │   └── main.yml
│   │       ├── templates
│   │       └── vars
│   │           └── main.yml
│   └── watches.yaml

```

- Watches file

Realizan el mapeo entre CRD y roles o playbooks. Los operadores esperan los watches files en /opt/ansible/watches.yaml. En nuestro caso se ha creado un watches.yaml en la raiz del proyecto

```
---
- version: v1alpha1
  group: cache.example.com
  kind: Memcached
  role: /opt/ansible/roles/memcached

```

### Building the Memcached Ansible Role

- The Ansible Operator will simply pass all key value pairs listed in the Custom Resource spec field along to Ansible as variables

- The names of all variables in the spec field are converted to snake_case by the operator before running ansible. For example, serviceAccount in the spec becomes service_account in ansible

- Modificamos el roles/memcached/defaults/main.yml

```
size: 1
```

- Ahora modificamos roles/memcached/tasks/main.yml para cargar la tarea de crear el deployment usando el módulo de k8s para ansible. Como nota importante el uso de la variable {{size}}
para determinar las replicas.

```
---
- name: start memcached
  k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ meta.name }}-memcached'
        namespace: '{{ meta.namespace }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211
```

### Build and run the operator

- Kubernetes needs to know about the new custom resource definition the operator will be watching. So first, we will deploy te CRD:

```
kubectl create -f ./deploy/crds/cache_v1alpha1_memcached_crd.yaml

kubectl get crd --all-namespaces
NAME                           CREATED AT
memcacheds.cache.example.com   2019-07-20T19:08:12Z

```

- RBAC setup

```
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml

```

- there are two ways to run the operator. Una es como POD y la otra es usando el utilitario:

	- As a pod inside a Kubernetes cluster (es mas para producción)
	- As a go program outside the cluster using operator-sdk (mas usado durante el proceso de desarrollo)


#### Ejecutar el operator con el utilitario operator-sdk

- update the role path on the watches file. (must be absolute path)

```
---
- version: v1alpha1
  group: cache.example.com
  kind: Memcached
  #role: /opt/ansible/roles/memcached
  role: /home/sp81891/myutils/operator-sdk/memcached-operator/roles/memcached
```

- install ansible-runner, ansible-runner-http , openshift module (used by ansible)

> Note: es importante tener python 2.7 respondinedo al python.
> Nota2: con la versión de python 2.7 funciona pero siempre da mensajes de error **did not receive playbook_on_stats event** y tiene exceso de logs de ansible.


Utilizarmeos pyenv para crear un entorno virtual con la versión 3.7.4 de python y así poder usar una versión mas actualizada de todos los componentes. Previamente hemos instalado la versión 3.7.4 de python usando pyenv. Documentación de pyenv aqui: -> 

```
cd myutils/operator-sdk/memcached-operator/

# crea el entorno virtual
pyenv virtualenv 3.7.4 operator

# lo activamos
pyenv local operator

pip install --upgrade pip
pip install ansible-runner
pip ansible-runner-http
pip install openshift

# lo desactivamos
pyenv local system
```

- run the operator

```
cd memcached-operator

pyenv local operator

operator-sdk up local
```


#### Ejecutar el operador como pod

- create and setup a quay.io account

- build de image

```
# build the image (una especie de docker build)
operator-sdk build quay.io/msoranno/memcached-operator:v0.0.1
``` 
Como se puede ver en el Dockerfile, se utiliza como imagen base anisble-operator, se copia el watches y el directorio roles.

```
FROM quay.io/operator-framework/ansible-operator:v0.9.0
COPY watches.yaml ${HOME}/watches.yaml
COPY roles/ ${HOME}/roles/
```

- Login and push
```
# login quay.io (account needed first)
docker login quay.io

# push the image (make it public)
docker push quay.io/msoranno/memcached-operator:v0.0.1

```

- Replace operator.yaml with the appropiate image path and the correct imagePullPolicy

```
sed -i 's|{{ REPLACE_IMAGE }}|quay.io/msoranno/memcached-operator:v0.0.1|g' deploy/operator.yaml
sed -i 's|{{ pull_policy\|default('\''Always'\'') }}|Always|g' deploy/operator.yaml

```

- Deploy the memcached-operator:

```
kubectl create -f deploy/operator.yaml
```

#### Crear un custom resource (aplica ambos métodos)

Da igual como se esté ejecutando el operador, si como POD o con el operator-sdk , el comportamiento debería ser el mismo.

```
kubeclt create -f ./deploy/crds/cache_v1alpha1_memcached_cr.yaml

```

Se puede observar que al crearse el CR el operador crea 3 replicas del Memchaed usando ansible.
