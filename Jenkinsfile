node {
  timeout(time: 30, unit: 'MINUTES') {
    withEnv([
      'JAVA_HOME=/usr/lib/jvm/java-11-openjdk-arm64',
      'PATH+JAVA=/usr/lib/jvm/java-11-openjdk-arm64/bin',
      'MAVEN_OPTS=-Xmx512m',
      'AZURE_SUBSCRIPTION_ID=cf22e882-a51f-47d4-bef5-05443446859f',
      'AZURE_TENANT_ID=21177912-5e6b-44c0-b693-e7ab7b281738',
      'AZURE_RESOURCE_GROUP=jenkins-get-started-rg',
      'AZURE_WEB_APP=max18kp-ex8-20261028'
    ]) {
      stage('Checkout') {
        checkout scm
      }

      stage('Build') {
        sh 'mvn --batch-mode --no-transfer-progress clean package'
        archiveArtifacts artifacts: 'target/calculator-1.0.war', fingerprint: true
      }

      stage('Deploy') {
        withCredentials([usernamePassword(
          credentialsId: 'AzureServicePrincipal',
          usernameVariable: 'AZURE_CLIENT_ID',
          passwordVariable: 'AZURE_CLIENT_SECRET'
        )]) {
          sh '''
            set +x
            set -eu
            umask 077
            export AZURE_CONFIG_DIR="$(mktemp -d)"
            trap 'az logout >/dev/null 2>&1 || true; rm -rf "$AZURE_CONFIG_DIR"' EXIT
            az login --service-principal --username "$AZURE_CLIENT_ID" \
              --password="$AZURE_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID" \
              --output none --only-show-errors
            az account set --subscription "$AZURE_SUBSCRIPTION_ID"
            az webapp deploy --resource-group "$AZURE_RESOURCE_GROUP" \
              --name "$AZURE_WEB_APP" --src-path target/calculator-1.0.war \
              --type war --timeout 600000 --output none --only-show-errors
            az webapp show --resource-group "$AZURE_RESOURCE_GROUP" \
              --name "$AZURE_WEB_APP" --query defaultHostName --output tsv \
              > deployed-hostname.txt
          '''
        }
      }

      stage('Verify calculator') {
        sh '''
          set -eu
          hostname="$(cat deployed-hostname.txt)"
          test -n "$hostname"
          attempt=0
          until curl --fail --silent --show-error --max-time 30 \
            "https://$hostname/api/calculator/ping" > calculator-ping.txt; do
            attempt=$((attempt + 1))
            if [ "$attempt" -ge 20 ]; then
              echo "Calculator did not become ready."
              exit 1
            fi
            sleep 15
          done
          curl --fail --silent --show-error --max-time 30 \
            "https://$hostname/api/calculator/add?x=23&y=33" > calculator-add.json
          python3 -c 'import json; result=json.load(open("calculator-add.json")); assert result["result"] == 56, result; print(result)'
          printf "Ping URL: https://%s/api/calculator/ping\\n" "$hostname"
          cat calculator-ping.txt
        '''
        archiveArtifacts artifacts: 'deployed-hostname.txt,calculator-ping.txt,calculator-add.json'
      }
    }
  }
}
