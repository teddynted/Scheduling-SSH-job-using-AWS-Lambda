How to schedule an ssh cron job on AWS Lambda(Node.js) function to upload a csv file to an EC2 instance? The job will run between Monday and Friday at 3am. 

_Cron Jobs are used for scheduling tasks to run on the server. They're most commonly used for automating system maintenance or administration._

<p class="markdown-paragraph">The code is more or less the same as the one on this <a class="markdown-link" href="https://github.com/teddynted/SSH-into-EC2-instance-from-AWS-Lambda">repo</a>. We will just make minor changes to cater for a cron job functionality.</p>

```bash
serverless create --template aws-nodejs --path scheduling-ssh-job-service --name scheduling-ssh-job-using-aws-lambda
cd scheduling-ssh-job-service
npm init -y
npm i json2csv ssh2-sftp-client 
```

`serverless.yml`:

```yaml
service: scheduling-ssh-job-using-aws-lambda

frameworkVersion: '2'

provider:
  name: aws
  runtime: nodejs10.x
  stage: dev
  region: us-east-1
  iamRoleStatements:
  - Effect: Allow
    Action:
      - ec2:CreateNetworkInterface
      - ec2:DescribeNetworkInterfaces
      - ec2:DeleteNetworkInterface
    Resource: '*'

functions:
  sshScheduler:
    handler: handler.sshScheduler
    events:
      - schedule:
          rate: cron(0 3 ? * MON-FRI *)
          enabled: true
```

`handler.js`:

```javascript
'use strict';

const fs = require('fs');

const uploadFileToSftpServer = async (content, remoteFolder) => {
      try {
          const connSettings = {
            host: 'public-ip-address', // EC2 public IP address
            port: 22, 
            username: 'ubuntu',
            privateKey: fs.readFileSync('./path/to/your/key.pem'), // EC2 keypair
          };
          const Client = require('ssh2-sftp-client');
          const { Parser } = require('json2csv');
          const sftp = new Client();
          // Extract fields' headers from a JSON object / file
          const fields = Object.keys(content[0]);
          const json2csvParser = new Parser({fields, quote: ''});
          const csv = json2csvParser.parse(content);
          // Define the name of the file to be uploaded to server
          const file = `colors.csv`;
          const awsTempFolder = `/tmp/${file}`;
          // Create your file in a Lambda temporary folder/directory 
          fs.writeFileSync(awsTempFolder, csv);
          const data = fs.createReadStream(awsTempFolder);
          // Define an absolute path of your remote sftp server
          const remotePath = `${remoteFolder}${file}`;
          // Connect to your server to dump that csv file 
          console.log('Connecting....');
          await sftp.connect(connSettings);
          // Upload the file, if your connection is successful
          console.log('Uploading....');
          await sftp.put(data, remotePath);
          // Remove/delete your file from a Lambda's temp folder/directory
          fs.unlinkSync(awsTempFolder);
          // And lastly disconnect from your remote server
          console.log(`File Uploaded!`);
          return await sftp.end();
      } catch (err) {
          console.log('File upload failed because ', err.message);
          return err.message;
      }
};

module.exports.sshScheduler = async (event, context) => {
  // Sample JSON data
  const data = [
    {
      color: 'red',
      value: '#f00'
    },
    {
      color: 'green',
      value: '#0f0'
    },
    {
      color: 'blue',
      value: '#00f'
    },
    {
      color: 'cyan',
      value: '#0ff'
    },
    {
      color: 'magenta',
      value: '#f0f'
    },
    {
      color: 'yellow',
      value: '#ff0'
    },
    {
      color: 'black',
      value: '#000'
    }
  ];
  const res = await uploadFileToSftpServer(data, 'Uploads/');
  const response = {
    statusCode: 200,
    body: res,
  };
  console.log(`Display response${response}`);
  return response;
};

```