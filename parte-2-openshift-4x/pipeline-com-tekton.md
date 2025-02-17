# 2.13 - Pipeline com Tekton

## Introdução <a id="user-content-introdu&#xE7;&#xE3;o"></a>

Este laboratório explora o uso dos pipelines OpenShift na plataforma OpenShift Container.

## Configuração dos repositórios de código-fonte <a id="user-content-configura&#xE7;&#xE3;o-dos-reposit&#xF3;rios-de-c&#xF3;digo-fonte"></a>

Você usará um aplicativo simples neste laboratório, que possui front-end e back-end. Embora você possa implantar os aplicativos aplicando os artefatos disponíveis no diretório k8s do respectivo repositório, você usará uma tarefa de pipeline para criar o aplicativo, caso ainda não exista.

1.No seu terminal, clone os dois repositórios de código-fonte que compõem o aplicativo no diretório `$HOME/pipelines`:

```text
cd $HOME
mkdir pipelines
cd pipelines
git clone https://github.com/openshift-pipelines/vote-api.git
git clone https://github.com/openshift-pipelines/vote-ui.git
```

2.Crie dois novos repositórios `vote-api` e `vote-ui` no seu github.

```text
cd $HOME/pipelines/vote-api
git remote add origin https://github.com/<usuario-do-github>/vote-api
git push gogs master

cd $HOME/pipelines/vote-ui
git remote add origin https://github.com/<usuario-do-github>/vote-ui
git push gogs master
```

3.Criar um novo projeto `workshop-pipeline`

```text
oc new-project workshop-pipeline
```

4.O OpenShift Pipelines adiciona e configura automaticamente um ServiceAccount chamado pipeline em cada projeto. Essa conta de serviço tem permissões suficientes para criar e enviar uma imagem.

execute o seguinte comando para ver a conta do serviço de pipeline:

```text
oc get serviceaccount pipeline
```

## Instalando as Tarefas Customizadas <a id="user-content-instalando-as-tarefas-customizadas"></a>

1.O OpenShift Pipelines já instala alguns aplicativos comuns **ClusterTasks** no cluster OpenShift. Você pode listar as tarefas de cluster disponíveis usando o utilitário de linha de comando `tkn`:

Saída

```text
NAME                       AGE
buildah                    5 days ago
buildah-v0-11-3            5 days ago
git-clone                  5 days ago
jib-maven                  5 days ago
kn                         5 days ago
maven                      5 days ago
openshift-client           5 days ago
openshift-client-v0-11-3   5 days ago
s2i                        5 days ago
s2i-dotnet-3               5 days ago
s2i-dotnet-3-v0-11-3       5 days ago
s2i-go                     5 days ago
s2i-go-v0-11-3             5 days ago
s2i-java-11                5 days ago
s2i-java-11-v0-11-3        5 days ago
s2i-java-8                 5 days ago
s2i-java-8-v0-11-3         5 days ago
s2i-nodejs                 5 days ago
s2i-nodejs-v0-11-3         5 days ago
s2i-perl                   5 days ago
s2i-perl-v0-11-3           5 days ago
s2i-php                    5 days ago
s2i-php-v0-11-3            5 days ago
s2i-python-3               5 days ago
s2i-python-3-v0-11-3       5 days ago
s2i-ruby                   5 days ago
s2i-ruby-v0-11-3           5 days ago
s2i-v0-11-3                5 days ago
tkn                        5 days ago
```

2.Examine a tarefa de cluster `openshift-client`:

```text
oc get clustertask openshift-client -o yaml
```

Saída

```text
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
[..]
  name: openshift-client
[..]
]spec:
  params:
  - default: oc $@
    description: The OpenShift CLI arguments to run
    name: SCRIPT
    type: string
  - default:
    - help
    description: The OpenShift CLI arguments to run
    name: ARGS
    type: array
  resources:
    inputs:
    - name: source
      optional: true
      type: git
  steps:
  - args:
    - $(params.ARGS)
    image: image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
    name: oc
    resources: {}
    script: $(params.SCRIPT)
```

3.Observe as seguintes configurações:

