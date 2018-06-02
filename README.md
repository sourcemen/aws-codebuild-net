# aws-codebuild-net
aws codebuild with custom build environments for the .net framework

# Requirements

To follow the steps in this post and build the Docker container image, you need to have Docker for Windows (which is installed if you use a Windows Server with Containers AMI), the AWS Command Line interface, and Git installed. In this post, I’m using Visual Studio Build Tools 2017.

# Step 1: Launch EC2 Windows Server 2016 with Containers

In the Amazon EC2 console, in your region, launch an Amazon EC2 instance from a Microsoft Windows Server 2016 Base with Containers AMI.
Increase disk space on the boot volume to at least 50 GB to account for the larger size of containers required to install and run Visual Studio Build Tools.
When the instance is running, connect to it using Remote Desktop.
# Step 2: Build and push the Docker image

1. Open an elevated command prompt (that is, right-click on your favorite shell and choose Run as Administrator).
2. Create a directory:
mkdir C:\BuildTools
cd C:\BuildTools
3. Save the Dockerfile content to C:\BuildTools\Dockerfile

4. Run the following command in that directory. This process can take a while. It depends on the size of EC2 instance you launched. In my tests, a t2.2xlarge takes less than 30 minutes to build the image and produces an approximately 15 GB image.
docker build -t buildtools2017:latest -m 2GB .
5. Run the following command to test the container and start a command shell with all the developer environment variables:
docker run -it buildtools2017
6. Create a repository in the Amazon ECS console. For the repository name, type buildtools2017. Choose Next step and then complete the remaining steps.
7. Execute the following command to generate authentication details for our registry to the local Docker engine. Make sure you have permissions to the Amazon ECR registry before you execute the command.
aws ecr get-login
8. In the same command prompt window, copy and paste the following commands:
docker tag buildtools2017:latest [YOUR ACCOUNT #].dkr.ecr.[YOUR REGION].amazonaws.com/ buildtools2017:latest
docker push [YOUR ACCOUNT #].dkr.ecr.[YOUR REGION].amazonaws.com/buildtools2017:latest
Note: Make sure you replace [YOUR ACCOUNT #] with your AWS account number and [YOUR REGION] with the region you are using.

# Step 3: Configure AWS CodeCommit

1. Before you can access CodeCommit for the first time, you must complete the initial configuration steps.
2. In the CodeCommit console, create a repository named DotNetFrameworkSampleApp. On the Configure email notifications page, choose Skip.
3. Clone a .NET Framework Docker sample application from GitHub. The repository includes a sample ASP.NET Framework that we’ll use to demonstrate our custom build environment.On the EC2 instance, open a command prompt and execute the following commands:

git clone https://github.com/Microsoft/dotnet-framework-docker-samples.git
cd dotnet-framework-docker-samples
del /Q /S .git
cd aspnetapp
git init
git add . 
git commit -m "First commit"
git remote add origin https://git-codecommit.[YOURREGION].amazonaws.com/v1/repos/DotNetFrameworkSampleApp
git remote -v
git push -u origin master

4. Navigate to the CodeCommit repository and confirm that the files you just pushed are there.
# Step 4: Configure build spec

To build your .NET Framework application with CodeBuild you use a build spec, which is a collection of build commands and related settings, in YAML format, that AWS CodeBuild can use to run a build. You can include a build spec as part of the source code or you can define a build spec when you create a build project. In this example, I include a build spec as part of the source code.

1. In the root directory of your source directory, create a YAML file named buildspec.yml.
2. Copy the buildspec.yml contents to buildspec.yml
3. Save the changes to the buildspec.yml and use the following commands to add the file to the CodeCommit repository:

# Step 5: Configure CodeBuild

At this point, we have a Docker image with Visual Studio Build Tools installed and stored in the Amazon ECR registry. We also have a sample ASP.NET Framework application in a CodeCommit repository. Now we are going to set up CodeBuild to build the ASP.NET Framework application.

1. In the Amazon ECR console, choose the repository that was pushed earlier with the docker push command. On the Permissions tab, choose Add.
  1. For Sid, field give it a unique name.
  2. For Effect, choose Allow.
  3. For Principal, type codebuild.amazonaws.com.
  4. For Action, choose Pull only actions.
  5. Choose Save All.
2. Go to the CodeBuild console, and choose Create Project.
3. For Project name, type DotNetFrameworkSampleApp.
4. For Source Provider, choose AWS CodeCommit and then choose the called DotNetFrameworkSampleApp repository.
5. For Environment Image, choose Specify a Docker image.
6. For Environment type, choose Windows.
7. For Custom image type, choose Amazon ECR.
8. For Amazon ECR repository, choose the Docker image with the Visual Studio Build Tools installed, buildtools2017. Your configuration should look like the image below:
9. Choose Continue and then Save and Build to create your CodeBuild project and start your first build. You can monitor the status of the build in the console. You can also configure notifications that will notify subscribers whenever builds succeed, fail, go from one phase to another, or any combination of these events.
