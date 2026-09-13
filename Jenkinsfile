pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Install deps') {
            steps {
                sh 'pip install uv'
                sh 'uv sync'
            }
        }
        stage('Smoke test') {
            steps {
                sh 'uv run python -c "import weather; print(\'weather.py imports cleanly\')"'
            }
        }
        stage('Build container image') {
            steps {
                sh 'podman build -t mcp-demo:${BUILD_NUMBER} .'
            }
        }
    }
}
