# Movie Picture Pipeline

## Project Overview

Movie Picture Pipeline is a movie catalog application demonstrating automated
Continuous Integration and Continuous Deployment using GitHub Actions,
Docker, Amazon ECR, Amazon EKS, and Kubernetes.

The project contains two applications:

- Frontend: React application
- Backend: Python Flask API

## Public GitHub Repository

https://github.com/ramak7262/cd12354-Movie-Picture-Pipeline

## Architecture

```text
                    GitHub Repository
                           |
                    GitHub Actions
                           |
             +-------------+-------------+
             |                           |
        Frontend CI/CD              Backend CI/CD
             |                           |
        Docker Image               Docker Image
             |                           |
             +---------- Amazon ECR -----+
                           |
                       Amazon EKS
                           |
              +------------+------------+
              |                         |
        Frontend Service          Backend Service
              |                         |
          React UI                 Flask API
