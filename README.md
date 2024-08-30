# Deploying an Amazon EKS Application Using CDK8s

This guide details how to deploy an Amazon EKS cluster using AWS CDK and create and deploy a Service and Deployment using CDK8s.
![CDk8s](https://github.com/user-attachments/assets/094c1c7f-363c-4f56-a607-4d4a17472210)

## Prerequisites

- AWS Account
- Basic knowledge of AWS services and Kubernetes
- AWS Cloud9 IDE

## Task 1: Configure Your AWS Cloud9 IDE

1. **Open Cloud9 IDE**
   - In the AWS Management Console, search for **Cloud9**.
   - Select **Cloud9-Lab-IDE** and click **Open**.

2. **Open a New Terminal**
   - Click the terminal icon at the top of the IDE and choose **New Terminal**.

3. **Disable Temporary Credentials**
   - Open **Preferences** by clicking the gear icon in the top-right.
   - In the navigation panel, scroll to **AWS Settings**.
   - Toggle off **AWS managed temporary credentials**.

4. **Enable Auto-Save**
   - In **Preferences**, go to **Experimental**.
   - Set **Auto-Save Files** to **On Focus Change**.

5. **Confirm AWS Credentials**
   - Run the following command to confirm that AWS managed temporary credentials are disabled:
     ```bash
     aws sts get-caller-identity
     ```

6. **Install CDK8s CLI**
   - Run the following command to install the CDK8s CLI:
     ```bash
     npm install -g cdk8s-cli
     ```

7. **Update CDK Version**
   - Update the CDK version by running:
     ```bash
     npm install -g aws-cdk@latest --force
     ```

## Task 2: Deploy Amazon EKS Cluster Using CDK

1. **Create Project Directory**
   - Create a project root directory called `cdk` and navigate into it:
     ```bash
     mkdir cdk && cd cdk
     ```

2. **Initialize AWS CDK Project**
   - Create a new AWS CDK project in TypeScript:
     ```bash
     cdk init -l typescript
     ```

3. **View Project Structure**
   - Expand the `cdk` directory in the navigation panel to view its contents.

4. **Edit `cdk.ts`**
   - Navigate to `/cdk/bin/cdk.ts` and replace its contents with the following:
     ```typescript
     #!/usr/bin/env node
     import 'source-map-support/register';
     import * as cdk from 'aws-cdk-lib';
     import { CdkStack } from '../lib/cdk-stack';

     const app = new cdk.App();
     new CdkStack(app, 'CdkStack', {
       // env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION },
     });
     ```

5. **Edit `cdk-stack.ts`**
   - Open `/cdk/lib/cdk-stack.ts` and replace the constructor with the following:
     ```typescript
     import * as eks from 'aws-cdk-lib/aws-eks';
     import * as iam from 'aws-cdk-lib/aws-iam';

     export class CdkStack extends cdk.Stack {
       constructor(scope: Construct, id: string, props?: cdk.StackProps) {
         super(scope, id, props);

         const cluster = new eks.Cluster(this, 'hello-eks', {
           version: eks.KubernetesVersion.V1_28,
           albController: {
             version: eks.AlbControllerVersion.V2_4_1
           },
         });

         const role = iam.Role.fromRoleName(this, 'admin-role', 'Cloud9InstanceRole');
         cluster.awsAuth.addRoleMapping(role, { groups: [ 'system:masters' ]});

         new cdk.CfnOutput(this, 'ConfigCommand', {
           value: `aws eks update-kubeconfig --name ${cluster.clusterName} --region ${this.region}`
         });
       }
     }
     ```

6. **Bootstrap and Deploy**
   - View the resources created by the bootstrap template:
     ```bash
     cat node_modules/aws-cdk/lib/api/bootstrap/bootstrap-template.yaml | yq '.Resources[].Type'
     ```
   - Launch the bootstrap stack:
     ```bash
     cdk bootstrap
     ```
   - Compile and build the application:
     ```bash
     npm install typescript@latest
     npm fund
     npm run build
     ```
   - View the CloudFormation snippet for the node group:
     ```bash
     cat cdk.out/CdkStack.template.json | jq '.Resources[]|select(.Type=="AWS::EKS::Nodegroup")'
     ```
   - Deploy the application:
     ```bash
     cdk deploy
     ```
   - Confirm the deployment when prompted:
     ```bash
     y
     ```

7. **Verify Cluster Connection**
   - Update your kubeconfig:
     ```bash
     aws eks update-kubeconfig --name <CLUSTER_NAME> --region <REGION>
     ```
   - Confirm connectivity:
     ```bash
     kubectl get nodes
     ```
   - View your pods:
     ```bash
     kubectl get pods -n kube-system
     ```

## Task 3: Create and Deploy a CDK8s Chart

1. **Create Application Directory**
   - Create a directory for your web application code:
     ```bash
     cd ~/environment/
     mkdir cdk8s-app && cd cdk8s-app
     ```

2. **Initialize CDK8s App**
   - Create a new CDK8s app in TypeScript:
     ```bash
     cdk8s init typescript-app
     ```

3. **Edit `main.ts`**
   - Open `cdk8s-app/main.ts` and replace the contents with the following:
     ```typescript
     import { Construct } from 'constructs';
     import { App, Chart, ChartProps } from 'cdk8s';
     import { KubeDeployment, KubeService, IntOrString } from "./imports/k8s";

     export class MyChart extends Chart {
       constructor(scope: Construct, id: string, props: ChartProps = { }) {
         super(scope, id, props);

         const label = { app: 'hello-k8s' };

         new KubeService(this, 'service', {
           spec: {
             type: 'LoadBalancer',
             ports: [ { port: 80, targetPort: IntOrString.fromNumber(8080) } ],
             selector: label
           }
         });

         new KubeDeployment(this, 'deployment', {
           spec: {
             replicas: 2,
             selector: {
               matchLabels: label
             },
             template: {
               metadata: { labels: label },
               spec: {
                 containers: [
                   {
                     name: 'hello-kubernetes',
                     image: 'paulbouwer/hello-kubernetes:1.7',
                     ports: [ { containerPort: 8080 } ]
                   }
                 ]
               }
             }
           }
         });
       }
     }

     const app = new App();
     new MyChart(app, 'hello');
     app.synth();
     ```

4. **Update `cdk8s.yaml`**
   - Update the imports label in `cdk8s.yaml` to use Kubernetes version 1.28:
     ```yaml
     language: typescript
     app: npx ts-node main.ts
     imports:
       - k8s@1.28.0
     ```

5. **Synthesize and Apply**
   - Synthesize your CDK8s app:
     ```bash
     npm run compile && cdk8s synth
     ```
   - View the generated Kubernetes manifest:
     ```bash
     cat dist/cdk8s-app.k8s.yaml
     ```
   - Apply the manifest:
     ```bash
     kubectl apply -f dist/cdk8s-app.k8s.yaml
     ```

6. **Access the Application**
   - Retrieve the application frontend URL:
     ```bash
     kubectl get service | awk '/cdk8s/ { print $4 }'
     ```
   - Copy the URL into a browser to access the application.

7. **Expected Output**
   - You should see a "Hello world!" message along with information about the Kubernetes pod and node it is running on.

