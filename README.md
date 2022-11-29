# awsecr-docker-tool
AWS-ECR login password tool 集成到[docker in docker] 镜像中

## [Docker in docker] 应用容器构建工具
```
registry.cn-hangzhou.aliyuncs.com/jinquan711/docker:22.06-aws-ecrloginpass
```

## 【使用案例】构建工具集成到jenkins流水线, 镜像构建完成后,自动推送镜像到AWS-ECR
```
////////////////////////////////////////////   函数定义：docker镜像构建、推送到AWS ECR仓库
// dockerfile: Dockerfile路径
// credential: AWS AKSK合成的Jenkins credential
// aws_region: AWS Region
// registry: 镜像仓库URL
// project: 镜像名称/项目名称
// branch: 项目分支
// app_routine_name: 容器内部应用进程名称
def docker_build(String dockerfile, String credential , String aws_region, String registry, String project, String branch, String app_routine_name) {
    withCredentials([usernamePassword(credentialsId: "${credential}", passwordVariable: 'SecurityKey', usernameVariable: 'AccessKey')]) {
        container('docker') {
            withEnv(["AwsRegion=${aws_region}", "Registry=${registry}", "Repository=${project}", "dockerFile=${dockerfile}", "Branch=${branch}", "AppRoutine=${app_routine_name}"]) {
                sh '''
                    ls -la /var/run/
                    export ACCESS_KEY=$AccessKey
                    export SECURITY_KEY=$SecurityKey
                    export AWS_REGION=$AwsRegion
                    echo $AWS_REGION
                    echo $ACCESS_KEY
                    ecrloginpass | docker login --username AWS --password-stdin $Registry 
                    cd src
                    ls -la .
                    export DOCKER_IMAGE=$Registry/$Repository:$(cat cicd/version)"-"${Branch}"-build"${BUILD_NUMBER}
                    docker build --build-arg APP_ROUTINE="${AppRoutine}" -t $DOCKER_IMAGE -f $dockerFile .
                    docker push $DOCKER_IMAGE
                '''
            }
        }
    }
}

/////////////////////////////////////////////   Jenkins pipeline 调用函数
    node(pod_name) {
        try {
            withEnv(['CiBuildProject=solar-system',
                     'CiAppRoutine=echo-server',
                     'GitRepository=git@gitee.com:jinquan711/solar-system.git',
                     'GitBranch=master',
                     'GitCheckoutKey=gitee-deploy-privkey']) {
                ...
                ...
                stage('build docker image & push to registry') {
                    docker_build("cicd/Dockerfile", "aws-ecr-ningxia-jinquan", 
                                "cn-northwest-1", "523497193792.dkr.ecr.cn-northwest-1.amazonaws.com.cn", 
                                "jinquan/solar-system", "${GitBranch}" , "${CiAppRoutine}" )
                }
                
            }
        }
        catch(all){
            currentBuild.result = 'FAILURE'
        }
        finally{
            stage('Send email'){
            }
        }
    }

```
![image](https://github.com/jinquan0/awsecr-docker-tool/blob/main/jenkinsfile%E9%9B%86%E6%88%90ECR-ecrlogin.PNG?raw=true)
![image](https://github.com/jinquan0/awsecr-docker-tool/blob/main/jenkinsfile%E9%9B%86%E6%88%90ECR-ecrpush.PNG?raw=true)


