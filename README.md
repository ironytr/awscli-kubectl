# Alpine Linux with AWS CLI v2 and kubectl
A minimal Docker image based on Alpine Linux with AWS CLI v2 and kubectl.

## Docker repository
https://hub.docker.com/r/gtsopour/awscli-kubectl

## Docker build/run
`docker build -t gtsopour/awscli-kubectl .`<br/>
`docker run -t -i -p 80:80 gtsopour/awscli-kubectl`



```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: ecr-token-helper
rules:
  - apiGroups: [""]
    resources:
      - secrets
      - serviceaccounts
      - serviceaccounts/token
    verbs:
      - 'delete'
      - 'create'
      - 'patch'
      - 'get'
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ecr-token-helper
  namespace: default
subjects:
  - kind: ServiceAccount
    name: sa-ecr-token-helper
    namespace: default
roleRef:
  kind: Role
  name: ecr-token-helper
  apiGroup: ""
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-ecr-token-helper
  namespace: default
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-token-helper
  namespace: default
spec:
  schedule: '* * * * *'
  successfulJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: sa-ecr-token-helper
          containers:
            - command:
                - /bin/sh
                - -c
                - |-
                  TOKEN=`aws ecr get-login-password --region ${REGION} | cut -d' ' -f6`
                  AWS_ACCOUNT_URL=${AWS_ACCESS_REGISTRY_ID}.dkr.ecr.${REGION}.amazonaws.com
                  echo $TOKEN
                  echo $AWS_ACCOUNT_URL
                  echo "hehehehe"
                  for REPOSITORY in ${ECR_REPOSITORIES//,/ }
                  do
                      # Add env var
                      REPOSITORY_VALUE=$(echo "$SECRET_NAME" | awk '{print tolower($0)}')-$(echo "$REPOSITORY" | awk '{print tolower($0)}')
                      REPO_URL=${AWS_ACCOUNT_URL}/$(echo "$REPOSITORY" | awk '{print tolower($0)}')
                      ECHO $REPO_URL "ASDASDASD"
                      echo "NE"+ $(echo "$REPOSITORY" | awk '{print tolower($0)}') + "hahahe"
                      kubectl delete secret -n default --ignore-not-found $REPOSITORY_VALUE
                      kubectl create secret -n default docker-registry $REPOSITORY_VALUE \
                      --docker-server=$REPO_URL \
                      --docker-username=AWS \
                      --docker-password=$TOKEN \
                      --namespace=default
                  done
                  kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"'$SECRET_NAME'"}]}' -n default

              env:
                - name: AWS_SECRET_ACCESS_KEY
                  value:""
                - name: AWS_ACCESS_KEY_ID
                  value: ""
                - name: AWS_ACCESS_REGISTRY_ID
                  value: ''
                - name: ACCOUNT
                  value: 'null'
                - name: SECRET_NAME
                  value: 'regcred'
                - name: REGION
                  value: 'us-east-1'
                - name: ECR_REPOSITORIES
                  value: "seperate,by,comma"
              image: biktimmk/awscli-kubectl-zsh:latest
              imagePullPolicy: IfNotPresent
              name: ecr-token-helper
          restartPolicy: Never
```