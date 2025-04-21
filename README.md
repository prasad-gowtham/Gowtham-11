# Gowtham-11
 Automated Deployment Pipeline for a Scalable Web Application on AWS
import boto3

def create_s3_bucket(bucket_name, region='us-east-1'):
    """Creates an S3 bucket in the specified region."""
    s3_client = boto3.client('s3', region_name=region)
    try:
        response = s3_client.create_bucket(
            Bucket=bucket_name,
            CreateBucketConfiguration={'LocationConstraint': region}
        )
        print(f"S3 bucket '{bucket_name}' created successfully in {region}.")
        return response
    except Exception as e:
        print(f"Error creating S3 bucket '{bucket_name}': {e}")
        return None

import subprocess
import sys

def run_tests():
    """Runs unit and integration tests."""
    try:
        print("Running unit tests...")
        unit_test_process = subprocess.run(['pytest', 'tests/unit'], check=True, capture_output=True, text=True)
        print(unit_test_process.stdout)

        print("\nRunning integration tests...")
        integration_test_process = subprocess.run(['pytest', 'tests/integration'], check=True, capture_output=True, text=True)
        print(integration_test_process.stdout)

        print("\nAll tests passed!")
        return 0  # Success
    except subprocess.CalledProcessError as e:
        print(f"Error running tests:")
        print(e.stderr)
        return 1  # Failure
    except FileNotFoundError as e:
        print(f"Error: {e}. Make sure pytest is installed and the test directories exist.")
        return 1

if __name__ == "__main__":
    sys.exit(run_tests())        

def main():
    bucket_name = "your-unique-app-artifacts-bucket"  # Replace with a unique name
    region = "ap-south-1"  # Replace with your desired AWS region (e.g., us-east-1)
    create_s3_bucket(bucket_name, region)

if __name__ == "__main__":
    main()

import boto3
import os
import time

def get_instance_ips(tag_key, tag_value, region='ap-south-1'):
    """Retrieves the public IP addresses of EC2 instances with the specified tag."""
    ec2 = boto3.client('ec2', region_name=region)
    filters = [{'Name': f'tag:{tag_key}', 'Values': [tag_value]}]
    response = ec2.describe_instances(Filters=filters)
    ips = []
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            if 'PublicIpAddress' in instance:
                ips.append(instance['PublicIpAddress'])
    return ips

def copy_files_to_instance(instance_ip, source_dir, remote_user='ec2-user', remote_path='/home/ec2-user/app'):
    """Copies files to an EC2 instance using SCP (requires SSH key configuration)."""
    try:
        print(f"Copying files to {remote_user}@{instance_ip}:{remote_path}...")
        command = [
            'scp',
            '-r',
            source_dir + '/',  # Ensure trailing slash for recursive copy
            f'{remote_user}@{instance_ip}:{remote_path}'
        ]
        subprocess.run(command, check=True)
        print("Files copied successfully.")
        return True
    except subprocess.CalledProcessError as e:
        print(f"Error copying files to {instance_ip}: {e}")
        return False
    except FileNotFoundError:
        print("Error: 'scp' command not found. Make sure it's installed.")
        return False

def run_remote_command(instance_ip, command_to_run, remote_user='ec2-user'):
    """Runs a command on a remote EC2 instance using SSH."""
    try:
        print(f"Running command '{command_to_run}' on {remote_user}@{instance_ip}...")
        ssh_command = [
            'ssh',
            '-o', 'StrictHostKeyChecking=no',  # Avoid prompting for adding to known_hosts
            f'{remote_user}@{instance_ip}',
            command_to_run
        ]
        process = subprocess.run(ssh_command, check=True, capture_output=True, text=True)
        print(process.stdout)
        if process.stderr:
            print(f"Errors: {process.stderr}")
        return True
    except subprocess.CalledProcessError as e:
        print(f"Error running command on {instance_ip}: {e}")
        print(e.stderr)
        return False
    except FileNotFoundError:
        print("Error: 'ssh' command not found. Make sure it's installed.")
        return False

def main():
    app_tag_key = "Project"  # Replace with your EC2 instance tag key
    app_tag_value = "MyWebApp" # Replace with your EC2 instance tag value
    region = "ap-south-1"     # Replace with your AWS region
    artifact_dir = "build_output" # Replace with the directory containing your built application
    remote_deploy_path = "/home/ec2-user/app" # Replace with your desired remote path
    start_app_command = "nohup python /home/ec2-user/app/main.py &" # Example command

    instance_ips = get_instance_ips(app_tag_key, app_tag_value, region)
    if not instance_ips:
        print("No EC2 instances found with the specified tag.")
        return

    for ip in instance_ips:
        if copy_files_to_instance(ip, artifact_dir, remote_path=remote_deploy_path):
            time.sleep(5) # Give time for files to copy
            run_remote_command(ip, f"mkdir -p {remote_deploy_path}") # Ensure directory exists
            run_remote_command(ip, start_app_command)
            print(f"Application deployed and started on {ip}")

if __name__ == "__main__":
    main()

import boto3

def get_cpu_utilization(instance_id, region='ap-south-1', period=300, statistics='Average'):
    """Gets the CPU utilization metric for a given EC2 instance."""
    cloudwatch = boto3.client('cloudwatch', region_name=region)
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[
            {
                'Name': 'InstanceId',
                'Value': instance_id
            },
        ],
        StartTime=datetime.utcnow() - timedelta(seconds=period * 5), # Last 25 minutes
        EndTime=datetime.utcnow(),
        Period=period,
        Statistics=[statistics]
    )
    datapoints = response['Datapoints']
    if datapoints:
        # Sort by timestamp to get the latest value
        datapoints.sort(key=lambda x: x['Timestamp'], reverse=True)
        return datapoints[0]['Average']
    else:
        return None

if __name__ == "__main__":
    from datetime import datetime, timedelta
    instance_id = "i-xxxxxxxxxxxxxxxxx"  # Replace with your instance ID
    region = "ap-south-1"
    cpu_utilization = get_cpu_utilization(instance_id, region)
    if cpu_utilization is not None:
        print(f"CPU Utilization for {instance_id}: {cpu_utilization}%")
    else:
        print(f"No CPU Utilization data found for {instance_id}.")    
