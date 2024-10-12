In this repo, I list some techs I have used in my two recents projects and share some technical details in regards to how I haved used them.


# CI/CD Pipeline in My Project

![CI/CD Pipeline](./assets/cicd.png)


In my project, I use Gogs, Drone, and Harbor to build a CI/CD pipeline. Gogs is used for code repo. With Harbor, I build a 
image registry. Drone automates the image build, push, and deployment process. It has enabled me to realize automation using just 
a YAML file.

Technical docs: [Gogs](gogs.md), [Harbor](harbor.md), [Drone](drone.md)