* A tarefa usa uma imagem `image-registry.openshift-image-registry.svc: 5000 / openshift / cli: latest`
* A tarefa espera dois parâmetros: `SCRIPT` e\` ARGS\`, ambos com padrões.
* Existe um recurso opcional, `source`. Isso pode ser usado para clonar um repositório com manifestos YAML primeiro, caso a tarefa precise criar objetos a partir do YAML.
* A tarefa possui uma etapa, chamada `oc`, que chama a imagem da CLI do OpenShift passando o parâmetro\` SCRIPT\`.

4.Você também pode criar suas próprias tarefas para executar etapas que não têm nenhuma tarefa pré-criada disponível.

5.Crie uma nova tarefa para aplicar manifestos k8s ao seu cluster

```text
cat << 'EOF' >$HOME/pipelines/task_apply_manifests.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: apply-manifests
spec:
  resources:
    inputs:
    - {type: git, name: source}
  params:
  - name: manifest_dir
    description: The directory in source that contains yaml manifests
    type: string
    default: "k8s"
  steps:
  - name: apply
    image: image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
    workingDir: /workspace/source
    command: ["/bin/bash", "-c"]
    args:
    - |-
      echo Applying manifests in $(inputs.params.manifest_dir) directory
      oc apply -f $(inputs.params.manifest_dir)
      echo -----------------------------------
EOF
```

6.Examine a definição da tarefa para entender o que ela faz.

7.Crie uma segunda tarefa que atualizará a imagem do contêiner em uma implantação:

```text
cat << 'EOF' >$HOME/pipelines/task_update_deployment.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: update-deployment
spec:
  resources:
    inputs:
    - {type: image, name: image}
  params:
  - name: deployment
    description: The name of the deployment patch the image
    type: string
  steps:
  - name: patch
    image: image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
    command: ["/bin/bash", "-c"]
    args:
    - |-
      oc patch deployment $(inputs.params.deployment) --patch='{"spec":{"template":{"spec":{
        "containers":[{
          "name": "$(inputs.params.deployment)",
          "image":"$(inputs.resources.image.url)"
        }]
      }}}}'
EOF
```

8.Mais uma vez, examine a definição da tarefa para entender como ela funciona. 9.Crie as duas tarefas:

```text
oc create -f $HOME/pipelines/task_apply_manifests.yaml
oc create -f $HOME/pipelines/task_update_deployment.yaml
```

10.Valide que suas tarefas foram criadas:

Saída

```text
NAME                AGE
apply-manifests     5 seconds ago
update-deployment   5 seconds ago
```

11.Como as tarefas são recursos do Kubernetes, você também pode usar a CLI do OpenShift para validar que suas tarefas foram criadas:

Saída

```text
NAME                AGE
apply-manifests     34s
update-deployment   34s
```

## Criar Pipeline <a id="user-content-criar-pipeline"></a>

Na próxima seção, você criará um Pipeline que usa as duas tarefas criadas, bem como a tarefa comum `buildah` para criar a imagem do contêiner para os dois aplicativos.

Os pipelines, assim como as tarefas, são projetados para serem reutilizáveis. Você criará apenas um pipeline - e depois usará parâmetros para selecionar qual aplicativo criar e implantar.

Aqui está um diagrama do pipeline que você criará.

Na caixa à direita, você vê o pipeline com as seguintes etapas:

* Usando a tarefa `buildah`, clone o código-fonte do Github, crie a imagem do contêiner e envie-a para o registro do OpenShift
* Aplique os manifestos Kubernetes no repositório de código-fonte para criar / atualizar o aplicativo
* Atualize a implantação para usar a imagem do contêiner criada recentemente \(que acionará a reimplantação do aplicativo\)

1.Criar aplicação

```text
cat << 'EOF' >$HOME/pipelines/pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  resources:
  - name: git-repo
    type: git
  - name: image
    type: image
  params:
  - name: deployment-name
    type: string
    description: name of the deployment to be patched
  tasks:
  - name: build-image
    taskRef:
      name: buildah
      kind: ClusterTask
    resources:
      inputs:
      - name: source
        resource: git-repo
      outputs:
      - name: image
        resource: image
    params:
    - name: TLSVERIFY
      value: "false"
  - name: apply-manifests
    taskRef:
      name: apply-manifests
    resources:
      inputs:
      - name: source
        resource: git-repo
    runAfter:
    - build-image
  - name: update-deployment
    taskRef:
      name: update-deployment
    resources:
      inputs:
      - name: image
        resource: image
    params:
    - name: deployment
      value: $(params.deployment-name)
    runAfter:
    - apply-manifests
