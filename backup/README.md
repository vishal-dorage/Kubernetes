### Medidata Sensor Cloud (sensorcloud) EKS Velero Backup Setup with kube2iam

#### prerequisite:

Velero CLI Client should be installed


#### Step:-

##### 1. It is necessary to create an IAM role which can assume other roles and assign it to each kubernetes worker.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sts:AssumeRole"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```
##### 2. Create k8s-velero IAM role With Trust Relationship with AWS EKS Node IAM Role

```
cat > node-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/kubernetes-worker-role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

```
aws iam create-role \
  --role-name k8s-velero \
  --assume-role-policy-document \
  file://node-trust-policy.json
```

##### 3. Create S3 IAM Policy and attach to the k8s-velero IAM role

```
cat > s3-velero-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": [
        "arn:aws:s3:::${BUCKET}/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::${BUCKET}"
      ]
    }
  ]
}
EOF
```

```
aws iam put-role-policy \
  --role-name k8s-velero \
  --policy-name s3 \
  --policy-document file://s3-velero-policy.json
```

##### 4. Setup kube2iam on AWS EKS cluster with following configuration

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:  
  name: kube2iam
rules:
  - apiGroups:
    - ""
    resources:
    - namespaces
    - pods
    verbs:
    - get
    - watch
    - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube2iam
subjects:
  - kind: ServiceAccount
    name: kube2iam
    namespace: default
roleRef:
  kind: ClusterRole
  name: kube2iam
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube2iam
  namespace: default
---
apiVersion: apps/v1
kind: DaemonSet
metadata:  
  name: kube2iam  
  namespace: default  
  labels:    
    app: kube2iam
spec:  
  selector:    
    matchLabels:      
      name: kube2iam  
  updateStrategy:    
    type: RollingUpdate  
  template:    
    metadata:      
      labels:        
        name: kube2iam    
    spec:      
      serviceAccountName: kube2iam      
      hostNetwork: true      
      containers:        
        - image: jtblin/kube2iam:0.10.7          
          imagePullPolicy: Always          
          name: kube2iam          
          args:            
            - "--auto-discover-base-arn"            
            - "--auto-discover-default-role=true"            
            - "--iptables=true"            
            - "--host-ip=$(HOST_IP)"            
            - "--node=$(NODE_NAME)"            
            - "--host-interface=eni+"          
          env:            
            - name: HOST_IP              
              valueFrom:                
                fieldRef:                  
                  fieldPath: status.podIP            
            - name: NODE_NAME              
              valueFrom:                
                fieldRef:                  
                  fieldPath: spec.nodeName          
          ports:            
            - containerPort: 8181              
              hostPort: 8181              
              name: http          
          securityContext:            
            privileged: true
```

The above configuration defines essential kube2iam resources; ServiceAccount, ClusterRole, ClusterRoleBinding and DaemonSet. All the templates run in a default namespace.

##### 4. Install and start Velero

```
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.3.0 \
    --bucket $BUCKET \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --pod-annotations iam.amazonaws.com/role=arn:aws:iam::076395046979:role/k8s-velero \
    --no-secret
```

##### To check the AWS S3 has been setup as a storage location sucessfully 

```
velero get backup-location
```
The output should be like as follow
```
NAME      PROVIDER   BUCKET/PREFIX       PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   aws        S3_Bucketname   Available   2021-10-25 14:54:48 +0530 IST   ReadWrite     true
```

##### We are done.!!

#### For Schedule the backup

You can run backups on a schedule, for example, to create a daily backup with expiration set to 90 days:

```
velero schedule create example.com --schedule="0 0 * * *" --include-namespaces default --ttl 2160h0m0s
```
Here:
  example.com is the backup name


#### For see schedule backups

```
velero get schedule
```
Output is someting like 
```
NAME          STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP   SELECTOR
example.com   Enabled   2021-10-24 20:15:51 +0530 IST   0 * * * *   2h0m0s       7m ago        <none>
```

#### For delete schedule backup

```
velero delete schedule example.com
```
