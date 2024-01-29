pipeline {
    agent any
    tools {
        terraform 'terraform'
    }
    environment {
        JENKINS_URL = "http://44.211.190.166:8080"
        NODE_NAME = "jnlp-node"
        NODE_DESCRIPTION = "Jenkins Agent Node"
        AWS_REGION = "us-east-1"
        instance_id = ''
        instance_ip = ''
        TERRAFORM_HOME = tool 'terraform'
        PATH = "${TERRAFORM_HOME}/bin:${env.PATH}"
    }

     stages {
        stage('Create Jenkins Agent Node') {
            steps {
                script {
                    sh """
                        curl -O $JENKINS_URL/jnlpJars/jenkins-cli.jar
                        java -jar jenkins-cli.jar -s $JENKINS_URL -webSocket create-node $NODE_NAME <<EOF
                        <slave>
                            <name>$NODE_NAME</name>
                            <description>$NODE_DESCRIPTION</description>
                            <remoteFS>/home/ubuntu</remoteFS>
                            <numExecutors>2</numExecutors>
                            <mode>NORMAL</mode>
                            <retentionStrategy class="hudson.slaves.RetentionStrategy\$Always"/>
                            <launcher class="hudson.slaves.JNLPLauncher" />
                            <label>jnlp</label>
                            <nodeProperties/>
                        </slave>
                        EOF
                    """
                }
            }
        }

        stage('Launch EC2 Instance') {
            steps {
                script {
                    instance_id = sh (
                        script: """
                            aws ec2 run-instances \
                                --image-id ami-0c7217cdde317cfec \
                                --instance-type t2.micro \
                                --key-name nabeel \
                                --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=jnlp-slave}]" \
                                --subnet-id subnet-01250bf99475a9e1c \
                                --security-group-ids sg-04af7f874de8c1104 \
                                --region $AWS_REGION \
                                --query 'Instances[0].InstanceId' \
                                --associate-public-ip-address \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()
                    env.instance_id = instance_id

                    sh "aws ec2 wait instance-running --instance-ids $instance_id --region $AWS_REGION"

                    sh "aws ec2 wait instance-status-ok --instance-ids $instance_id --region $AWS_REGION"

                    //public IP of the instance
                    instance_ip = sh (
                        script: """
                            aws ec2 describe-instances \
                                --instance-ids $instance_id \
                                --region $AWS_REGION \
                                --query 'Reservations[0].Instances[0].PublicIpAddress' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Instance IP: $instance_ip"
                    env.instance_ip = instance_ip

                    sh "curl -O $JENKINS_URL/jnlpJars/agent.jar"

                    sh "scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/Jenkinsfile/nabeel.pem agent.jar ubuntu@$instance_ip:/home/ubuntu/agent.jar"
                }
            }
        }

        stage('Launch Jenkins Agent') {
            steps {
                script {
                    sh "ssh -v -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/Jenkinsfile/nabeel.pem ubuntu@$instance_ip 'sudo apt-get update'"
                    sh "ssh -v -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/Jenkinsfile/nabeel.pem ubuntu@$instance_ip 'sudo apt install openjdk-11-jre-headless -y'"
                    sh "ssh -v -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/Jenkinsfile/nabeel.pem ubuntu@$instance_ip 'nohup java -jar /home/ubuntu/agent.jar -jnlpUrl $JENKINS_URL/computer/$NODE_NAME/slave-agent.jnlp > /dev/null 2>&1 & disown'"   
                    // sh """
                    //     ssh -v -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/Jenkinsfile/nabeel.pem ubuntu@$instance_ip \
                    //         "sudo apt-get update &&
                    //         sudo apt install openjdk-11-jre-headless -y &&
                    //         nohup java -v -jar /home/ubuntu/agent.jar -jnlpUrl $JENKINS_URL/computer/$NODE_NAME/slave-agent.jnlp > /dev/null 2>&1 &
                    //     "
                    // """
                    
                }
                            // nohup java -jar /home/ubuntu/agent.jar -jnlpUrl $JENKINS_URL/computer/$NODE_NAME/slave-agent.jnlp > /dev/null 2>&1 & 
                            // disown
                            // nohup java -jar /home/ubuntu/agent.jar -jnlpUrl $JENKINS_URL/computer/$NODE_NAME/slave-agent.jnlp > /dev/null 2>&1 &
                            // disown
            }
        }
      }
}
