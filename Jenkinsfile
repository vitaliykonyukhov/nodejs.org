pipeline {
    agent { label 'master' }

    /*
    parameters {
        gitParameter name: 'TAG', 
                     type: 'PT_TAG',
                     defaultValue: 'master'
    }
    parameters {
        gitParameter branchFilter: 'origin/(.*)', defaultValue: 'master', name: 'BRANCH', type: 'PT_BRANCH'
    }
    */

    environment {
        PROD_USER = 'deploy'
        PROD_HOST = '100.100.100.11'
        
        /*
        //Имя тэга у последнего протегированного коммита
        NEW_TAG_NAME = sh(
            returnStdout: true,
            script: "git show-ref --tags |\
                     grep \$(git rev-list --tags --date-order | head -1) |\
                     awk -F / '{print \$NF}'"
        ).trim()
        */

        // Проверка наличия дирректории с релизом на production сервере
        TAG_EXIST_ON_PROD = sh(
            returnStdout: true,
            script: "ssh ${PROD_USER}@${PROD_HOST} \"if [[ -d /var/www/myapp/releases/${TAG_NAME} ]]; then echo true; else echo false; fi\""
        ).trim()

        // Текущая версия релиза на production сервере 
        CURRENT_VERSION_ON_PROD = sh(
            returnStdout: true,
            script: "ssh ${PROD_USER}@${PROD_HOST} 'cd /var/www/myapp; ls -la | grep current | cut -d \'/\' -f6' "
            ).trim()
    }

    stages{
        stage('Build') {
            when {
                expression { TAG_EXIST_ON_PROD == "false" }
            }
            agent { label 'nodejs' }
            steps {
                    sh '''
                    git clean -fdx
                    set +ex
                    export NVM_DIR="$HOME/.nvm"
                    . ~/.nvm/nvm.sh
                    set -ex
                    nvm use v10.12.0
                    npm install
                    npm run build
                    '''
                    zip zipFile: 'build.zip', archive: true, dir: 'build'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'build.zip', fingerprint: true
                }
            }
        }
        
        stage('Deploy') {
            agent { label 'deploy' }
            /*
             Деплой будет происходить только если:
                - новый коммит в мастер ветке был помечен тэгом
                - симлинк указывает на дирректорию имя которой отлично от последнего добавленного тэга
            */
            when {
                 expression { CURRENT_VERSION_ON_PROD != TAG_NAME }
            }
            steps {
                script {
                    if (TAG_EXIST_ON_PROD == "true") {
                        ssh ${PROD_USER}"@"${PROD_HOST} "ln -sfn /var/www/myapp/releases/${TAG_NAME}/ /var/www/myapp/current"
                        echo "symlink changed to ${TAG_NAME}"
                    } else {
                        sh 'git clean -fdx'
                        copyArtifacts filter: 'build.zip', fingerprintArtifacts: true, projectName: '${JOB_NAME}', selector: specific('${BUILD_NUMBER}')
                        unzip zipFile: 'build.zip', dir: './build'
                        sh '''
                        echo "CURRENT_VERSION_ON_PROD: ${CURRENT_VERSION_ON_PROD}"
                        echo "TAG_NAME: ${TAG_NAME}"

                        rsync -avr ./build/ ${PROD_USER}@${PROD_HOST}:/var/www/myapp/releases/${TAG_NAME}/
                        
                        ssh ${PROD_USER}@${PROD_HOST} "ln -sfn /var/www/myapp/releases/${TAG_NAME}/ /var/www/myapp/current"
                        '''
                        sh '''
                        ssh ${PROD_USER}@${PROD_HOST} 'cd /var/www/myapp/releases && rm -rf $(ls -t | awk "NR>5")'
                        ''' 
                    }
                }
            }
        }     
    }
}