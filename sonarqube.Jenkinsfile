pipeline {
    options {
        buildDiscarder(logRotator(
            artifactDaysToKeepStr: '14',
                artifactNumToKeepStr: '14',
            daysToKeepStr: '14',
            numToKeepStr: '4'))
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }
 
 
    agent { node { label 'dev && linux && slave && test' } }
 
    triggers {
        GenericTrigger(
            genericRequestVariables: [
            ],
            genericVariables: [
                [key: 'Email', value: '$.actor.emailAddress'],
                [key: 'Repo', value: '$.repository.slug'],
                [key: 'ownerName', value: '$.pullrequest.fromRef.repository.project.key'],
                [key: 'Branch', value: '$.pullrequest.fromRef.branch.name'],
                [key: 'PullRequest', value: '$.pullrequest.id'],
            ],
 
            causeString: 'This job was triggered by a webhook from Bitbucket',
            token: 'terte1lterl',
            regexpFilterExpression: '',
            regexpFilterText: '',
            printContributedVariables: true,
            printPostContent: true
        )
    }
 
    environment {
            BITBUCKET_BASE_URL = "BITBUCKET"
            URL_SONAR = "SONARQUBE"
            SONAR_SCANNER = "sonarscanner:rb-4.6.2.2472"
            DOCKER_REGISTRY = "DOCKER_REGISTRY"
            DEFAULT_CICD = "./source/ci_cd.properties"
            CICD_BRANCH = "SMALL-7438"
            EXCLUSIONS = "*.png,*.img,*.gif,*.jpg,*.crt,*.cer,*.pem,*.key,*.class"
            DEFAULT_JAVAVERSION = "1.8"
            DEFAULT_GRADLE = "./source/build.gradle"
            NOT_FIND_MODULES = " -type d -not -name source -not -name gradle -not -name .git -not -name ci_cd -not -name environment -not -name .settings -not -name .idea -not -name ."
    }
 
    stages {
        stage('Checkout cicd') {
            steps {
                sh """
                echo ---Load from json----
                echo Echoing variables:  $Email
                echo Echoing variables:  $Branch
                echo Echoing variables:  $Repo
                echo Echoing variables:  $ownerName
                echo Echoing variables:  $PullRequest
                """               
                
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${CICD_BRANCH}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [
                        [$class: 'WipeWorkspace']
                    ],
                    submoduleCfg: [],
                    userRemoteConfigs: [[
                        credentialsId: 'cicd_abpm_developers',
                        url: "${BITBUCKET_BASE_URL}/scm/${ownerName}/cicd.git"]]
                ])
                echo "---repo:---"
                echo "repo: ${Repo}"
                echo "---branch:---"
                echo "branch: ${Branch}"
                sh "rm -Rf ${env.workspace}/*"
                sh "docker container prune"
                sh "docker image prune"
                sh "mkdir source"
            }
        }
        stage('Checkout source to check'){
            steps{
                dir ("source"){
                    git(
                        url: "${BITBUCKET_BASE_URL}/scm/${ownerName}/${Repo}.git",
                        credentialsId: "cicd_abpm_developers",
                        branch: "${Branch}")
                }
           }
        }
 
        stage('Build and run docker'){
            steps {
                script {
                    def myArray=[]
                    def myArray2=[]
                    multi_modules = false
                    single_module = false
                    modules_in_dir = false
 
                    // ищем файлы с настройками ci-cd
                    FILES = sh(returnStdout: true, script: 'find ./source -name "ci_cd.properties"').trim()
                    echo "---FILES---"
                    print FILES
                    // если не находим то создаем дефолтный файл с настройками
                    if (FILES.size() == 0) {
                        def FILES = ''
                        sh"""
                            pwd
                            echo \$SHELL
                            touch ${DEFAULT_CICD}
                            echo "APP_NAME=${env.Repo}" >> ${DEFAULT_CICD}
                            echo "SONAR_MODULES_SCAN=." >> ${DEFAULT_CICD}
                            cat ${DEFAULT_CICD}
                        """
                        myArray << DEFAULT_CICD
                        echo "default cicd file is created"
                        print(DEFAULT_CICD)
                    }
                    // если нашлось несколько файлов с настройками
                    if (FILES.size() > 0) {
                        FILES.split().each {
                            myArray << it
                        }
                    }
                    echo "---found ci_cd.propreties:---"
                    print myArray
                    print myArray.size()
                    if (myArray.size() == 1) {
                        // найден один файл с настройками cicd
                        multi_modules = true
                        ci_cd_patch = myArray[0]
                        sonar_modules_scan = takeParam("SONAR_MODULES_SCAN", "${myArray[0]}")
                        echo "---single module is found---"
                        print(sonar_modules_scan)
                        // найдена точка в списке модулей для сканирования, это значит или один модуль или модули разбросаны по каталогам
                        if (sonar_modules_scan == ".") {
                            single_module = true
                            echo "---real single module is found---"
                        }
                        // найдено несколько названий модулей в строке SONAR_MODULES_SCAN
                        else {
                            myArray=[]
                            sonar_modules_scan.split(',').each {
                                echo "---few modules in sonar_modules_scan is found---"
                                myArray << './source/' + it +'/'
                               print(myArray)
                            }
                        }
                     // если найдено несколько файлов с настройками, то добавляем путь в массив с названиями модулей
                    if (myArray.size() > 1) {
                        myArray=[]
                        echo "---myArray>1: found sonar_modules: ${sonar_modules_scan}"
                        sonar_modules_scan.split(',').each {
                            myArray << './source/' + it +'/'
                        }
                        echo "---myArray >1 print myArray---"
                        print(myArray)
                    }
                    // Проверяем наличие разбросанных по каталогам модудей
                    if (single_module == true) {
                        myArray = []
                        MODULES = sh (returnStdout: true, script: 'find ./source -maxdepth 1 \${NOT_FIND_MODULES}').trim()
                        echo "---MODULES---"
                        print(MODULES)
                        // найдены разбросанные по каталогам модули
                        if (MODULES.size() > 0) {
                            MODULES.split().each {
                                echo "---it---"
                                print(it)
                                myArray << it
                            }
                            echo"---myArray2---"
                            print(myArray)
                            modules_in_dir = true
                        }
                    }
 
                    // Устанавливаем версию Java по умолчанию
                    JAVAVERSION = DEFAULT_JAVAVERSION
                    // Ищем pom-файлы
                    POM = sh(returnStdout: true, script: 'find ./source -name "pom.xml"').trim()
                    echo "---POM---"
                    print(POM)
                    if (POM.size() > 0) {
                        JAVAVERSION = sh(
                            script: """xmllint --xpath "//*[local-name()='project']/*[local-name()='properties']/*[local-name()='java.version']/text()" ./source/pom.xml""",
                            returnStdout: true
                        ).trim()
                        echo JAVAVERSION   
                        }
                    if (POM.size() == 0) {
                        // нет pom, ищем build.gradle
                        echo "---no pom.xml found, try to find build.gradle---"
                        GRADLE = sh(returnStdout: true, script: 'find ./source/ -name "build.gradle"').trim()
                        echo "---GRADLE:---"
                        echo GRADLE
                        // не нашли build.gradle, устанавливаем версию java по умолчанию
                        if (GRADLE.size() == 0) {
                            echo "No Gradle"
                            JAVAVERSION = DEFAULT_JAVAVERSION }
                        // если нашли build.gradle
                        if (GRADLE.size() > 0) {
                            //проверяем build.gradle в пути по умолчанию
                            if (fileExists(DEFAULT_GRADLE)) {
                                echo 'build.gradle in source exists'
                                def Result = ''
                                // проверяем содержит ли build.gradle в пути по умолчанию версию Java
                                echo "--check build_gradle and output from function 1---"
                                Result = (checkJavaVersionInBuildGradle("sourceCompatibility","./source/build.gradle"))
                                echo "--start check JavaVersion in build.gradle---"
                                // если не содержит, то устанавливаем версию по умолчанию
                                if (Result == '0') {
                                        JAVAVERSION = DEFAULT_JAVAVERSION
                                        echo "no java version in ./source/${sonar_modules_scan}/build.gradle"
                                }
                                // если содержит, то извлекаем версию Java из найденного файла из пути по умолчанию
                                if (Result != '0') {
                                        echo "---start search in build.gradle---"
                                        JAVAVERSION = getJavaFromBuildGrade("sourceCompatibility", "./source/build.gradle")
                                        echo "there is java version in ./source/${sonar_modules_scan}/build.gradle"
                                }
                            }
                        }
                    }
                    echo "-----binary vars-----"
                    echo "modules in dir"
                    print(modules_in_dir)
                    echo "multi modules"
                    print(multi_modules)
                    echo "single module"
                    print(single_module)
                    // перебираем модули в цикле
                    for (x in myArray) {
                        echo "----x:----"
                        print(x)
                        // if (myArray.size() == 1) {
                        //}
                        if (multi_modules == true) {
                            x2 = ci_cd_patch
                            echo "--- ci_cd properties: ${x2} ---"
                            app_name = takeParam("APP_NAME", "${x2}")
                            if (single_module == false) {
                                sonar_modules_scan = x[9..(x.length()-2)]
                                echo "try to cut string: ${sonar_modules_scan}"
                            }
                            if (modules_in_dir == true && multi_modules == true && single_module == true) {
                                sonar_modules_scan = x[9..(x.length()-1)]
                                echo "modules in dir found"
                                echo "try to cut string: ${sonar_modules_scan}"
                            }
                        }
                        if (multi_modules == false && modules_in_dir == false) {
                            app_name = takeParam("APP_NAME", "${x}")
                            sonar_modules_scan = takeParam("SONAR_MODULES_SCAN", "${x}")
                        }
                        echo "---found :---"
                        echo "sonar_modules_scan: ${sonar_modules_scan}"
                        echo "url_sonar: ${URL_SONAR}"
                        echo "sonar_scanner: ${SONAR_SCANNER}"
                        echo "docker_registry: ${DOCKER_REGISTRY}"
                        echo "app_name: ${app_name}"
                        echo "myArray size:"
                        print myArray.size()
                        // проверяем build.gradle в модуле
                        if (fileExists("./source/${sonar_modules_scan}/build.gradle")) {
                            def Result = ''
                            echo "--check build_gradle and output from function 2---"
                            Result = (checkJavaVersionInBuildGradle("sourceCompatibility","./source/${sonar_modules_scan}/build.gradle"))
                            echo "--Result:---"
                            print(Result)
                            print(Result.getClass())
                            echo "--start check JavaVersion in build.gradle---"
                            if (Result == '0') {
                                    JAVAVERSION = DEFAULT_JAVAVERSION
                                    echo "no java version in ./source/${sonar_modules_scan}/build.gradle"
                            }
                            if (Result != '0') {
                                    echo "---start search in build.gradle---"
                                    JAVAVERSION = getJavaFromBuildGrade("sourceCompatibility", "./source/${sonar_modules_scan}/build.gradle")
                                    echo "there is java version in ./source/${sonar_modules_scan}/build.gradle"
                            }
                        }
 
                        sh "echo jdk version = '${JAVAVERSION}'"
                        // ищем  все java-файлы и копируем их в ./src2
                        dir("./source/${sonar_modules_scan}"){
                            sh script: $/
                                df -h
                                pwd
                                ls -la
                                mkdir src2
                                find . -type f -iname "*.java" -not -name ".git" -not -path "./src2/*" -exec cp --parents {} ./src2 \;
                                find . -type f -iname "*.java" -not -name ".git" -not -path "./src2/*" ;
                                cd ./src2
                                ls -la
                                /$
                        }
                        sh "mkdir src"
                        sh "mv ./source/${sonar_modules_scan}/src2/* ./src"
                        withCredentials([usernamePassword(credentialsId: 'cicd_abpm_developers', passwordVariable: 'NEXUS_PASS', usernameVariable: 'NEXUS_LOGIN')]){
                            sh "docker login docker-registry.gts.rus.socgen -u '${NEXUS_LOGIN}' -p '${NEXUS_PASS}'"}
                        script {
                                echo "----myarray > 1----"
                                if (single_module == true && modules_in_dir == false) {
                                    string = "${env.Repo}::${PullRequest}"   
                                }
                                if (single_module == true && modules_in_dir == true) {
                                    string = "${env.Repo}::${sonar_modules_scan}::${PullRequest}"
                                }
                                if (single_module == false) {
                                    string = "${env.Repo}::${sonar_modules_scan}::${PullRequest}"
                                }
                                echo "${string}"
                                // собираем файл с настройками для сонар сканера
                                def data =
                                    "sonar.projectKey=SMALL::${string}\n" +
                                    "sonar.projectName=SMALL::${string}\n" +
                                    "sonar.host.url=${url_sonar}/\n"+
                                    "sonar.sources=/tmp/src/\n"+
                                    "sonar.java.source=${env.JavaVersion}\n"+
                                    "sonar.exclusions=${env.Exclusions}\n"+
                                    "sonar.java.binaries=/."
                                writeFile(file: 'sonar.properties', text: data)
                                // собираем Dockerfile c найденными для анализа файлами
                                def data2 = "FROM $docker_registry/general/$sonar_scanner\n" +
                                    "WORKDIR /tmp\n" +
                                    "CMD ['mkdir /tmp/src']\n" +
                                    "COPY src /tmp/src\n" +
                                    "RUN df -h \n"+
                                    "COPY sonar.properties /tmp/sonar.properties"
                                writeFile(file: 'Dockerfile', text: data2)
                        }
                        // запуск сканера
                        dir ("${env.workspace}"){sh "docker build -t myscanner ."}
                        withCredentials([string(credentialsId: "sonar_secret", variable: 'sonar_secret')]){
                            TEST = sh(returnStdout: true, script:"docker run myscanner sonar-scanner -Dproject.settings=sonar.properties -Dsonar.login=${sonar_secret}").trim()
                        }
                        sh "rm -R ${env.workspace}/src"
                        string2 = ""
                        for (y in TEST) {
                            string2 = string2+y
                            if (string2.contains ("INFO: Analysis report uploaded in")) {string2 = ""}
                            // echo string2
                            if (string2.contains("INFO: Note that you will be able to access")) {
                                //echo string2
                                break}
                        }
                        
                        echo "---write to urls.txt---"
                        string2 = string2[12..(string2.length()-43)].trim()
                        sh """
                            echo ${string2} >> urls.txt
                            cat urls.txt
                        """
 
                    }
                    }
                }
            }
        }
       
        stage ('clean workspace') {
            steps{
                script{
                    URLS = sh (returnStdout: true, script: 'cat urls.txt').trim()
                   
                    if (URLS.size() == 0) {URLS = "ANALYSIS IS FAILED"}
                }
                emailext  to: "${Email}",
                    recipientProviders: [[$class: 'RequesterRecipientProvider']],
                    subject: "SonarQube Analysis report: ${env.Repo}::${PullRequest}",
                    body: "${URLS}"
                sh "docker system prune -a"
                sh "rm -Rf ${env.workspace}/*"
                //sh "docker rm -f $$(docker ps -aq)"
            }
        }
    }
}
 
// функция для поиска переменных
def takeParam(input_param, path){
out_param = sh(
          script: """grep -oP '(?<=${input_param}\\=).*' ${path}""",
          returnStdout: true
        ).trim()
sh "echo '${input_param}' = '${out_param}'"
return out_param
}
 
// функция для поиска версии Java в build.gradle
def getJavaFromBuildGrade(input_param, path){
    print("---getJava...---")
    print(path)
    print(input_param)
    out_param = sh(
            script: """grep -oP '(?<=${input_param}\\ =).*' ${path}""",
            returnStdout: true
        ).trim()
    return out_param
}
 
// функция проверки содержит ли build.gradle версию java
def checkJavaVersionInBuildGradle(input_param, path){
    print("---checkJava...---")
    print(path)
    print(input_param)   
    out_param = sh(
            script: """grep -oP '(?<=${input_param}\\ =).*' ${path} | wc -l """,
            returnStdout: true).trim()
    print(out_param)
    return out_param
}