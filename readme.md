JFrog Artifactory is a universal repository manager that provides a central location for managing and distributing software packages and artifacts. It supports various package types, including Maven, Gradle, Docker, npm, NuGet, and more.

Artifactory is often used in DevOps and CI/CD pipelines to streamline the process of storing, versioning, and retrieving build artifacts. It can integrate with other tools like Jenkins, TeamCity, and Azure DevOps, allowing for seamless artifact management in the build and deployment workflows.

Some key features of Artifactory include:

Repository management: Organize and manage repositories for different package types, which can be local, remote, or virtual.
Metadata: Store additional information about artifacts, such as build numbers or custom properties.
Security: Fine-grained access control with user roles and permissions.
Integration: Integrates with CI/CD tools, version control systems, and cloud platforms.
High availability: Supports clustering for scaling and ensuring uptime in enterprise environments.

Artifactory will be deployed on an EKS cluster, ingress-controller will be setup to achieve connection to the application over the internet. Also, secure connection will be achieved with cert-manager install and configurations.

REQUIREMENTS
1. An AWS console
2. Helm 