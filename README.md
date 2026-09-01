# Movie Picture Pipeline

## Project Overview

Movie Picture Pipeline is a full-stack movie catalog application demonstrating
Continuous Integration and Continuous Deployment (CI/CD) using GitHub Actions,
Docker, Amazon ECR, Amazon EKS, and Kubernetes.

The project contains two applications:

- **Frontend:** React application
- **Backend:** Python Flask REST API

The application retrieves movie information from the Flask backend and displays
the movie list through the React frontend.

---

## Public GitHub Repository

**Repository:**

https://github.com/ramak7262/cd12354-Movie-Picture-Pipeline

The repository is public and contains the complete project source code,
GitHub Actions workflows, Kubernetes configuration, deployment files, and
project verification screenshots.

---

## Architecture

```text
                         GitHub Repository
                                |
                                v
                         GitHub Actions
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
              Frontend CI/CD          Backend CI/CD
                    |                       |
                    v                       v
              Docker Image             Docker Image
                    |                       |
                    +-----------+-----------+
                                |
                                v
                         Amazon ECR
                                |
                                v
                          Amazon EKS
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
             Frontend Service        Backend Service
                    |                       |
                    v                       v
               React UI              Flask REST API
