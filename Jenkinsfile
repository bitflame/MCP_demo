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
                sh 'curl -LsSf https://astral.sh/uv/install.sh | sh'
                sh 'export PATH="$HOME/.local/bin:$PATH" && uv sync'
            }
        }
        stage('Smoke test') {
            steps {
                sh 'export PATH="$HOME/.local/bin:$PATH" && uv run python -c "import weather; print(\'weather.py imports cleanly\')"'
            }
        }
        /*
        stage('Build container image') {
            steps {
                sh 'podman build -t mcp-demo:${BUILD_NUMBER} .'
            }
        }
        */
    }
}
