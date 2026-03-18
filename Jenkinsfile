pipeline {

    agent any
 
   
    environment {
        PERF_EMAIL = 'manishas@ivavsys.com'
        PT_REPO = 'https://github.com/ManishaS1713/Jenkins_Pipeline_PT.git'

    }
    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
    stages {
 
     stage('Verify JMeter') {
        steps {
                bat """
                    "C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat" -v
                    """
            }
     }
 
     stage('Checkout Performance Code') {
        steps {
                 
                    git branch: 'main',
                        url: "${PT_REPO}",
                        credentialsId: 'PT_PipelineToken'
                 
        }

     }
 
    stage('Run Performance Tests') {
    steps {
        
            bat '''
            C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat -n ^
            -t prefScale.jmx ^
            -l performance-result.jtl ^
            -e -o performance-report
            '''
        
    }
}
 
     stage('Email After Performance') {
        steps {
                emailext(
                    subject: "JMeter Performance Test Report",
                    body: """
						Performance Test Execution Completed
		                """,
                    to: "${PERF_EMAIL}"
                )
            }
      }
    }
 

}
