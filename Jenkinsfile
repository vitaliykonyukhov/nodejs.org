pipeline {
    agent { label 'master' }
    /*
    parameters {
        gitParameter branchFilter: 'origin/(.*)', defaultValue: 'master', name: 'BRANCH', type: 'PT_BRANCH'
    }
    */
    /*
    environment {
        PROD_USER = 'deploy'
        PROD_HOST = '100.100.100.11'
        
        //Имя тэга у последнего протегированного коммита
        NEW_TAG_NAME = sh(
            returnStdout: true,
            script: "git show-ref --tags |\
                     grep \$(git rev-list --tags --date-order | head -1) |\
                     awk -F / '{print \$NF}'"
            ).trim()
        
        //Текущая версия релиза на production сервере 
        CURRENT_VERSION_ON_PROD = sh(
            returnStdout: true,
            script: "ssh ${PROD_USER}@${PROD_HOST} 'cd /var/www/myapp; ls -la | grep current | cut -d \'/\' -f6' "
            ).trim()
    }
    */
    
    stages {
        stage('check current environment') {
            agent { label 'master' }
            steps {
                sh '''
                hostname
                pwd
                id
                ssh deploy@100.100.100.11 hostname
                '''
            }
        }

        stage('Build') {
            agent { label 'slave' }
            steps {
                sh '''
                echo Build stage
                hostname
                '''
            }
        }

        /*
        stage('Deploy') {
            agent { label 'master' }
             Деплой будет происходить только если:
                - новый коммит в мастер ветке был помечен тэгом
                - симлинк указывает на дирректорию имя которой отлично от последнего добавленного тэга
             
            when {
                 expression { CURRENT_VERSION_ON_PROD != NEW_TAG_NAME }
            }
            steps {
                sh '''
                echo "CURRENT_VERSION_ON_PROD: ${CURRENT_VERSION_ON_PROD}"
                echo "NEW_TAG_NAME: ${NEW_TAG_NAME}"

                rsync -avr ./index.html ${PROD_USER}@${PROD_HOST}:/var/www/myapp/releases/${NEW_TAG_NAME}/
                
                ssh ${PROD_USER}@${PROD_HOST} "ln -sfn /var/www/myapp/releases/${NEW_TAG_NAME}/ /var/www/myapp/current"
                
                ssh ${PROD_USER}@${PROD_HOST} "cd /var/www/myapp/releases && rm -rf \$(ls -t | awk 'NR>5')"
                '''
            }
        }
        */       
    }
}