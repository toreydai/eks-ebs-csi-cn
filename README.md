## AWS中国区域Amazon EKS使用EBS CSI驱动程序

### 1 为集群创建OIDC提供商
1.1 查看集群的OIDC提供商URL

```
aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
```
输出示例：

```
https://oidc.eks.cn-north-1.amazonaws.com.cn/id/EXAMPLED539D4633E53DE1B716D3041E
```
列出账户中的IAM OIDC提供商，使用上一条命令返回的值，替换EXAMPLED539D4633E53DE1B716D3041E

```
aws iam list-open-id-connect-providers | grep <EXAMPLED539D4633E53DE1B716D3041E>
```
输出示例：

```
"Arn": "arn:aws-cn:iam::111122223333:oidc-provider/oidc.eks.cn-north-1.amazonaws.com.cn/id/EXAMPLED539D4633E53DE1B716D3041E"
```
如果输出类似上述结果，则表明OIDC提供商已经创建，否则需要创建。

1.2 执行如下命令创建OIDC提供商，将<cluster_name>替换成实际的值

```
eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
```
### 2 部署Amazon EBS CSI驱动程序

2.1 执行如下命令，创建IAM策略文档

```
cat <<EoF > ./iam-examply-policy-cn.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateTags"
      ],
      "Resource": [
        "arn:aws-cn:ec2:*:*:volume/*",
        "arn:aws-cn:ec2:*:*:snapshot/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": [
            "CreateVolume",
            "CreateSnapshot"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteTags"
      ],
      "Resource": [
        "arn:aws-cn:ec2:*:*:volume/*",
        "arn:aws-cn:ec2:*:*:snapshot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeSnapshotName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    }
  ]
}
EoF
```
2.2 创建IAM策略

```
aws iam create-policy \
    --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
    --policy-document file://example-iam-policy-cn.json
```
记录返回的Policy ARN

```
arn:aws-cn:iam::111122223333:policy/AmazonEKS_EBS_CSI_Driver_Policy
```

2.3 创建IAM角色并附加此IAM策略，将my-cluster替换成当前集群名称，policy-arn替换成上一步骤中的Policy ARN

```
eksctl create iamserviceaccount \
                            --name ebs-csi-controller-sa \
                            --namespace kube-system \
                            --cluster my-cluster \
                            --attach-policy-arn policy-arn \
                            --approve \
                            --override-existing-serviceaccounts
```
检索创建角色的ARN并记下返回的值，以便在下一个步骤中使用，需要将my-cluster替换成当前集群名称

```
aws cloudformation describe-stacks \
    --stack-name eksctl-test-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa \
    --query='Stacks[].Outputs[?OutputKey==`Role1`].OutputValue' \
    --output text
```
输出示例

```
arn:aws-cn:iam::111122223333:role/eksctl-my-cluster-addon-iamserviceaccount-kube-sy-Role1-1J7XB63IN3L6T
```
2.4 部署AWS EBS CSI Driver

部署EBS CSI驱动程序
```
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git

cd aws-ebs-csi-driver/deploy/kubernetes/base/

kubectl apply -k aws-ebs-csi-driver/deploy/kubernetes/overlays/stable
```
修改image为国内可用版本
```
kubectl set image deployment ebs-csi-controller ebs-plugin=amazon/aws-ebs-csi-driver:v1.2.0 -n kube-system
kubectl set image daemonset ebs-csi-node ebs-plugin=amazon/aws-ebs-csi-driver:v1.2.0 -n kube-system
```
### 3 部署示例应用程序

3.1 部署示例应用

```
kubectl apply -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/
```
3.2 查看存储类

```
kubectl describe storageclass ebs-sc
```
输出参考：

```
Name:            ebs-sc
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           ebs.csi.aws.com
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```
3.3 持续查看Pod状态，等待Pod的状态变为Running

```
kubectl get pods --watch
```
3.4 查看名称为ebs-claim的pv

```
kubectl get pv | grep ebs-claim
```
输出参考

```
pvc-108dd7ee-149f-40b5-b562-3c86b892b9cc   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  5h3m
```
3.5 验证数据成功写入EBS卷

```
kubectl exec -it app -- cat /data/out.txt
```
输出参考

```
Thu Aug 12 12:18:33 UTC 2021
Thu Aug 12 12:18:38 UTC 2021
Thu Aug 12 12:18:43 UTC 2021
Thu Aug 12 12:18:48 UTC 2021
Thu Aug 12 12:18:53 UTC 2021
Thu Aug 12 12:18:58 UTC 2021
...
```
3.5 验证成功，删除示例应用程序

```
kubectl delete -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/
```
