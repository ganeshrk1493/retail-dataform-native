pipeline {
    agent any

    environment {
        PROJECT_ID    = 'sa-testing-515015'
        REGION        = 'asia-south1'
        REPO_ID       = 'retail-dataform-repo'
        EXECUTOR_SA   = 'sa-dataform-executor@sa-testing-515015.iam.gserviceaccount.com'
        DATAFORM_API  = "https://dataform.googleapis.com/v1beta1/projects/${PROJECT_ID}/locations/${REGION}/repositories/${REPO_ID}"
    }

    stages {
        stage('1. Verify Active GCP Service Account') {
            steps {
                sh 'gcloud auth list'
            }
        }

        stage('2. Compile in GCP Dataform') {
            steps {
                script {
                    def compileResponse = sh(
                        script: '''
                            ACCESS_TOKEN=$(gcloud auth print-access-token)
                            curl -s -X POST "${DATAFORM_API}/compilationResults" \
                              -H "Authorization: Bearer ${ACCESS_TOKEN}" \
                              -H "Content-Type: application/json" \
                              -d '{"gitCommitish": "main"}'
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Compile Response: ${compileResponse}"

                    env.COMPILATION_NAME = sh(
                        script: "python3 -c 'import sys, json; print(json.loads(sys.argv[1])[\"name\"])' '${compileResponse}'",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('3. Invoke Dataform Workflow') {
            steps {
                sh '''
                    ACCESS_TOKEN=$(gcloud auth print-access-token)
                    curl -s -X POST "${DATAFORM_API}/workflowInvocations" \
                      -H "Authorization: Bearer ${ACCESS_TOKEN}" \
                      -H "Content-Type: application/json" \
                      -d "{
                        \\"compilationResult\\": \\"${COMPILATION_NAME}\\",
                        \\"invocationConfig\\": {
                          \\"serviceAccount\\": \\"${EXECUTOR_SA}\\"
                        }
                      }"
                '''
            }
        }
    }
}