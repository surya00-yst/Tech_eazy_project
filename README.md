# Tech_eazy_project
#!/bin/bash

# Usage: ./deploy.sh <Stage>
# Example: ./deploy.sh Dev

set -e

STAGE=${1:-Dev}
CONFIG_FILE="${STAGE,,}_config.env"

if [ ! -f "$CONFIG_FILE" ]; then
  echo "Configuration file $CONFIG_FILE not found!"
  exit 1
fi

source $CONFIG_FILE

# Create Key Pair if not exists
if [ ! -f "$KEY_NAME.pem" ]; then
  aws ec2 create-key-pair --key-name "$KEY_NAME" --query 'KeyMaterial' --output text > "$KEY_NAME.pem"
  chmod 400 "$KEY_NAME.pem"
fi

# Launch EC2 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-groups $SECURITY_GROUP \
  --user-data file://user-data.sh \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Launched EC2 instance with ID: $INSTANCE_ID"

# Wait until instance is running
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# Get Public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "EC2 instance is running at: http://$PUBLIC_IP"

echo "Waiting 60 seconds for instance to initialize..."
sleep 60

# Test Application
if curl --silent --fail http://$PUBLIC_IP; then
  echo "App is reachable via port 80."
else
  echo "App is not reachable!"
fi

# Schedule shutdown after configured timeout
echo "Scheduling shutdown in $SHUTDOWN_TIME seconds..."
sleep $SHUTDOWN_TIME
aws ec2 stop-instances --instance-ids "$INSTANCE_ID"
echo "Instance $INSTANCE_ID stopped."
