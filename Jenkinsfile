pipeline {
    agent any

    parameters {
        string(name: 'EC2_HOST', defaultValue: '', description: 'EC2 instance IP or hostname')
        credentials(name: 'EC2_SSH_KEY', credentialType: 'com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey', required: true, description: 'SSH private key for EC2')
        string(name: 'EC2_USER', defaultValue: 'ec2-user', description: 'SSH user (ec2-user for Amazon Linux, ubuntu for Ubuntu)')
    }

    environment {
        NGINX_HTML_DIR = '/usr/share/nginx/html'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    if (!params.EC2_HOST?.trim()) {
                        error 'EC2_HOST parameter is required. Provide your EC2 instance IP or hostname.'
                    }
                }
                sshagent(credentials: [params.EC2_SSH_KEY]) {
                    sh """
                        # Install nginx if not present (idempotent for Amazon Linux 2 / RHEL)
                        ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} \
                            'sudo yum install -y nginx 2>/dev/null || sudo apt-get update && sudo apt-get install -y nginx'

                        # Ensure nginx html dir exists
                        ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} \
                            "sudo mkdir -p ${env.NGINX_HTML_DIR} && sudo chown \\\$USER:\\\$USER ${env.NGINX_HTML_DIR}"

                        # Copy index.html to EC2
                        scp -o StrictHostKeyChecking=no index.html ${params.EC2_USER}@${params.EC2_HOST}:${env.NGINX_HTML_DIR}/index.html

                        # Fix ownership if needed (some AMIs use nginx user)
                        ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} \
                            "sudo chown -R nginx:nginx ${env.NGINX_HTML_DIR} 2>/dev/null || sudo chown -R www-data:www-data ${env.NGINX_HTML_DIR} 2>/dev/null || true"

                        # Start and enable nginx
                        ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} \
                            'sudo systemctl start nginx; sudo systemctl enable nginx'
                    """
                }
            }
        }

        stage('Test index.html on EC2') {
            steps {
                sshagent(credentials: [params.EC2_SSH_KEY]) {
                    sh """
                        sleep 2
                        # Test on EC2 via localhost (works even if port 80 is not open to Jenkins)
                        ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} \
                            'curl -f -s -o /dev/null -w "%{http_code}" http://localhost/index.html' | grep -q 200 || exit 1
                        echo "Test passed: index.html returns HTTP 200 on EC2"
                        ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} \
                            'curl -s http://localhost/index.html' | head -15
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Build and deploy successful. Nginx is serving index.html at http://${params.EC2_HOST}/"
        }
        failure {
            echo "Build or deploy failed. Check EC2_HOST, SSH key, and security group (allow port 80)."
        }
    }
}
