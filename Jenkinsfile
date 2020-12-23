/*
Сборка будет проходить если тега нет в списке релизов на сервере 

Новая версия приложения будет загружаться на сервер если:
- коммит по которому запустилась job помечен тэгом
- версия с тэгом не равна той на которую указывает симлинк на сервере

Роллбэк
ролбэк запускается через выбор параметра с версией тэга
если тэг уже есть на проде, то пропускаем билд и меняем симлинк
*/

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
        //Имя тэга у последнего протегированного коммита
        /*
        NEW_TAG_NAME = sh(
            returnStdout: true,
            script: "git show-ref --tags |\
                     grep \$(git rev-list --tags --date-order | head -1) |\
                     awk -F / '{print \$NF}'"
        ).trim()
        */
        NEW_TAG_NAME = sh(
            returnStdout: true,
            script: '''
                    LAST_TAG_HASH=$(git rev-parse --short $(git describe --tags --abbrev=0))
                    LAST_COMMIT_HASH=$(git rev-parse --short HEAD)

                    if [ "$LAST_TAG_HASH" != "$LAST_COMMIT_HASH" ]; then
                        git describe --tags --abbrev=0
                    else
                        echo ERROR: Last commit not tagged 1>&2
                        exit 1 # terminate and indicate error
                    fi 
            '''
        ).trim()

        // Проверка наличия дирректории с релизом на production сервере
        TAG_EXIST_ON_PROD = sh(
            returnStdout: true,
            script: "ssh ${PROD_USER}@${PROD_HOST} \"if [[ -d /var/www/myapp/releases/${NEW_TAG_NAME} ]]; then echo true; else echo false; fi\""
        ).trim()
    }

    stages{
        // Проверка что коммит по которому запущена job протегирован
        stage('Check before build') {
            agent { label 'master' }
            steps {
                echo "NEW_TAG_NAME: ${NEW_TAG_NAME}"
            }
        }

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
                     sh '''#!/bin/bash
                    LAST_TAG_HASH=$(git rev-parse --short $(git describe --tags --abbrev=0))
                    LAST_COMMIT_HASH=$(git rev-parse --short HEAD)

                    if [ "$LAST_TAG_HASH" != "$LAST_COMMIT_HASH" ]; then
                    CURRENT_VERSION_NUM=$(git describe --tags --abbrev=0 | cut -c 2-)
                    NEW_TAG_NAME=v$(echo | awk "{print $CURRENT_VERSION_NUM + 0.1}")

                    git tag $NEW_TAG_NAME
                    git push origin $NEW_TAG_NAME
                    else
                    echo 'В ветке master нет новых коммитов после последнего тэга'
                    fi 
                    '''
                    archiveArtifacts artifacts: 'build.zip', fingerprint: true
                }
            }
        }
        

        /*
        Деплой будет происходить только если:
            - новый коммит в мастер ветке был помечен тэгом
            - симлинк указывает на дирректорию имя которой отлично от последнего добавленного тэга
        */
        stage('Deploy') {
            environment {
                PROD_USER = 'deploy'
                PROD_HOST = '100.100.100.11'

                // Текущая версия релиза на production сервере 
                CURRENT_VERSION_ON_PROD = sh(
                returnStdout: true,
                script: "ssh ${PROD_USER}@${PROD_HOST} 'cd /var/www/myapp; ls -la | grep current | cut -d \'/\' -f6' "
                ).trim()
            }

            agent { label 'deploy' }
            when {
                 expression { CURRENT_VERSION_ON_PROD != NEW_TAG_NAME }
            }
            steps {
                script {
                    if (TAG_EXIST_ON_PROD == "true") {
                        sh '''ssh ${PROD_USER}@${PROD_HOST} "ln -sfn /var/www/myapp/releases/${TAG_NAME}/ /var/www/myapp/current"'''
                        echo "symlink changed to ${NEW_TAG_NAME}"
                    } else {
                        sh 'git clean -fdx'
                        copyArtifacts filter: 'build.zip', fingerprintArtifacts: true, projectName: '${JOB_NAME}', selector: specific('${BUILD_NUMBER}')
                        unzip zipFile: 'build.zip', dir: './build'
                        sh '''
                        echo "CURRENT_VERSION_ON_PROD: ${CURRENT_VERSION_ON_PROD}"
                        echo "TAG_NAME: ${TAG_NAME}"

                        rsync -avr ./build/ ${PROD_USER}@${PROD_HOST}:/var/www/myapp/releases/${NEW_TAG_NAME}/
                        
                        ssh ${PROD_USER}@${PROD_HOST} "ln -sfn /var/www/myapp/releases/${NEW_TAG_NAME}/ /var/www/myapp/current"
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