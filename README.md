# DevOps Deployment Guides

A comprehensive collection of DevOps deployment guides and CI/CD pipelines for modern web applications. This repository provides step-by-step instructions and automation scripts for deploying applications to various cloud platforms.

## 🚀 Current Deployments

### Node.js Backend on AWS EC2 with CI/CD

Complete guide for deploying a Node.js backend application to AWS EC2 instances using automated GitHub Actions CI/CD workflows.

**Features:**

- Automated deployment with GitHub Actions
- Zero-downtime deployments using PM2
- Nginx reverse proxy configuration
- Automatic rollback on deployment failures
- Environment management and secrets handling

📖 **[Read the full deployment guide](./nodejs-backend/deploy.md)**

### Kubernetes Crash Course (Local Setup with Minikube)

Complete guide for running containerized apps locally using Kubernetes. Covers core K8s concepts, local cluster setup with Minikube and kubectl, and production-ready manifests for Deployment, Service, and Horizontal Pod Autoscaler (HPA).

**Features:**

- Local Kubernetes cluster with Minikube (macOS, Windows, Linux)
- Deployment with rolling updates, health checks (liveness + readiness probes)
- NodePort Service for local access
- HPA — auto-scales Pods based on CPU/memory usage
- Full kubectl + Minikube commands cheatsheet with real-life explanations

📖 **[Read the full Kubernetes guide](./kubernetes/README.md)**

---

## 🔮 Future Deployments

We're continuously expanding our deployment guides. Upcoming additions include:

- **Next.js Deployment**: Full-stack Next.js applications with optimized build and deployment strategies
- **AWS ECS Deployment**: Containerized deployments using Amazon ECS for scalable Node.js applications
- **AWS Fargate Deployment**: Serverless container deployments with AWS Fargate for cost-effective scaling

## 🛠️ Getting Started

1. Choose your deployment target from the sections above
2. Follow the step-by-step guides in each subdirectory
3. Customize the configurations for your specific application needs
4. Set up your CI/CD pipelines for automated deployments

## 📋 Prerequisites

- AWS Account with EC2 access
- GitHub repository for CI/CD integration
- Basic knowledge of Node.js, Docker, and cloud platforms

## 🤝 Contributing

We welcome contributions! If you have deployment guides or improvements:

1. Fork the repository
2. Create a feature branch
3. Add your deployment guide or improvements
4. Submit a pull request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🔗 Useful Links

- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [PM2 Process Manager](https://pm2.keymetrics.io/)
- [Nginx Documentation](https://nginx.org/en/docs/)

---

**Keywords:** DevOps, AWS EC2 deployment, CI/CD pipelines, GitHub Actions, Node.js deployment, AWS ECS, Fargate, Next.js deployment, cloud deployment, infrastructure as code
