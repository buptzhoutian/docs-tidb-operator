---
title: 使用 BR 工具恢复 S3 兼容存储上的备份数据
summary: 介绍如何使用 BR 工具恢复 Amazon S3 兼容存储上的备份数据。
aliases: ['/docs-cn/tidb-in-kubernetes/dev/restore-from-aws-s3-using-br/']
---

# 使用 BR 工具恢复 S3 兼容存储上的备份数据

本文详细描述了如何将存储在 Amazon S3 存储的备份数据恢复到 AWS Kubernetes 环境中的 TiDB 集群，底层通过使用 [BR](https://pingcap.com/docs-cn/stable/br/backup-and-restore-tool/) 进行数据恢复。

本文使用的恢复方式基于 TiDB Operator 新版（v1.1 及以上）的 Custom Resource Definition (CRD) 实现。

以下示例将 Amazon S3 的存储（指定路径）上的备份数据恢复到 AWS Kubernetes 环境中的 TiDB 集群。

## AWS 账号的三种权限授予方式

参考[使用 BR 工具备份 AWS 上的 TiDB 集群](backup-to-aws-s3-using-br.md#aws-账号权限授予的三种方式)

## 环境准备

> **注意：**
>
> 如果使用 TiDB Operator >= v1.1.7 && TiDB >= v4.0.8, BR 会自动调整 `tikv_gc_life_time` 参数，以下创建 `backup-demo1-tidb-secret` secret 的步骤可以省略。

### 通过 AccessKey 和 SecretKey 授权

1. 下载文件 [backup-rbac.yaml](https://github.com/pingcap/tidb-operator/blob/master/manifests/backup/backup-rbac.yaml)，并执行以下命令在 `test2` 这个 namespace 中创建备份需要的 RBAC 相关资源：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f backup-rbac.yaml -n test2
    ```

2. 创建 `s3-secret` secret。该 secret 存放用于访问 S3 兼容存储的凭证。

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl create secret generic s3-secret --from-literal=access_key=xxx --from-literal=secret_key=yyy --namespace=test2
    ```

3. 创建 `restore-demo2-tidb-secret` secret。该 secret 存放用于访问 TiDB 集群的 root 账号和密钥。

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl create secret generic restore-demo2-tidb-secret --from-literal=password=${password} --namespace=test2
    ```

### 通过 IAM 绑定 Pod 授权

1. 下载文件 [backup-rbac.yaml](https://github.com/pingcap/tidb-operator/blob/master/manifests/backup/backup-rbac.yaml)，并执行以下命令在 `test2` 这个 namespace 中创建备份需要的 RBAC 相关资源：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f backup-rbac.yaml -n test2
    ```

2. 创建 `restore-demo2-tidb-secret` secret。该 secret 存放用于访问 TiDB 集群的 root 账号和密钥：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl create secret generic restore-demo2-tidb-secret --from-literal=password=${password} --namespace=test2
    ```

3. 创建 IAM 角色：

    可以参考 [AWS 官方文档](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) 来为账号创建一个 IAM 角色，并且通过 [AWS 官方文档](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html)为 IAM 角色赋予需要的权限。由于 `Restore` 需要访问 AWS 的 S3 存储，所以这里给 IAM 赋予了 `AmazonS3FullAccess` 的权限。

4. 绑定 IAM 到 TiKV Pod:

    在使用 BR 备份的过程中，TiKV Pod 和 BR Pod 一样需要对 S3 存储进行读写操作，所以这里需要给 TiKV Pod 打上 annotation 来绑定 IAM 角色。

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl edit tc demo2 -n test2
    ```

    找到 `spec.tikv.annotations`, 增加 annotation `arn:aws:iam::123456789012:role/user`, 然后退出编辑, 等到 TiKV Pod 重启后，查看 Pod 是否加上了这个 annotation。

> **注意：**
>
> `arn:aws:iam::123456789012:role/user` 为步骤 4 中创建的 IAM 角色。

### 通过 IAM 绑定 ServiceAccount 授权

1. 下载文件 [backup-rbac.yaml](https://github.com/pingcap/tidb-operator/blob/master/manifests/backup/backup-rbac.yaml)，并执行以下命令在 `test2` 这个 namespace 中创建备份需要的 RBAC 相关资源：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f backup-rbac.yaml -n test2
    ```

2. 创建 `restore-demo2-tidb-secret` secret。该 secret 存放用于访问 TiDB 集群的 root 账号和密钥:

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl create secret generic restore-demo2-tidb-secret --from-literal=password=${password} --namespace=test2
    ```

3. 在集群上为服务帐户启用 IAM 角色：

    可以参考 [AWS 官方文档](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)开启所在的 EKS 集群的 IAM 角色授权。

4. 创建 IAM 角色:

    可以参考 [AWS 官方文档](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html) 创建一个 IAM 角色，为角色赋予 `AmazonS3FullAccess` 的权限，并且编辑角色的 `Trust relationships`。

5. 绑定 IAM 到 ServiceAccount 资源上：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl annotate sa tidb-backup-manager -n eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/user --namespace=test2
    ```

6. 将 ServiceAccount 绑定到 TiKV Pod：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl edit tc demo2 -n test2
    ```

    将 `spec.tikv.serviceAccount` 修改为 tidb-backup-manager , 等到 TiKV Pod 重启后，查看 Pod 的 `serviceAccountName` 是否有变化。

> **注意：**
>
> `arn:aws:iam::123456789012:role/user` 为步骤 4 中创建的 IAM 角色。

## 数据库账户权限

* `mysql.tidb` 表的 `SELECT` 和 `UPDATE` 权限：恢复前后，restore CR 需要一个拥有该权限的数据库账户，用于调整 GC 时间

## 将指定备份数据恢复到 TiDB 集群

+ 创建 `Restore` CR，通过 accessKey 和 secretKey 授权的方式恢复集群：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f resotre-aws-s3.yaml
    ```

    `restore-aws-s3.yaml` 文件内容如下：

    ```yaml
    ---
    apiVersion: pingcap.com/v1alpha1
    kind: Restore
    metadata:
      name: demo2-restore-s3
      namespace: test2
    spec:
      br:
        cluster: demo2
        clusterNamespace: test2
        # logLevel: info
        # statusAddr: ${status_addr}
        # concurrency: 4
        # rateLimit: 0
        # timeAgo: ${time}
        # checksum: true
        # sendCredToTikv: true
      # Only needed for TiDB Operator < v1.1.7 or TiDB < v4.0.8
      to:
        host: ${tidb_host}
        port: ${tidb_port}
        user: ${tidb_user}
        secretName: restore-demo2-tidb-secret
      s3:
        provider: aws
        secretName: s3-secret
        region: us-west-1
        bucket: my-bucket
        prefix: my-folder
    ```

+ 创建 `Restore` CR，通过 IAM 绑定 Pod 授权的方式备份集群：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f restore-aws-s3.yaml
    ```

    `restore-aws-s3.yaml` 文件内容如下：

    ```yaml
    ---
    apiVersion: pingcap.com/v1alpha1
    kind: Restore
    metadata:
      name: demo2-restore-s3
      namespace: test2
      annotations:
        iam.amazonaws.com/role: arn:aws:iam::123456789012:role/user
    spec:
      br:
        cluster: demo2
        sendCredToTikv: false
        clusterNamespace: test2
        # logLevel: info
        # statusAddr: ${status_addr}
        # concurrency: 4
        # rateLimit: 0
        # timeAgo: ${time}
        # checksum: true
      # Only needed for TiDB Operator < v1.1.7 or TiDB < v4.0.8
      to:
        host: ${tidb_host}
        port: ${tidb_port}
        user: ${tidb_user}
        secretName: restore-demo2-tidb-secret
      s3:
        provider: aws
        region: us-west-1
        bucket: my-bucket
        prefix: my-folder
    ```

+ 创建 `Restore` CR，通过 IAM 绑定 ServiceAccount 授权的方式备份集群：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl apply -f restore-aws-s3.yaml
    ```

    `restore-aws-s3.yaml` 文件内容如下：

    ```yaml
    ---
    apiVersion: pingcap.com/v1alpha1
    kind: Restore
    metadata:
      name: demo2-restore-s3
      namespace: test2
    spec:
      serviceAccount: tidb-backup-manager
      br:
        cluster: demo2
        sendCredToTikv: false
        clusterNamespace: test2
        # logLevel: info
        # statusAddr: ${status_addr}
        # concurrency: 4
        # rateLimit: 0
        # timeAgo: ${time}
        # checksum: true
      # Only needed for TiDB Operator < v1.1.7 or TiDB < v4.0.8
      to:
        host: ${tidb_host}
        port: ${tidb_port}
        user: ${tidb_user}
        secretName: restore-demo2-tidb-secret
      s3:
        provider: aws
        region: us-west-1
        bucket: my-bucket
        prefix: my-folder
    ```

创建好 `Restore` CR 后，可通过以下命令查看恢复的状态：

{{< copyable "shell-regular" >}}

```shell
kubectl get rt -n test2 -o wide
```

更多 `Restore` CR 字段的详细解释：

* `.spec.metadata.namespace`：`Restore` CR 所在的 namespace。
* `.spec.to.host`：待恢复 TiDB 集群的访问地址。
* `.spec.to.port`：待恢复 TiDB 集群的访问端口。
* `.spec.to.user`：待恢复 TiDB 集群的访问用户。
* `.spec.to.tidbSecretName`：待恢复 TiDB 集群 `.spec.to.user` 用户的密码所对应的 secret。
* `.spec.to.tlsClientSecretName`：指定恢复备份使用的存储证书的 Secret。

    如果 TiDB 集群开启了 [TLS](enable-tls-between-components.md)，但是不想使用[文档](enable-tls-between-components.md)中创建的 `${cluster_name}-cluster-client-secret` 恢复备份，可以通过这个参数为恢复备份指定一个 Secret，可以通过如下命令生成：

    {{< copyable "shell-regular" >}}

    ```shell
    kubectl create secret generic ${secret_name} --namespace=${namespace} --from-file=tls.crt=${cert_path} --from-file=tls.key=${key_path} --from-file=ca.crt=${ca_path}
    ```

    > **注意：**
    >
    > 如果使用 TiDB Operator >= v1.1.7 && TiDB >= v4.0.8, BR 会自动调整 `tikv_gc_life_time` 参数，无需配置 `spec.to`.

* `.spec.tableFilter`：恢复时指定让 BR 恢复符合 [table-filter 规则](https://docs.pingcap.com/zh/tidb/stable/table-filter/) 的表。默认情况下该字段可以不用配置。当不配置时，BR 会恢复备份文件中的所有数据库：

    > **注意：**
    >
    > `tableFilter` 如果要写排除规则导出除 `db.table` 的所有表，`"!db.table"` 前必须先添加 `*.*` 规则来导出所有表，如下面例子所示：

    ```
    tableFilter:
    - "*.*"
    - "!db.table"
    ```

以上示例中，`.spec.br` 中的一些参数项均可省略，如 `logLevel`、`statusAddr`、`concurrency`、`rateLimit`、`checksum`、`timeAgo`、`sendCredToTikv`。

* `.spec.br.cluster`：代表需要备份的集群名字。
* `.spec.br.clusterNamespace`：代表需要备份的集群所在的 `namespace`。
* `.spec.br.logLevel`：代表日志的级别。默认为 `info`。
* `.spec.br.statusAddr`：为 BR 进程监听一个进程状态的 HTTP 端口，方便用户调试。如果不填，则默认不监听。
* `.spec.br.concurrency`：备份时每一个 TiKV 进程使用的线程数。备份时默认为 4，恢复时默认为 128。
* `.spec.br.rateLimit`：是否对流量进行限制。单位为 MB/s，例如设置为 `4` 代表限速 4 MB/s，默认不限速。
* `.spec.br.checksum`：是否在备份结束之后对文件进行验证。默认为 `true`。
* `.spec.br.timeAgo`：备份 timeAgo 以前的数据，默认为空（备份当前数据），[支持](https://golang.org/pkg/time/#ParseDuration) "1.5h", "2h45m" 等数据。
* `.spec.br.sendCredToTikv`：BR 进程是否将自己的 GCP 权限传输给 TiKV 进程。默认为 `true`。

## 故障诊断

在使用过程中如果遇到问题，可以参考[故障诊断](deploy-failures.md)。
