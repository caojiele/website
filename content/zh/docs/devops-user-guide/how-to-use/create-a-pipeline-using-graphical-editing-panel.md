---
title: "使用图形编辑面板创建流水线"
keywords: 'KubeSphere, Kubernetes, Jenkins, CICD, 图形化流水线'
description: '学习如何使用 KubeSphere 图形编辑面板创建并运行流水线。'
linkTitle: '使用图形编辑面板创建流水线'
weight: 11220
---

KubeSphere 中的图形编辑面板包含用于 Jenkins [阶段 (Stage)](https://www.jenkins.io/zh/doc/book/pipeline/#阶段) 和[步骤 (Step)](https://www.jenkins.io/zh/doc/book/pipeline/#步骤) 的所有必要操作。您可以直接在交互式面板上定义这些阶段和步骤，无需创建任何 Jenkinsfile。

本教程演示如何在 KubeSphere 中使用图形编辑面板创建流水线。KubeSphere 在整个过程中将根据您在编辑面板上的设置自动生成 Jenkinsfile，您无需手动创建 Jenkinsfile。待流水线成功运行，它会相应地在您的开发环境中创建一个部署 (Deployment) 和一个服务 (Service)，并将镜像推送至 Docker Hub。

## 准备工作

- 您需要[启用 KubeSphere DevOps 系统](../../../pluggable-components/devops/)。
- 您需要有一个 [Docker Hub](http://www.dockerhub.com/) 帐户。
- 您需要创建一个企业空间、一个 DevOps 工程和一个帐户 (`project-regular`)，必须邀请该帐户至 DevOps 工程中并赋予 `operator` 角色。如果尚未创建，请参见[创建企业空间、项目、帐户和角色](../../../quick-start/create-workspace-and-project/)。
- 设置 CI 专用节点来运行流水线。有关更多信息，请参见[为缓存依赖项设置 CI 节点](../../../devops-user-guide/how-to-use/set-ci-node/)。
- 配置您的电子邮件服务器用于接收流水线通知（可选）。有关更多信息，请参见[为 KubeSphere 流水线设置电子邮件服务器](../../../devops-user-guide/how-to-use/jenkins-email/)。
- 配置 SonarQube 将代码分析纳入流水线中（可选）。有关更多信息，请参见[将 SonarQube 集成到流水线](../../../devops-user-guide/how-to-integrate/sonarqube/)。

## 流水线概述

本示例流水线包括以下六个阶段。

![Pipeline](https://pek3b.qingstor.com/kubesphere-docs/png/20190516091714.png#align=left&display=inline&height=1278&originHeight=1278&originWidth=2190&search=&status=done&width=2190)

{{< notice note >}} 

- **阶段 1：Checkout SCM**：从 GitHub 仓库拉取源代码。
- **阶段 2：单元测试**：待该测试通过后才会进行下一阶段。
- **阶段 3：代码分析**：配置 SonarQube 用于静态代码分析。
- **阶段 4：构建并推送**：构建镜像并附上标签 `snapshot-$BUILD_NUMBER` 推送至 Docker Hub，其中 `$BUILD_NUMBER` 是流水线活动列表中的记录的序列号。
- **阶段 5：制品**：生成一个制品（jar 文件包）并保存。
- **阶段 6：部署至开发环境**：在开发环境中创建一个部署和一个服务。该阶段需要进行审核，部署成功运行后，会发送电子邮件通知。

{{</ notice >}}

## 动手实验

### 步骤 1：创建凭证

1. 以 `project-regular` 身份登录 KubeSphere 控制台。转到您的 DevOps 工程，在**工程管理**下的**凭证**页面创建以下凭证。有关如何创建凭证的更多信息，请参见[凭证管理](../credential-management/)。

   {{< notice note >}} 

   如果您的帐户或密码中有任何特殊字符，例如 `@` 和 `$`，可能会因为无法识别而在流水线运行时导致错误。在这种情况下，您需要先在一些第三方网站（例如 [urlencoder](https://www.urlencoder.org/)）上对帐户或密码进行编码，然后将输出结果复制粘贴作为您的凭证信息。

   {{</ notice >}} 

   | 凭证 ID         | 类型       | 用途       |
   | --------------- | ---------- | ---------- |
   | dockerhub-id    | 帐户凭证   | Docker Hub |
   | demo-kubeconfig | kubeconfig | Kubernetes |

2. 您还需要为 SonarQube 创建一个凭证 ID (`sonar-token`)，用于上述的阶段 3（代码分析）。请参考[为新工程创建 SonarQube 令牌 (Token)](../../../devops-user-guide/how-to-integrate/sonarqube/#create-sonarqube-token-for-new-project)，在下图所示的**密钥**字段中输入令牌。点击**确定**完成操作。

   ![Sonar 令牌](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/sonar-token.PNG)

3. 您可以在列表中看到已创建的三个凭证。

   ![凭证列表](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/credential-list.PNG)

### 步骤 2：创建项目

在本教程中，示例流水线会将 [sample](https://github.com/kubesphere/devops-java-sample/tree/sonarqube) 应用部署至一个项目。因此，您必须先创建一个项目（例如 `kubesphere-sample-dev`）。待流水线成功运行，会在该项目中自动创建该应用的部署和服务。

您可以使用 `project-admin` 帐户创建项目。此外，该帐户也是 CI/CD 流水线的审核员。请确保将 `project-regular` 帐户邀请至该项目并授予 `operator` 角色。有关更多信息，请参见[创建企业空间、项目、帐户和角色](../../../quick-start/create-workspace-and-project/)。

### 步骤 3：创建流水线

1. 请确保以 `project-regular` 身份登录 KubeSphere 控制台，转到您的 DevOps 工程。在**流水线**页面点击**创建**。

   ![创建流水线](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/create-pipeline.PNG)

2. 在弹出对话框中，将它命名为 `graphical-pipeline`，点击**下一步**。

   ![基本信息](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/basic-info.PNG)

3. 在**高级设置**页面，点击**添加参数**以添加以下三个字符串参数。这些参数将用于流水线的 Docker 命令。添加完成后，点击**创建**。

   ![添加参数](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/add-parameter.PNG)

   | 参数类型 | 名称                | 值              | 描述信息                                   |
   | -------- | ------------------- | --------------- | ------------------------------------------ |
   | 字符串   | REGISTRY            | `docker.io`     | 镜像仓库地址。本示例使用 `docker.io`。     |
   | 字符串   | DOCKERHUB_NAMESPACE | Docker ID       | 您的 Docker Hub 帐户或该帐户下的组织名称。 |
   | 字符串   | APP_NAME            | `devops-sample` | 应用名称。                                 |

   {{< notice note >}}

   有关其他字段，请直接使用默认值或者参考[流水线设置](../pipeline-settings/)以自定义配置。

   {{</ notice >}} 

4. 创建的流水线会显示在列表中。

   ![流水线列表](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/pipeline-list.PNG)
   

### 步骤 4：编辑流水线

点击流水线进入其详情页面。要使用图形编辑面板，请点击**流水线**选项卡下的**编辑流水线**。在弹出对话框中，点击**自定义流水线**。该流水线包括六个阶段，请按照以下步骤设置每个阶段。

![编辑流水线](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/edit-pipeline.PNG)

{{< notice note >}}

您也可以点击**编辑 Jenkinsfile** 手动为流水线创建一个 Jenkinsfile。

{{</ notice >}} 

![click-custom-pipeline](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/click-custom-pipeline.png)

{{< notice note >}}

流水线详情页面会显示**同步状态**，它是 KubeSphere 和 Jenkins 之间的同步结果。若同步成功，您会看到**成功**图标。您也可以点击**持续集成 (CI)** 和**持续集成&交付 (CI/CD)** 来使用 KubeSphere 提供的内置流水线模板。

{{</ notice >}}

#### 阶段 1：拉取源代码 (Checkout SCM)

图形编辑面板包括两个区域：左侧的**画布**和右侧的**内容**。它会根据您对不同阶段和步骤的配置自动生成一个 Jenkinsfile，为开发者提供更加用户友好的操作体验。

{{< notice note >}}

流水线包括[声明式流水线](https://www.jenkins.io/zh/doc/book/pipeline/syntax/#声明式流水线)和[脚本化流水线](https://www.jenkins.io/zh/doc/book/pipeline/syntax/#脚本化流水线)。目前，您可以使用该面板创建声明式流水线。有关流水线语法的更多信息，请参见 [Jenkins 文档](https://www.jenkins.io/zh/doc/book/pipeline/syntax/)。

{{</ notice >}}

1. 在图形编辑面板上，从**类型**下拉列表中选择 **node**，在 **label** 中输入 `maven`。

   {{< notice note >}}

   `agent` 用于定义执行环境。`agent` 指令指定 Jenkins 执行流水线的位置和方式。有关更多信息，请参见[选择 Jenkins Agent](../choose-jenkins-agent/)。

   {{</ notice >}} 

   ![图形面板](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/graphical-panel.PNG)

2. 请点击左侧的加号图标来添加阶段。点击**添加步骤**上方的文本框，然后在右侧的**名称**字段中为该阶段设置名称（例如 `Checkout SCM`）。

   ![编辑面板](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/edit-panel.PNG)

3. 点击**添加步骤**。在列表中选择 **git**，以从 GitHub 拉取示例代码。在弹出对话框中，填写必需的字段。点击**确定**完成操作。

   - **Url**：输入 GitHub 仓库地址 `https://github.com/kubesphere/devops-java-sample.git`。请注意，这里是示例地址，您需要使用您自己的仓库地址。
   - **凭证 ID**：本教程中无需输入凭证 ID。
   - **分支**：如果您将其留空，则默认为 master 分支。请输入 `sonarqube`，或者如果您不需要代码分析阶段，请将其留空。

   ![输入仓库 URL](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/enter-repo-url.PNG)

4. 第一阶段设置完成。

   ![第一阶段设置完成](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/first-stage-set.PNG)

#### 阶段 2：单元测试

1. 点击阶段 1 右侧的加号图标添加新的阶段，以在容器中执行单元测试。将它命名为 `Unit Test`。

   ![单元测试](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/unit-test.PNG)

2. 点击添加步骤，在列表中选择**指定容器**。将其命名为 `maven` 然后点击**确定**。

   ![指定容器](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/container.PNG)

3. 点击**添加嵌套步骤**，在 `maven` 容器下添加一个嵌套步骤。在列表中选择 **shell** 并在命令行中输入以下命令。点击**确定**保存操作。

   ```shell
   mvn clean -o -gs `pwd`/configuration/settings.xml test
   ```

   {{< notice note >}}

   您可以在图形编辑面板上指定在给定阶段指令中执行的一系列[步骤](https://www.jenkins.io/zh/doc/book/pipeline/syntax/#steps)。

   {{</ notice >}} 

   ![shell](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/shell.PNG)
   
   ![单元测试设置完成](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/unit-test-set.PNG)
   

#### 阶段 3：代码分析（可选）

本阶段使用 SonarQube 来测试您的代码。如果您不需要代码分析，可以跳过该阶段。

1. 点击 `Unit Test` 阶段右侧的加号图标添加一个阶段，以在容器中进行 SonarQube 代码分析。将它命名为 `Code Analysis`。

   ![代码分析阶段](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/code-analysis-stage.PNG)

2. 在 **Code Analysis** 中，点击**任务**下的**添加步骤**，选择**指定容器**。将其命名为 `maven` 然后点击**确定**。

   ![Maven 容器](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/maven-container.PNG)

3. 点击 `maven` 容器下的**添加嵌套步骤**，以添加一个嵌套步骤。点击**添加凭证**并从**凭证 ID** 列表中选择 SonarQube 令牌 (`sonar-token`)。在**文本变量**中输入 `SONAR_TOKEN`，然后点击**确定**。

   ![SonarQube 凭证](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/sonarqube-credentials.PNG)

4. 在**添加凭证**步骤下，点击**添加嵌套步骤**为其添加一个嵌套步骤。

   ![嵌套步骤](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/nested-step.PNG)

5. 点击 **Sonarqube 配置**，在弹出对话框中保持默认名称 `sonar` 不变，点击**确定**保存操作。

   ![sonar](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/sonar.PNG)

6. 在 **Sonarqube 配置**步骤下，点击**添加嵌套步骤**为其添加一个嵌套步骤。

   ![添加嵌套步骤](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/add-nested-step.PNG)

7. 点击 **shell** 并在命令行中输入以下命令，用于 sonarqube 分支和认证，点击**确定**完成操作。

   ```shell
   mvn sonar:sonar -o -gs `pwd`/configuration/settings.xml -Dsonar.login=$SONAR_TOKEN
   ```

   ![新的 SonarQube shell](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/sonarqube-shell-new.PNG)

8. 点击**指定容器**步骤下的**添加嵌套步骤**（第三个），选择**超时**。在时间中输入 `1` 并将单位选择为**小时**，点击**确定**完成操作。

   ![添加嵌套步骤-2](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/add-nested-step-2.PNG)

   ![超时](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/timeout.PNG)

9. 点击**超时**步骤下的**添加嵌套步骤**，选择**代码质量检查 (SonarQube)**。在弹出对话框中选择**检查通过后开始后续任务**。点击**确定**保存操作。

   ![waitforqualitygate](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/waitforqualitygate.PNG)

   ![sonar 就绪](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/sonar-ready.PNG)

#### 阶段 4：构建并推送镜像

1. 点击前一个阶段右侧的加号图标添加一个新的阶段，以构建并推送镜像至 Docker Hub。将其命名为 `Build and Push`。

   ![构建并推送镜像](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/build-and-push-image.PNG)

2. 点击**任务**下的**添加步骤**，选择**指定容器**，将其命名为 `maven`，然后点击**确定**。

   ![maven 设置完成](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/maven-set.PNG)

3. 点击 `maven` 容器下的**添加嵌套步骤**添加一个嵌套步骤。在列表中选择 **shell** 并在弹出窗口中输入以下命令，点击**确定**完成操作。

   ```shell
   mvn -o -Dmaven.test.skip=true -gs `pwd`/configuration/settings.xml clean package
   ```

   ![maven 嵌套步骤](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/nested-step-maven.PNG)

4. 再次点击**添加嵌套步骤**，选择 **shell**。在命令行中输入以下命令，以根据 [Dockerfile](https://github.com/kubesphere/devops-java-sample/blob/sonarqube/Dockerfile-online) 构建 Docker 镜像。点击**确定**确认操作。

   {{< notice note >}}

   请勿遗漏命令末尾的点 `.`。

   {{</ notice >}} 

   ```shell
   docker build -f Dockerfile-online -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .
   ```

   ![shell 命令](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/shell-command.PNG)

5. 再次点击**添加嵌套步骤**，选择**添加凭证**。在弹出对话框中填写以下字段，点击**确定**确认操作。

   - **凭证 ID**：选择您创建的 Docker Hub 凭证，例如 `dockerhub-id`。
   - **密码变量**：输入 `DOCKER_PASSWORD`。
   - **用户名变量**：输入 `DOCKER_USERNAME`。

   {{< notice note >}} 

   出于安全原因，帐户信息在脚本中显示为变量。

   {{</ notice >}} 

   ![docker 凭证](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/docker-credential.PNG)

6. 在**添加凭证**步骤中点击**添加嵌套步骤**（第一个）。选择 **shell** 并在弹出窗口中输入以下命令，用于登录 Docker Hub。点击**确定**确认操作。

   ```shell
   echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin
   ```

   ![Docker 登录命令](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/login-docker-command.PNG)

7. 在**添加凭证**步骤中点击**添加嵌套步骤**。选择 **shell** 并输入以下命令，将 SNAPSHOT 镜像推送至 Docker Hub。点击**确定**完成操作。

   ```shell
   docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER
   ```

   ![推送 snapshot 至 Docker](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/push-snapshot-to-docker.PNG)

#### 阶段 5：生成制品

1. 点击 **Build and Push** 阶段右侧的加号图标添加一个新的阶段，以保存制品，将其命名为 `Artifacts`。本示例使用 jar 文件包。

   ![添加制品阶段](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/add-artifact-stage.PNG)

2. 选中 **Artifacts** 阶段，点击**任务**下的**添加步骤**，选择**保存制品**。在弹出对话框中输入 `target/*.jar`，用于设置 Jenkins 中制品的保存路径。点击**确定**完成操作。

   ![制品信息](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/artifact-info.PNG)

#### 阶段 6：部署至开发环境

1. 点击 **Artifacts** 阶段右侧的加号图标添加最后一个阶段，将其命名为 `Deploy to Dev`。该阶段用于将资源部署至您的开发环境（即 `kubesphere-sample-dev` 项目）。

   ![部署至开发环境](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/deploy-to-dev.PNG)

2. 点击 **Deploy to Dev** 阶段下的**添加步骤**，在列表中选择**审核**，然后在**消息**字段中填入 `@project-admin`，即 `project-admin` 帐户在流水线运行到该阶段时会进行审核。点击**确定**保存操作。

   ![输入信息](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/input-message.PNG)

   {{< notice note >}}

   在 KubeSphere 3.1 中，能够运行流水线的帐户也能够继续或终止该流水线。此外，流水线创建者、拥有该工程管理员角色的帐户或者您指定的帐户也有权限继续或终止流水线。

   {{</ notice >}}

3. 再次点击 **Deploy to Dev** 阶段下的**添加步骤**。在列表中选择 **kubernetesDeploy** 并在弹出对话框中填写以下字段。点击**确定**保存操作。

   - **Kubeconfig**：选择您创建的 Kubeconfig，例如 `demo-kubeconfig`。
   - **配置文件路径**：输入 `deploy/no-branch-dev/**`，即代码仓库中 Kubernetes 资源 [YAML](https://github.com/kubesphere/devops-java-sample/tree/sonarqube/deploy/no-branch-dev) 文件的相对路径。

   ![kubernetesDeploy](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/kubernetesDeploy.PNG)

4. 如果您想在流水线成功运行时接收电子邮件通知，请点击**添加步骤**，选择**邮件**，以添加电子邮件信息。请注意，配置电子邮件服务器是可选操作，如果您跳过该步骤，依然可以运行流水线。

   {{< notice note >}}

   有关配置电子邮件服务器的更多信息，请参见[为 KubeSphere 流水线设置电子邮件服务器](../jenkins-email/)。

   {{</ notice >}} 

5. 待您完成上述步骤，请在右下角点击**确认**和**保存**。随后，您可以看到该流水线有完整的工作流，并且每个阶段也清晰列示。当您用图形编辑面板定义流水线时，KubeSphere 会自动创建相应的 Jenkinsfile。点击**编辑 Jenkinsfile** 查看该 Jenkinsfile。

   ![流水线设置完成](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/pipeline-done.PNG)
   
   {{< notice note >}}
   
   在**流水线**页面，您可以点击该流水线右侧的三个点，然后选择**复制流水线**来创建该流水线的副本。如果您需要同时运行多个不包含多分支的流水线，您可以全部选中这些流水线，然后点击**运行**来批量运行它们。
   
   {{</ notice >}}

### 步骤 5：运行流水线

1. 您需要手动运行使用图形编辑面板创建的流水线。点击**运行**，您可以在弹出对话框中看到步骤 3 中已定义的三个字符串参数。点击**确定**来运行流水线。

   ![运行流水线](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/run-pipeline.PNG)
   
2. 要查看流水线的状态，请转到**活动**选项卡，点击您想查看的记录。

3. 稍等片刻，流水线如果成功运行，则会在 **Deploy to Dev** 阶段停止。`project-admin` 作为流水线的审核员，需要进行审批，然后资源才会部署至开发环境。

   ![流水线成功运行](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/pipeline-successful.PNG)

4. 登出 KubeSphere 控制台，以 `project-admin` 身份重新登录。转到您的 DevOps 工程，点击 `graphical-pipeline` 流水线。在**活动**选项卡下，点击要审核的记录。要批准流水线，请点击**继续**。

### 步骤 6：查看流水线详情

1. 以 `project-regular` 身份重新登录控制台。转到您的 DevOps 工程，点击 `graphical-pipeline` 流水线。在**活动**选项卡下，点击**状态**下标记为**成功**的记录。

2. 如果所有配置都成功运行，您可以看到所有阶段都已完成。

   ![完成](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/complete.PNG)

3. 在右上角点击**查看日志**，查看所有日志。点击每个阶段查看其详细日志。您可以根据日志排除故障和问题，也可以将日志下载到本地进行进一步分析。

   ![查看日志](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/inspect-logs.PNG)

### 步骤 7：下载制品

点击**制品**选项卡，然后点击右侧的图标下载该制品。

![下载制品](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/download-artifact.PNG)

### 步骤 8：查看代码分析结果

在**代码质量**页面，可以查看由 SonarQube 提供的本示例流水线的代码分析结果。如果您没有事先配置 SonarQube，则该部分不可用。有关更多信息，请参见[将 SonarQube 集成到流水线](../../../devops-user-guide/how-to-integrate/sonarqube/)。

![SonarQube 详细结果](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/sonarqube-result-detail.PNG)

### 步骤 9：验证 Kubernetes 资源

1. 如果流水线的每个阶段都成功运行，则会自动构建一个 Docker 镜像并推送至您的 Docker Hub 仓库。最终，流水线将在您事先设置的项目中自动创建一个部署和一个服务。

2. 前往该项目（本教程中即 `kubesphere-sample-dev`），请点击**应用负载**下的**工作负载**，您可以看到列表中显示的部署。

   ![查看部署](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/view-deployment.PNG)

3. 在**服务**页面，您可以看到示例服务通过 NodePort 暴露其端口号。要访问服务，请访问 `node IP:port number`。

   ![服务暴露](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/service-exposed.PNG)

   ![访问服务](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/access-service.PNG)

   {{< notice note >}}

   访问服务前，您可能需要配置端口转发规则并在安全组中放行该端口。

   {{</ notice >}} 

4. 现在流水线已成功运行，将会推送一个镜像至 Docker Hub。登录 Docker Hub 查看结果。

   ![DockerHub 镜像](/images/docs/zh-cn/devops-user-guide/use-devops/create-a-pipeline-using-graphical-editing-panel/dockerhub-image.PNG)

5. 该应用的名称为 `devops-sample`，即 `APP_NAME` 的值，标签即 `SNAPSHOT-$BUILD_NUMBER` 的值。`$BUILD_NUMBER` 即**活动**选项卡列示的记录的序列号。

6. 如果您在最后一个阶段设置了电子邮件服务器并添加了电子邮件通知的步骤，您还会收到电子邮件消息。

## 另请参见

[使用 Jenkinsfile 创建流水线](../create-a-pipeline-using-jenkinsfile/)

[选择 Jenkins Agent](../choose-jenkins-agent/)

[为 KubeSphere 流水线设置电子邮件服务器](../jenkins-email/)