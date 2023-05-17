***Twoge Webpage**

Twoge Documentation 

<img width="805" alt="image" src="https://github.com/perryb3693/twoge_app/assets/129805541/a9e1c429-c84b-49e2-a19f-7acbdbf7c30a">


Twoge Architecture Diagram – Key Points
The Twoge VPC has public and private subnets in two availability zones
The Twoge web servers run in the public subnets and receive traffic through a load balancer
The web servers across both availability zones are both configured for auto scaling. 
The database server runs in the private subnet and receives traffic from the web servers through the routing table. 
The private subnets have access to the internet through the NAT gateway.
Further Improvements:
Multi-AZ Database instance in private subnet 2 a failover for the primary database
Removal of the NAT gateway in favor of the S3 gateway, depending on networking requirements for the private subnets
Milestones
 Twoge VPC- Consist of two public and two private subnets split between two Avs. One nat gateway for each private subnet for redundancy. Internet gateway configured to route traffic to the internet.
S3 bucket-  Created to store images and videos and configured for access to the public
S3 IAM Role: Created to grant full access to the S3 storage to public users of Twoge
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "s3-object-lambda:*"
            ],
            "Resource": "*"
        }
    ]
}
```
EC2 Instances: One Instance created per subnet. Twoge setup and configured on public web servers using Chandra’s Twoge Setup Script. Edited to use new RDS.
```
sudo yum update -y
sudo yum install git -y
git clone  https://github.com/chandradeoarya/twoge
cd twoge
sudo yum install python-pip -y
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
echo 'SQLALCHEMY_DATABASE_URI = "postgresql://postgres:password@twoge-privatedb.ctfbo8wbotzl.ca-central-1.rds.amazonaws.com"' > .env
echo '
Description=Gunicorn instance to serve twoge
Wants=network.target
After=syslog.target network-online.target
[Service]
Type=simple
WorkingDirectory=/home/ec2-user/twoge
Environment="PATH=/home/ec2-user/twoge/venv/bin"
ExecStart=/home/ec2-user/twoge/venv/bin/gunicorn app:app -c /home/ec2-user/twoge/gunicorn_config.py
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target' > twoge.service
sudo cp twoge.service /etc/systemd/system/twoge.service
sudo systemctl daemon-reload
sudo systemctl enable twoge
sudo systemctl start twoge
sudo systemctl status twoge
```
ALB: Load Balancer configured for HTTP traffic going to the Twoge web servers, unable to listen for HTTPS traffic valid certificate from CA
Auto Scaling Group: Autoscale group developed based off of an AMI launch template of the configured Twoge Web server Instance
Amazon SNS Config (Activity): auto scaling group activity notifications sent to subscribers through Amazon SNS 
Dynamic Scaling Policies configured to add an ec2 instance once CPU utilization has risen to over 80% for over 60 seconds
Stress Test:
```
#!/usr/bin/env python
"""
Produces load on all available CPU cores
"""
from multiprocessing import Pool
from multiprocessing import cpu_count
def f(x):
while True:
x*x
if __name__ == '__main__':
processes = cpu_count()
print ('utilizing %d cores\n' % processes)
pool = Pool(processes)
pool.map(f, range(processes))
```