EOF
```

1. Examine o pipeline e observe o seguinte:
   1. Você define dois recursos, um repositório git e uma imagem
   2. Você deve ter notado que não há referências ao repositório git ou ao registro de imagens real. Isso ocorre porque o pipeline em Tekton é projetado para ser genérico e reutilizável em ambientes e estágios ao longo do ciclo de vida do aplicativo. Os pipelines abstraem as especificidades do repositório e da imagem do git source a serem produzidos como PipelineResources.
   3. Há um parâmetro, o nome da implantação
   4. Há três tarefas listadas, com suas entradas
   5. A ordem de execução da tarefa é determinada pelas dependências definidas entre as tarefas por meio de entradas e saídas, bem como por ordens explícitas definidas por runAfter.

3.Criar o pipeline:

```text
oc create -f $HOME/pipelines/pipeline.yaml
```

4.Verifique se o pipeline foi criado:

Saída

```text
NAME               AGE              LAST RUN   STARTED   DURATION   STATUS
build-and-deploy   37 seconds ago   ---        ---       ---        ---
```

## Validar o pipeline no console do OpenShift <a id="user-content-validar-o-pipeline-no-console-do-openshift"></a>

O Operador de pipelines OpenShift também criou uma nova seção no OpenShift Console para criar, atualizar e exibir pipelines. Nesta seção, você examina o pipeline no console da web.

1. Faça logon no OpenShift Web Console
2. Mude sua perspectiva para a perspectiva \* Developer \*
3. Verifique se você está no \* seu \* projeto, `workshop-pipeline`
4. Navegue para `Pipelines` à esquerda.
5. Explore seu pipeline.
6. Quando terminar, deixe o console da web OpenShift aberto. Você usará a exibição Pipelines no Console da Web na próxima seção para seguir a execução do seu pipeline.

Antes de executar seu pipeline, você deve criar as entradas e saídas para seus pipelines. Eles são definidos nos objetos `PipelineResource`. . Crie um recurso de pipeline para o seu repositório Git `vote-ui`:

```text
echo "
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: ui-repo
spec:
  type: git
  params:
  - name: url
    value: https://github.com/openshift-pipelines/vote-ui.git
" >$HOME/pipelines/pipeline_resource_gogs_vote_ui.yaml
```

7.Crie outro recurso de pipeline para o seu repositório Git `vote-api`:

```text
echo "
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: api-repo
spec:
  type: git
  params:
  - name: url
    value: https://github.com/openshift-pipelines/vote-api.git
" >$HOME/pipelines/pipeline_resource_gogs_vote_api.yaml
```

Observe como você está configurando o parâmtetro URL para a URL específica do seu repositório do Git.

8.Crie um terceiro recurso de pipeline para a imagem do contêiner da interface do usuário criada:

```text
echo "
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: ui-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/workshop-pipeline/vote-ui:latest
" >$HOME/pipelines/pipeline_resource_image_vote_ui.yaml
```

9.Finalmente, crie um recurso de pipeline para a imagem do contêiner da API construída:

```text
echo "
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: api-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/workshop-pipeline/vote-api:latest
" >$HOME/pipelines/pipeline_resource_image_vote_api.yaml
```

10.Agora crie todos os quatro recursos de pipeline:

```text
oc create -f $HOME/pipelines/pipeline_resource_gogs_vote_ui.yaml
oc create -f $HOME/pipelines/pipeline_resource_gogs_vote_api.yaml
oc create -f $HOME/pipelines/pipeline_resource_image_vote_ui.yaml
oc create -f $HOME/pipelines/pipeline_resource_image_vote_api.yaml
```

11.E valide que eles estão todos lá:

Saída

```text
NAME        TYPE    DETAILS
api-repo    git     url: http://http://gogs-gogs-a4c4-gogs.apps.cluster-navilt.navilt.example.opentlc.com/Pipeline/vote-api.git
ui-repo     git     url: http://http://gogs-gogs-a4c4-gogs.apps.cluster-navilt.navilt.example.opentlc.com/Pipeline/vote-ui.git
api-image   image   url: image-registry.openshift-image-registry.svc:5000/a4c4-pipeline/vote-api:latest
ui-image    image   url: image-registry.openshift-image-registry.svc:5000/a4c4-pipeline/vote-ui:latest
```

12.Agora você está pronto para executar seu pipeline pela primeira vez.

Para executar o pipeline, você precisa criar uma execução de pipeline que vincule os recursos do pipeline à sua pielina.

Crie um `PipelineRun` usando o comando\` tkn\` para iniciar o pipeline `build-and-deploy` passando os recursos necessários \(\` -r\`\) e parâmetros \(`-p`\):

```text
tkn pipeline start build-and-deploy \
    -r git-repo=api-repo \
    -r image=api-image \
    -p deployment-name=vote-api
```

Saída

```text
Pipelinerun iniciado: build-and-deploy-run-l52wd

Para acompanhar o andamento do pipelinerun, execute:
tkn pipelinerun logs build-and-deploy-run-l52wd -f -n a4c4-pipeline
```

13.Valide se o seu pipeline está em execução \(você também pode verificar o console da web OpenShift\):

Saída

```text
NAME               AGE              LAST RUN                     STARTED          DURATION   STATUS
build-and-deploy   21 minutes ago   build-and-deploy-run-wj26p   19 seconds ago   ---        Running
```

14.Siga os logs do pipeline \(se você tiver mais de um pipeline em execução, o tkn solicitará a você que pipeline você deseja seguir os logs\):

Saída

```text
[build-image : git-source-api-repo-6gtwh] {"level":"info","ts":1591297908.8857565,"caller":"git/git.go:105","msg":"Successfully cloned https://github.com/osmanlirajr/vote-api.git @ master in path /workspace/source"}
[build-image : git-source-api-repo-6gtwh] {"level":"warn","ts":1591297908.885824,"caller":"git/git.go:152","msg":"Unexpected error: creating symlink: symlink /tekton/home/.ssh /root/.ssh: file exists"}

