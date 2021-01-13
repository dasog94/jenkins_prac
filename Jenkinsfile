// 파이프라인 선언
pipeline {
    // 스테이지 별로 다른 거
    agent any // 노드 하나니까 아무거나 써라

    triggers {
        pollSCM('*/3 * * * *') // git을 3분 주기로 구동 Cron syntax
    }

    environment {
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId') // AWS 명령어 쓰기 위해
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey') // AWS 명령어 쓰기 위해
      AWS_DEFAULT_REGION = 'ap-northeast-2' // 서울
      HOME = '.' // Avoid npm root owned
    }

    stages {
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/dasog94/jenkins_prac.git', // git 주소
                    branch: 'master',
                    credentialsId: 'gitTokenforJenkins' // credentials에 등록된 git 토큰 id
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                    echo 'Successfully Cloned Repository'
                }

                always {
                  echo "i tried..."
                }

                cleanup {
                  echo "after all other post condition"
                }
            }
        }
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함. aws 명령어 사용하기 때문
            dir ('./website'){
                sh '''
                aws s3 sync ./ s3://jenkinsprac
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

                  mail  to: 'dasog94@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'dasog94@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              docker { // agent가 도커 쓴다
                image 'node:latest'
              }
            }
            
            steps {
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
            // 도커를 server라는 이름으로 빌드
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {
            failure { // 실패하면 이거 읽고 종료됨
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
//                 docker rm -f $(docker ps -aq) 아래에 이것이 추가되어 있으면 도커 돌고 있는 것 다 꺼버리는 것
                sh '''
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'dasog94@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
