pipeline {
    agent { label 'master' }
    /*
    parameters {
        gitParameter branchFilter: 'origin/(.*)', defaultValue: 'master', name: 'BRANCH', type: 'PT_BRANCH'
    }

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
    */

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

    stages{
        stage('Build') {
            agent { label 'slave' }
            steps {
               // sh 'bash -l -c ". $HOME/.nvm/nvm.sh ; nvm use v10.12.0 || nvm install v10.12.0 && nvm use v10.12.0"' 
                nvm('v10.12.0') {
                    sh '''
                    node -v
                    /home/jenkins/.nvm/versions/node/v10.12.0/bin/npm install
                    /home/jenkins/.nvm/versions/node/v10.12.0/bin/npm run build
                    zip zipFile: 'build.zip', archive: false, dir: 'build'
                    '''
                }
            }
            //    archiveArtifacts artifacts: 'test_file.txt', 'build'
            post {
                success {
                    archiveArtifacts artifacts: 'build.zip', fingerprint: true
                }
            }
        }

        
        stage('Deploy') {
            agent { label 'master' }
            /*
             Деплой будет происходить только если:
                - новый коммит в мастер ветке был помечен тэгом
                - симлинк указывает на дирректорию имя которой отлично от последнего добавленного тэга
            */
            when {
                 expression { CURRENT_VERSION_ON_PROD != NEW_TAG_NAME }
            }
            steps {
                copyArtifacts filter: 'build.zip', fingerprintArtifacts: true, projectName: '${JOB_NAME}', selector: specific('${BUILD_NUMBER}')
                unzip zipFile: 'build.zip', dir: './build'
                sh '''
                git clean -fdx
                ls -la
                echo "CURRENT_VERSION_ON_PROD: ${CURRENT_VERSION_ON_PROD}"
                echo "NEW_TAG_NAME: ${NEW_TAG_NAME}"

                rsync -avr ./build/ ${PROD_USER}@${PROD_HOST}:/var/www/myapp/releases/${NEW_TAG_NAME}/
                
                ssh ${PROD_USER}@${PROD_HOST} "ln -sfn /var/www/myapp/releases/${NEW_TAG_NAME}/ /var/www/myapp/current"
                
                ssh ${PROD_USER}@${PROD_HOST} "cd /var/www/myapp/releases && rm -rf \$(ls -t | awk 'NR>5')"
                '''
            }
        }     
    }
}