[...]

[build-image : build] STEP 1: FROM golang:alpine AS builder
[build-image : build] Getting image source signatures

[...]

build-image : push] Getting image source signatures
[build-image : push] Copying blob sha256:2da4a4a49c06b6400fd23a96be0d9b90cc0bf2341303aac1f015afe4882f9157

[...]

[apply-manifests : git-source-api-repo-ckx7l] {"level":"info","ts":1591297959.729658,"caller":"git/git.go:105","msg":"Successfully cloned https://github.com/osmanlirajr/vote-api.git @ master in path /workspace/source"}
[apply-manifests : git-source-api-repo-ckx7l] {"level":"info","ts":1591297959.7956636,"caller":"git/git.go:133","msg":"Successfully initialized and updated submodules in path /workspace/source"}

[apply-manifests : apply] Applying manifests in k8s directory
[apply-manifests : apply] deployment.apps/vote-api created
[apply-manifests : apply] service/vote-api created
[apply-manifests : apply] -----------------------------------

[update-deployment : patch] deployment.apps/vote-api patched
```

15.Quando o pipeline terminar, verifique se o aplicativo está em execução:

Saída

```text
NAME                                                           READY   STATUS      RESTARTS   AGE
build-and-deploy-run-wj26p-apply-manifests-895lv-pod-d88zs     0/2     Completed   0          119s
build-and-deploy-run-wj26p-build-image-s6l7t-pod-st6gr         0/5     Completed   0          3m24s
build-and-deploy-run-wj26p-update-deployment-k8cbs-pod-bqpxj   0/1     Completed   0          99s
vote-api-68d8d7fdb-w9vjw                                       1/1     Running     0          92s
```

Observe o seguinte:

* Seu pod `vote-api` está em execução
* Você tem três outros pods completos. Estas foram as três tarefas em seu pipeline: **build image**, **apply manifests** and **update deployment**.
* As tarefas são executadas como pods - e cada etapa de uma tarefa é executada em seu próprio contêiner. Você pode dizer que a tarefa \* **build image** teve 5 etapas.

17.Agora construa o segundo aplicativo. Você usará exatamente o mesmo pipeline - mas com diferentes entradas \(recursos e parâmetros\):

```text
tkn pipeline start build-and-deploy \
    -r git-repo=ui-repo \
    -r image=ui-image \
    -p deployment-name=vote-ui
```

Saída

```text
Pipelinerun iniciado: build-and-deploy-run-b8rw8

Para acompanhar o andamento do pipelinerun, execute:
tkn pipelinerun logs build-and-deploy-run-b8rw8 -f -n a4c4-pipeline
```

18.Mais uma vez, siga os logs do seu pipeline.

19.Depois que a execução do pipeline terminar, verifique se o seu segundo aplicativo também está em execução:

Saída

```text
NAME                                                           READY   STATUS        RESTARTS   AGE
build-and-deploy-run-b8rw8-apply-manifests-h9xzb-pod-9h74n     0/2     Completed     0          42s
build-and-deploy-run-b8rw8-build-image-8gmtt-pod-95cjm         0/5     Completed     0          105s
build-and-deploy-run-b8rw8-update-deployment-fh5xk-pod-hhc9f   0/1     Completed     0          15s
build-and-deploy-run-wj26p-apply-manifests-895lv-pod-d88zs     0/2     Completed     0          6m47s
build-and-deploy-run-wj26p-build-image-s6l7t-pod-st6gr         0/5     Completed     0          8m12s
build-and-deploy-run-wj26p-update-deployment-k8cbs-pod-bqpxj   0/1     Completed     0          6m27s
vote-api-68d8d7fdb-w9vjw                                       1/1     Running       0          6m20s
vote-ui-c867566c5-6jx7j                                        1/1     Running       0          8s
```

Você vê os pods que fazem parte da segunda execução do pipeline. E você vê o pod `vote-ui`.

21.Recupere a rota para o seu aplicativo:

```text
oc get route vote-ui --template='http://{{.spec.host}}'
```

Saída

```text
http://vote-ui-a4c4-pipeline.apps.cluster-navilt.navilt.example.opentlc.com
```

22.No seu navegador, navegue até a rota para ver o aplicativo em ação.

