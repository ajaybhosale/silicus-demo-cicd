pipeline {
  agent any
  stages {
    stage('Static Code Analysis') {
      parallel {
        stage('PHP CPD') {
          steps {
            sh 'phpcpd --exclude vendor workspace'
          }
        }
        stage('Calculating LOC') {
          steps {
            sh 'phploc workspace'
          }
        }
        stage('PHP Code Sniffer') {
          steps {
            sh 'phpcs --warning-severity=6 --standard=PSR2 --extensions=php --ignore=*/migrations/* workspace/modules'
          }
        }
        stage('PHP Depend') {
          steps {
            sh 'pdepend --ignore=vendor --summary-xml=/tmp/summary.xml --jdepend-chart=/tmp/jdepend.svg --overview-pyramid=/tmp/pyramid.svg workspace'
          }
        }
        stage('PHP MD') {
          steps {
            sh 'phpmd workspace/public/index.php text static_analysis/md_rulesets/cleancode.xml,static_analysis/md_rulesets/naming.xml,static_analysis/md_rulesets/codesize.xml'
          }
        }
        stage('PHP Lint Test') {
          steps {
            sh 'php -l workspace'
          }
        }
      }
    }
    stage('PHP Unit Test Cases...') {
      steps {
        sh '''cp dit-app.conf workspace/.env
cd workspace
chmod -R 777 storage/
chmod -R 777 bootstrap/
composer update
composer dumpautoload
php artisan migrate
php artisan cache:clear
php artisan config:clear
php artisan config:cache
vendor/bin/phpunit --log-junit build/logs/junit.xml'''
      }
    }
    stage('SonarQube Analysis') {
      environment {
        PROJECT_NAME = 'Silicus-PHP-Demo-CICD'
        PROJECT_KEY = 'silicus-php-demo-cicd'
        SONAR_HOST_URL = 'https://codeanalysis.silicus.com/'
        SONAR_LOGIN = 'ee0efa18054280e8a8f4399bba3797086991119d'
        PROJECT_SOURCE_ENCODING = 'UTF-8'
        PROJECT_LANGUAGE = 'php'
      }
      steps {
        sh 'chmod -R 777 workspace/build'
        sh '''PROJECT_VERSION=1.0.$(date +%y)$(date +%j).$BUILD_NUMBER
/opt/sonar/bin/sonar-runner -Dsonar.projectName=$PROJECT_NAME \\
-Dsonar.projectKey=$PROJECT_KEY \\
-Dsonar.login=$SONAR_LOGIN \\
-Dsonar.host.url=$SONAR_HOST_URL \\
-Dsonar.sourceEncoding=$PROJECT_SOURCE_ENCODING \\
-Dsonar.sources=$WORKSPACE \\
-Dsonar.language=$PROJECT_LANGUAGE \\
-Dsonar.projectVersion=$PROJECT_VERSION \\
-Dsonar.php.tests.reportPath=$WORKSPACE/build/logs/junit.xml \\
-Dsonar.php.coverage.reportPaths=$WORKSPACE/build/logs/clover.xml \\
-Dorg.sonar.plugins.jmeter.jtlpath==$WORKSPACE/build/jmeter.jtl \\
-Dsonar.exclusions="workspace/app/**, workspace/bootstrap/**, workspace/resources/**,workspace/config/**, workspace/database/**, workspace/modules/infrastructure/**,workspace/modules/user/**, workspace/public/**, workspace/routes/**, workspace/storage/**, workspace/tests/**, workspace/vendor/**"'''
      }
    }
    stage('Archive Artifacts') {
      steps {
        archiveArtifacts(artifacts: 'workspace/**', excludes: 'selenium', allowEmptyArchive: true)
        archiveArtifacts(artifacts: 'composer.json,composer.lock,dit-app.conf,Dockerfile', allowEmptyArchive: true)
      }
    }
    stage('Deploy to Development') {
      environment {
        AZURE_CR_USERNAME = 'silicus'
        AZURE_CR_PASSWORD = '5zNvbJC7tidlPx/erzMysNuPwx5IRREF'
        AZURE_CR_Login = 'silicus.azurecr.io'
        AZURE_CR_IMAGE = 'silicus-php-demo-dit'
      }
      steps {
        sh '''cd /var/lib/jenkins/jobs/silicus-demo-cicd/branches/development/builds/${BUILD_NUMBER}/archive
cp dit-app.conf workspace/.env
docker login --username silicus --password 5zNvbJC7tidlPx/erzMysNuPwx5IRREF silicus.azurecr.io
docker build -t silicus-php-demo-dit .
docker tag silicus-php-demo-dit silicus.azurecr.io/silicus-php-demo-dit:latest
docker tag silicus-php-demo-dit silicus.azurecr.io/silicus-php-demo-dit:1
docker push silicus.azurecr.io/silicus-php-demo-dit:latest
docker push silicus.azurecr.io/silicus-php-demo-dit:1'''
        mail(subject: 'SilicusDemo Approval for Staging', body: "Hi, Please take a action on new build  <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>", to: 'ajay.bhosale@silicus.com', replyTo: 'testmili@gmail.com', mimeType: 'text/html', from: 'testmili@gmail.com')
        sh '''pwd
ls -all'''
      }
    }
    stage('Deploy to QA') {
      steps {
        input(message: 'Deploy to QA?', id: 'deploy-to-qa', ok: 'Proceed', submitter: 'ajay', submitterParameter: 'yes')
        sh '''pwd
ls -all'''
        sh '''cd /var/lib/jenkins/jobs/silicus-demo-cicd/branches/development/builds/${BUILD_NUMBER}/archive
cp dit-app.conf workspace/.env
docker login --username silicus --password 5zNvbJC7tidlPx/erzMysNuPwx5IRREF silicus.azurecr.io
docker build -t silicus-php-demo-sit .
docker tag silicus-php-demo-dit silicus.azurecr.io/silicus-php-demo-sit:latest
docker tag silicus-php-demo-dit silicus.azurecr.io/silicus-php-demo-sit:1
docker push silicus.azurecr.io/silicus-php-demo-sit:latest
docker push silicus.azurecr.io/silicus-php-demo-sit:1'''
        mail(subject: 'SilicusDemo Approval for UAT ', body: "Hi, Please take a action on new build  <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>", to: 'ajay.bhosale@silicus.com', replyTo: 'testmili@gmail.com', mimeType: 'text/html', from: 'testmili@gmail.com')
      }
    }
    stage('Selenium Test Cases...') {
      steps {
        sh 'chmod -R 777 workspace/selenium/'
        sh 'java -cp workspace/selenium/Restapi1/bin:workspace/selenium/Restapi1/lib/* org.testng.TestNG workspace/selenium/Restapi1/testng.xml'
      }
    }
    stage('Jmeter Test Cases...') {
      steps {
        sh '/opt/apache-jmeter-5.0/bin/jmeter -Jjmeter.save.saveservice.output_format=xml -n -t JavaDevOps.jmx -l workspace/build/jmeter.jtl'
        perfReport 'workspace/build/jmeter.jtl'
      }
    }
    stage('Deploy to Staging/UAT') {
      steps {
        input(id: 'deploy-to-uat', message: 'Deploy to UAT', ok: 'Proceed', submitter: 'ajay', submitterParameter: 'yes')
        sh '''cd /var/lib/jenkins/jobs/silicus-demo-cicd/branches/development/builds/${BUILD_NUMBER}/archive
cp dit-app.conf workspace/.env
docker login --username silicus --password 5zNvbJC7tidlPx/erzMysNuPwx5IRREF silicus.azurecr.io
docker build -t silicus-php-demo-uat .
docker tag silicus-php-demo-dit silicus.azurecr.io/silicus-php-demo-uat:latest
docker tag silicus-php-demo-dit silicus.azurecr.io/silicus-php-demo-uat:1
docker push silicus.azurecr.io/silicus-php-demo-uat:latest
docker push silicus.azurecr.io/silicus-php-demo-uat:1'''
      }
    }
    stage('Delete Workspace') {
      parallel {
        stage('Delete Workspace') {
          steps {
            cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, cleanWhenSuccess: true, cleanupMatrixParent: true, deleteDirs: true)
          }
        }
      }
    }
  }
  post {
    success {
      mail(to: 'ajay.bhosale@silicus.com', subject: "Success Pipeline: ${currentBuild.fullDisplayName}", body: "Congratulations pipeline build successfully ${env.BUILD_URL}")

    }

    failure {
      mail(to: 'ajay.bhosale@silicus.com', subject: "Failed Pipeline: ${currentBuild.fullDisplayName}", body: "Something is wrong with ${env.BUILD_URL}")

    }

  }
}