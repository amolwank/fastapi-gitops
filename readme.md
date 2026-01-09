Importance Notice:


This needs to run every 12 hours as 
kubectl create secret docker-registry ecr-secret \
  --docker-server=774305591014.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  --namespace=fastapi-namespace \
  --dry-run=client -o yaml | kubectl apply -f -