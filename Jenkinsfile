pipeline {

    agent any

    parameters {
        choice(name: 'TEST_TYPE', choices: ['LOAD', 'STRESS', 'SPIKE'], description: 'Select Test Type')
        choice(name: 'ENV', choices: ['DEV', 'QA', 'PROD'], description: 'Select Environment')
    }

    environment {
        PT_REPO = 'https://github.com/ManishaS1713/PTTypes_Jenkins.git'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: "${PT_REPO}",
                    credentialsId: 'PT_PipelineToken'
            }
        }

        stage('Set Environment URL') {
            steps {
                script {
                    if (params.ENV == 'DEV') {
                        env.HOST = "dev.prefscale.com"
                    }
                    else if (params.ENV == 'QA') {
                        env.HOST = "qa.prefscale.com"
                    }
                    else {
                        env.HOST = "prefscale-frontend-9icy.onrender.com"
                    }

                    echo "Environment: ${params.ENV}"
                    echo "Host: ${env.HOST}"
                }
            }
        }

        stage('Set Test Type Configuration') {
            steps {
                script {

                    if (params.TEST_TYPE == 'LOAD') {
                        env.THREADS = "50"
                        env.RAMPUP = "10"
                        env.DURATION = "300"
                    }
                    else if (params.TEST_TYPE == 'STRESS') {
                        env.THREADS = "200"
                        env.RAMPUP = "20"
                        env.DURATION = "600"
                    }
                    else {
                        env.THREADS = "500"
                        env.RAMPUP = "1"
                        env.DURATION = "120"
                    }

                    echo "Test Type: ${params.TEST_TYPE}"
                    echo "Threads: ${env.THREADS}"
                    echo "Ramp-up: ${env.RAMPUP}"
                    echo "Duration: ${env.DURATION}"
                }
            }
        }

        stage('Run Performance Test') {
            steps {
                bat """
                IF EXIST PTTypes_Jenkins-result.jtl del PTTypes_Jenkins-result.jtl
                IF EXIST PTTypes_Jenkins-report rmdir /s /q PTTypes_Jenkins-report

                C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat -n ^
                -t PTTypes_Jenkins.jmx ^
                -Jthreads=${env.THREADS} ^
                -Jrampup=${env.RAMPUP} ^
                -Jduration=${env.DURATION} ^
                -Jhost=${env.HOST} ^
                -l PTTypes_Jenkins-result.jtl ^
                -e -o PTTypes_Jenkins-report
                """
            }
        }

        stage('Generate Summary') {
            steps {
                script {

                    def TOTAL, SUCCESS, FAIL, AVG, TPS, ERRORPCT, SLA_STATUS

                    def raw = powershell(
                        returnStdout: true,
                        script: '''
                        $data = Import-Csv "PTTypes_Jenkins-result.jtl"

                        $total = $data.Count
                        $success = ($data | Where-Object {$_.success -eq "true"}).Count
                        $fail = $total - $success
                        $avg = [math]::Round(($data | Measure-Object -Property elapsed -Average).Average,2)

                        $start = $data[0].timeStamp
                        $end = $data[-1].timeStamp
                        $duration = ($end - $start)/1000

                        if ($duration -eq 0) { $duration = 1 }

                        $tps = [math]::Round($total / $duration,2)
                        $errorPct = [math]::Round(($fail/$total)*100,2)

                        Write-Output "TOTAL=$total"
                        Write-Output "SUCCESS=$success"
                        Write-Output "FAIL=$fail"
                        Write-Output "AVG=$avg"
                        Write-Output "TPS=$tps"
                        Write-Output "ERRORPCT=$errorPct"
                        '''
                    ).trim()

                    echo raw

                    def lines = raw.split("\\r?\\n")

                    TOTAL = lines.find { it.startsWith("TOTAL=") }?.split("=")[1]
                    SUCCESS = lines.find { it.startsWith("SUCCESS=") }?.split("=")[1]
                    FAIL = lines.find { it.startsWith("FAIL=") }?.split("=")[1]
                    AVG = lines.find { it.startsWith("AVG=") }?.split("=")[1]
                    TPS = lines.find { it.startsWith("TPS=") }?.split("=")[1]
                    ERRORPCT = lines.find { it.startsWith("ERRORPCT=") }?.split("=")[1]

                    SLA_STATUS = "PASS"
                    if ((AVG ?: "0").toFloat() > 1000 || (ERRORPCT ?: "0").toFloat() > 1) {
                        SLA_STATUS = "FAIL"
                    }

                    // Store globally for email stage
                    env.TOTAL = TOTAL
                    env.SUCCESS = SUCCESS
                    env.FAIL = FAIL
                    env.AVG = AVG
                    env.TPS = TPS
                    env.ERRORPCT = ERRORPCT
                    env.SLA_STATUS = SLA_STATUS
                }
            }
        }

        stage('Publish Report') {
            steps {
                publishHTML([
                    allowMissing: true,
                    reportDir: 'PTTypes_Jenkins-report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true
                ])
            }
        }

        stage('Send Email') {
            steps {
                script {
                    emailext(
                        subject: "Performance Test Result - ${env.SLA_STATUS}",
                        body: """
<h2>Performance Test Report (${params.ENV})</h2>

<b>Total Requests:</b> ${env.TOTAL} <br>
<b>Success:</b> ${env.SUCCESS} <br>
<b>Failures:</b> ${env.FAIL} <br>
<b>Avg Response Time:</b> ${env.AVG} ms <br>
<b>Throughput:</b> ${env.TPS} req/sec <br>
<b>Error %:</b> ${env.ERRORPCT} % <br>

<br><b>SLA Status:</b> <span style="color:${env.SLA_STATUS == 'PASS' ? 'green' : 'red'}">${env.SLA_STATUS}</span>

<br><br>
<b>Environment:</b> ${params.ENV} <br>
<b>Host:</b> ${env.HOST} <br>

<br>
<a href="${BUILD_URL}">👉 View Full Report</a>
""",
                        to: 'manishas@ivavsys.com,team@company.com',
                        mimeType: 'text/html'
                    )
                }
            }
        }
    }
}
