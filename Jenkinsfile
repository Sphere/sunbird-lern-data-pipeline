
// Runs Maven inside a container instead of needing it on the agent
def mvn(String args) {
    sh """
        mkdir -p "\$WORKSPACE/.build-home/.m2"
        docker run --rm \\
            -u "\$(id -u):\$(id -g)" \\
            -v "\$PWD":/src \\
            -v "\$WORKSPACE/.build-home":/var/maven \\
            -e HOME=/var/maven \\
            -e MAVEN_CONFIG=/var/maven/.m2 \\
            -w /src \\
            maven:3.9-eclipse-temurin-11 \\
            mvn -B -Duser.home=/var/maven ${args}
    """
}

node {
    // Required job parameters - no defaults, so each job must set its own
    def branch    = (params.BRANCH ?: '').trim()
    def imageRepo = (params.IMAGE_REPO ?: '').trim()

    if (!branch)    { error 'BRANCH parameter is required' }
    if (!imageRepo) { error 'IMAGE_REPO parameter is required' }

    try {
        stage('Checkout') {
           
            // Uses the repo + branch from the job's SCM config (Branch Specifier: */${BRANCH})
            checkout scm

            def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
            // Docker tags can't contain '/', so feature/foo -> feature-foo
            def safeBranch  = branch.replaceAll('[^A-Za-z0-9_.-]', '-')
            env.IMAGE_TAG = "${safeBranch}_${shortCommit}_${env.BUILD_NUMBER}"
            echo "Image tag: ${env.IMAGE_TAG}"
        }

        stage('Maven Build') {
            mvn('clean install -DskipTests -Paws')
        }

        stage('maven packgae build') {
            sh "cd jobs-distribution/"
            mvn('clean package -DskipTests -Paws')
        }

        stage('Docker Build') {
            sh "cd jobs-distribution/"
            sh "docker build --target aws -t ${imageRepo}/${imageName}:${env.IMAGE_TAG} ."
           sh """echo '{"image_name" : "${imageName}", "image_tag" : "${env.IMAGE_TAG}"}' > metadata.json"""      
        }

         stage('ArchiveArtifacts') {
                    
                    archiveArtifacts "metadata.json"
                    currentBuild.description = "${imageRepo}:${env.IMAGE_TAG}"

                }
        currentBuild.result = 'SUCCESS'

    } catch (err) {
        currentBuild.result = 'FAILURE'
        echo "Build failed: ${err}"
        throw err
    } 
    }
